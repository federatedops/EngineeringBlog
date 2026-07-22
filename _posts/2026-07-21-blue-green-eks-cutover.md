---
title: "Blue/Green Your Cluster, Not Just Your Deploys"
description: >-
  How we moved production off an aging EKS 1.29 cluster onto a fresh 1.36 one
  with a DNS cutover instead of an in-place upgrade — and the dev-cluster outage
  that taught us why.
author: federated
date: 2026-07-21 18:00:00 -0400
categories: [Engineering]
tags: [aws, eks, kubernetes, terraform, infrastructure, migration]
---

Last week, production traffic for GroupVAN and ServicePro started flowing through a
brand-new EKS cluster. The old one is still running, still healthy, and completely
idle — which is exactly the point. This is the story of why we stopped upgrading
Kubernetes clusters in place and started replacing them, blue/green style, with DNS
as the traffic switch.

## Where we started

Our production cluster was born on an old EKS version and had been walked up to
**1.29** over the years. It carried the accumulated history you'd expect:

- **AL2 node groups**, with AL2023 looming as the required successor.
- **Unmanaged add-ons** — the EBS CSI driver, CNI, and friends were self-managed
  installs from before EKS managed add-ons were a real option, applied as raw
  manifests and upgraded by hand when someone remembered.
- **`aws-auth` ConfigMap auth**, the legacy IAM-to-RBAC mapping, long since
  superseded by EKS access entries.

None of that is broken on any given Tuesday. All of it is debt that comes due the
moment you touch the version number.

## The outage that made the decision for us

In November 2025 we in-place upgraded our dev cluster past 1.29, and it melted
down.

The postmortem was almost boring in how predictable it looks in hindsight. Two big
changes rode the same upgrade window:

1. **A single-shot jump of the EBS CSI add-on** — adopting the managed add-on over
   the self-managed driver with `resolve_conflicts` unset, so EKS refused to take
   ownership cleanly and the add-on wedged instead of installing.
2. **AL2 → AL2023 node migration at the same time**, so nodes were rolling while
   the storage layer was in a broken half-adopted state.

With the CSI driver wedged, volumes couldn't attach; with nodes rolling, pods that
needed those volumes were being rescheduled onto fresh nodes that couldn't mount
them. Each problem made the other worse, and there was no way to "go back" — an
in-place upgrade has no previous cluster to fail back to. EKS control planes don't
downgrade. We fixed it forward, eventually, on a cluster nobody's customers
depended on.

That was the cheap version of the lesson. The expensive version would have been
running the same playbook on production, where the same self-managed CSI driver,
the same AL2 nodes, and seven versions of upgrade debt were all waiting.

## Why blue/green instead

An in-place upgrade couples every risky change to the same irreversible timeline:
control-plane bump, add-on adoption, node OS migration, auth migration — all on the
cluster that's serving traffic, all without a rollback path.

A cluster cutover decouples all of it. We stood up **`federated-production-v2`**
next to the old cluster and made every scary change *before* a single request
depended on it:

- **EKS 1.36 from day one** — no seven-version ladder to climb.
- **AL2023 node groups from day one** — no OS migration under load; more on what
  AL2023 actually changes in the detour below. (Fun wrinkle:
  the Windows node group's original AMI had been deregistered by AWS by the time we
  needed it again; the fix was recreating the node group on a current AMI, which is
  a non-event on a cluster with no traffic.)
- **Managed add-ons, pinned and guarded.** Add-on versions and conflict resolution
  are now explicit inputs on our shared Terraform cluster module: OVERWRITE on
  create to heal a wedged install, PRESERVE on update across control-plane bumps.
  The exact failure mode that took down dev is now structurally impossible to
  express in our config.
- **EKS access entries instead of `aws-auth`**, with deploy roles scoped to
  namespace-level admin, and the namespaces themselves owned by the infrastructure
  pipeline — so app deploys can't (and don't need to) hold cluster-admin.
- **A private-only API endpoint**, which the old cluster never had, with deploys
  running on in-VPC runners.

And critically: the old cluster keeps serving the whole time. Rollback isn't a
runbook, it's *the current state of the world* until we flip DNS.

## A detour: what AL2 and AL2023 actually are

We've been throwing the acronyms around, so let's unpack them, because "new node
OS" undersells how much actually changes.

**AL2 is Amazon Linux 2**, AWS's general-purpose Linux distribution from 2018,
with lineage in the RHEL/CentOS 7 era: `yum` for packages, cgroup v1, an old
systemd, and classic `network-scripts` networking. It's what EKS node AMIs were
built on for years — and its end of support landed in **June 2026**, which turned
"we should migrate at some point" into a deadline.

**AL2023 is Amazon Linux 2023**, its Fedora-based successor, and it modernizes
the whole stack at once:

- **cgroup v2** — better resource accounting for containers, but older runtimes
  and JVMs that only understand v1 memory limits need updating first.
- **`dnf` with versioned, point-in-time repositories** — package installs are
  deterministic by default. The same node group launched twice gets the same
  packages, instead of whatever the mirror had that morning.
- **IMDSv2 enforced, with a default hop limit of 1** — pods can no longer reach
  the node's instance-role credentials through the metadata service. That's a
  security win, but it silently breaks any workload that was (perhaps
  unknowingly) leaning on the node role. We hit exactly this: a workload that
  needed S3 access got it via the node role on the old cluster and got
  `AccessDenied` on v2. The fix is the supported path — IRSA (IAM Roles for
  Service Accounts), where the pod's own service account is annotated with a
  role — which is *better*, but it's a migration item you have to find, and
  you'd much rather find it on a cluster serving no traffic.
- **`nodeadm` instead of `bootstrap.sh`** — node bootstrap is a structured
  `NodeConfig` YAML in user data rather than a shell script with flags, so any
  custom launch templates need rewriting, not tweaking.

### DNS management changed underneath you too

The sneakiest difference is name resolution. On AL2, networking is the classic
setup: `dhclient` gets a DHCP lease and writes the VPC resolver's address
straight into `/etc/resolv.conf`. Every process on the box — kubelet included —
reads that file and talks directly to the VPC DNS server.

AL2023 replaces that stack with **`systemd-networkd`** for network configuration
and **`systemd-resolved`** for DNS. `resolved` runs a local caching stub
resolver on `127.0.0.53`, and `/etc/resolv.conf` becomes a symlink pointing at
it; the *real* upstream servers live in a separate file
(`/run/systemd/resolve/resolv.conf`) that `resolved` maintains.

On a laptop, nobody notices. On a Kubernetes node, it matters: CoreDNS builds
its upstream forwarding from the `resolv.conf` kubelet hands it. If kubelet
naively passes the stub file along, pods inherit a nameserver of `127.0.0.53` —
a loopback address that doesn't exist inside their network namespace, or worse,
a forwarding loop of CoreDNS pointing at itself. The EKS-optimized AL2023 AMI
handles this for you — kubelet is pointed at the real upstream file, so CoreDNS
forwards to the VPC resolver like it always did — but if you build custom AMIs
or override kubelet's `resolvConf`, this is the difference that pages you at
2 a.m. with "DNS is flaky in some pods."

The theme across all of these: none are exotic, every one is findable in an
afternoon of testing — and every one is a thing you want to discover on a
cluster with zero traffic, not mid-upgrade on the one serving production.

## What made cutover cheap for us

Blue/green clusters have a reputation for being expensive because of state. Two
things made it nearly free here:

**Production was almost stateless.** The one real exception was GroupVAN's
self-hosted MongoDB, which stores request logs. Instead of a dump/restore
choreography, we let the v2 Mongo start **blank**: DNS splits traffic during the
cutover, so every request is served — and logged — by exactly one cluster. The two
clusters' logs are disjoint by construction, and the two ingest pipelines append
into the *same* analytics table with no dedup step at all.

**The network outlives the cluster.** Our NAT gateways, EIPs, and subnets live
outside the cluster module and are shared by both clusters. Partners who allowlist
our egress IP never noticed anything happened.

## The cutover itself: DNS as the traffic switch

Every production hostname is a Route53 alias record in Terraform pointing at an
ALB minted by the cluster's ingresses. That made the actual cutover a series of
small, reviewable, individually-revertable pull requests:

1. **Enable the AWS Load Balancer Controller on v2** so the ingresses mint real
   ALBs. This had its own mini-lesson: the controller's IAM policy in our module
   was a years-old snapshot of the upstream recommended policy, and the new
   controller version calls ELB APIs that didn't exist back then. Every reconcile
   403'd on `elasticloadbalancing:DescribeListenerAttributes` until we re-synced
   the policy file from upstream — verified additive, action by action, before
   apply.
2. **Canary two hostnames** — one ServicePro host, one GroupVAN API host — after
   smoke-testing every host against the new ALBs with pinned DNS. Alias records
   flip atomically, and the old ALB stays up, so "rollback" is reverting a
   two-line PR.
3. **Flip the rest** once the canaries ran clean, and while we were in there,
   **adopt the strays**: two hostnames turned out to exist only as hand-made
   Route53 records that predated our Terraform coverage. They're now regular
   Terraform resources like everything else — the cutover doubled as a cleanup of
   DNS state nobody owned.

Each step was a Terraform PR with a reviewable plan. Nothing was clicked in a
console; nothing depends on someone remembering what they typed.

## What we'd tell you

- **In-place upgrades concentrate risk; cutovers spread it.** Everything scary
  about our migration happened on a cluster serving zero requests.
- **Let a dev cluster teach you cheaply.** Our November outage was the best
  infrastructure investment we made all year — it just didn't feel like it that
  day.
- **Never bundle add-on adoption with an upgrade.** Managed add-on adoption over
  self-managed installs is its own risky change. Pin versions, set conflict
  resolution explicitly, and do it in its own window.
- **Keep long-lived network primitives outside the cluster.** Clusters should be
  replaceable; your egress IPs and subnets shouldn't be.
- **DNS is a great traffic switch if — and only if — everything is in code.** The
  flip is atomic, the rollback is a revert, and the plan diff is the change
  review. The strays we found along the way are exactly why "mostly in Terraform"
  isn't the same thing as "in Terraform."

The old cluster gets decommissioned once the last few internal consumers move
over. Until then it costs us a little money to keep the best rollback story we've
ever had: a whole production environment, one DNS revert away.
