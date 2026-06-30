---
title: GroupVAN (for geeks!)
description: >-
  GroupVAN is the B2B gateway that connects APSG members to the next-generation
  technologies serving their business, moving hundreds of millions of dollars in
  parts orders a year. Here's what it is, how it's built, and the data it powers.
author: federated
date: 2026-06-29 14:00:00 -0400
categories: [Engineering, Platform]
tags: [api-gateway, architecture, async-python, b2b, ecommerce]
---

**GroupVAN** — short for **Group Value Added Network** — is the gateway that connects
[APSG](https://www.thegroupapsg.com/) members to the next-generation technologies
serving their business, moving **hundreds of millions of dollars in parts orders
every year**.

It's two things at once: an **inventory gateway** that answers "do you have this part,
where, and at what price?" in real time, and an **order gateway** that turns the
answer into a purchase. Wrapped around both is a translation layer that lets systems
which don't speak the same language transact with each other anyway.

## The problem GroupVAN solves

The automotive aftermarket runs on a patchwork of software. A counter app, a
shop-management system, an e-commerce storefront, a procurement platform — each
speaks its own dialect. Some speak modern JSON. Some speak XML conventions that
predate JSON. API versions drift over the years, and not every system upgrades on
the same schedule.

GroupVAN sits in the middle and makes them interoperate. A buyer's app can send a
request in the newest format and have it delivered to a seller's system in the older
format that system still expects — then translated back on the way home. Nobody has
to rewrite their integration just because someone else on the network changed theirs.

## Rebuilt from first principles

A few years ago we rebooted GroupVAN from the ground up, optimizing for one thing:
**simplicity**. The original system had accumulated branching logic for every edge
case until it was hard to read and harder to change. The rebuild traded that for a
handful of clear ideas.

**One model, many formats.** Internally, every inquiry and every order is a single,
strongly-typed object. We never special-case "the v2.1 version" of an order against
"the v3.2 version" deep in the code. Instead, a **serializer/deserializer** (a
"SerDe") translates that one canonical object to and from each wire format —
Federated JSON from v2.1 through v3.2, partner XML dialects, and the SOAP/XML APIs
of third-party systems. Adding support for a new format means writing one more
translation, not threading a new branch through the whole codebase.

**Async, because the work is mostly waiting.** A gateway spends almost all of its
time idle — waiting on a member's pricing engine or inventory service to respond.
That's an *IO-bound* workload, and it's exactly the wrong fit for billing models
that charge for compute while the CPU sits idle. So GroupVAN is built async-first on
the [Quart](https://quart.palletsprojects.com/) microframework: a single modest
container juggles thousands of in-flight requests at once, and we scale out by adding
containers, not by paying per request to wait. The rebuilt gateway runs **10–50×
faster** than the platform it replaced, at a lower fixed cost.

**Authorized, and cached close to the metal.** Every request is checked against
fine-grained permissions — which member, which integration, which service, which
user — before anything is relayed. Those permission records are cached in Redis, which
takes a lookup that used to cost ~75 ms down to single-digit milliseconds. Auth is
handled with signed JWTs, so integrators hold a token rather than a password.

**Versioned, namespaced endpoints.** Rather than funneling every command through one
catch-all URL, the API is organized into clean, versioned resources. Integrators get
a stable contract they can build against and a clear upgrade path when a new version
ships.

## Lightweight, flexible, and built to be trusted

GroupVAN is deliberately a thin layer that gets out of the way of the real work
happening in members' systems. But "thin" doesn't mean "casual":

- **Validated end to end.** Requests and responses are checked against typed models
  at runtime, so malformed traffic fails fast with a clear message instead of
  silently corrupting an order.
- **Observable by default.** Live metrics, health checks, and a correlation ID
  stamped on every request mean we can see exactly how each transaction flowed and
  where time went.
- **Made for integrators.** A published, versioned API spec; generated reference
  documentation; and client SDKs in several languages (Python, Node.js, PHP, C#, and
  Dart) so partners can integrate in whatever stack they already use.

> The design goal is to be boringly reliable: fast, predictable, and easy to build
> against, so a distributor's first integration is also their last headache.
{: .prompt-tip }

## Two gateways and a content layer

Everything above serves a few core jobs:

- **Inventory gateway** — a real-time *item inquiry* returns availability,
  account-specific pricing, and alternate locations, relayed to and from the right
  member's system.
- **Order gateway** — *place order* turns an inquiry into a real purchase, alongside
  related operations like cancellations, invoices, statements, and account lookups.
- **Catalog content services** — a family of APIs for the parts catalog itself:
  product and part search (including the streaming OmniSearch described below),
  vehicle lookups by VIN, license plate, or year/make/model/engine, interchange and
  cross-reference data, and product attributes and images. These are the services
  behind the Catalog app we're building — and because they live on the same
  published, versioned API, **third-party and member integrators can build on the
  exact same content services we do**, in their own apps and storefronts.

## A fast path of our own

GroupVAN is built to speak everyone's language — but it also speaks natively to
**Parts Manager**, our in-house point-of-sale and warehouse-management system.

That pairing is where the architecture pays off. The gateway itself adds only about
**3 milliseconds** of overhead to an inquiry in production: it authorizes, translates,
logs, and gets out of the way. Backed by Parts Manager, a member can answer a live
availability-and-pricing check in **tens of milliseconds** — where the legacy
back-office systems that still dominate the industry often take *seconds*. For a
counter rep with a customer waiting, that gap is the entire experience.

## The data it generates

Because every inquiry and every order flows through one place, GroupVAN produces a
clean, structured record of what the whole network is doing — and that record feeds
our data platform.

We keep the gateway itself dumb on purpose: it relays and logs, and the analysis
happens downstream. At a high level, that transaction stream becomes network-wide
**intelligence** — demand signals and sales patterns, pricing and availability
trends, and a view of where orders are being lost to stockouts. Combined with
vehicle-population and parts-fitment data, it can even forecast what's likely to sell
where, and suggest what each location should stock next. None of that requires the
gateway to do anything but its job well; the value falls out of having one honest
record of every transaction.

## The catalog it will power

The next thing GroupVAN is set to enable is **Catalog**, the modern parts-shopping
and ordering app we're building on top of it. It's a single
[Flutter](https://flutter.dev/) codebase designed to target web, mobile, and desktop,
starting with the web — and it's a genuinely *thin* client: nearly all of the business
logic lives behind the GroupVAN API. Catalog just renders it.

A few things it's designed to do:

- **OmniSearch** — start typing and results stream back over a single multiplexed
  WebSocket: part types, catalog parts, member parts, and matching vehicles all
  populate progressively instead of waiting on one big response.
- **Search the way the trade actually works** — by part number, keyword, VIN,
  license plate, or year/make/model/engine — then layer interchange and
  cross-reference lookups and account-specific pricing on top.
- **Cart to order** — build a cart and check out into a real member order, with
  invoices and statements available right in the app.

The catalog content Catalog renders is the data our platform curates; the live
pricing and order placement are relayed by GroupVAN to each member's system. For
everything that matters — search, catalog data, pricing, and orders — Catalog talks
to GroupVAN and nothing else, which is exactly the point. Curated catalog data, a
gateway that speaks everyone's language, and a fast, modern UI will add up to a
modern e-commerce experience for members who could never have built one alone.

## In short

GroupVAN is the quiet infrastructure under a lot of our network's commerce: a
lightweight, async gateway that translates between everyone's systems, authorizes and
logs every transaction, and gets out of the way. It turns a patchwork of incompatible
software into one network that can shop, price, and order in real time — and the clean
record it leaves behind powers everything from analytics to the next generation of our
catalog.

More on each of these pieces in future posts.
