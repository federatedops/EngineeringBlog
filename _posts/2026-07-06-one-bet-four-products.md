---
title: One Bet, Four Products
description: >-
  The follow-up to "Why Flutter?" — what happens when you make the same framework
  bet across a whole product suite: shared components, a shared typed client, one
  product embedded inside another, and a small team that can build across all of it.
author: federated
date: 2026-07-06 10:00:00 -0400
categories: [Engineering]
tags: [flutter, dart, architecture, state-management, bloc]
---

In ["Why Flutter?"](/posts/why-flutter/) we made the case for the language: real
types, a real compiler, one codebase, and a path to typed binaries at the edge. That
post was about a single choice. This one is about what happens when you make that
choice *everywhere*.

We build **four products** on Flutter:

- **GroupVAN Console** — the admin surface for the GroupVAN gateway.
- **Catalog Manager** — where our catalog data is managed and published.
- **Catalog** — the parts-shopping and ordering app.
- **ServicePro** — a full shop-management system for service and repair.

Four products, one language, one state-management model, and a growing pile of shared
code. Here's the leverage that creates — the part of the Flutter bet that only pays
off once you've made it more than once.

## Shared code, literally

The obvious win of one framework is a shared look and feel, and we take it: a single
design-system package, **Widgets**, gives every product the same components. A button
in the Console behaves like a button in Catalog because it *is* the same button.

The less obvious win is sharing *logic*. Every product talks to the GroupVAN gateway
through one typed client — the **GroupVAN SDK** — so the contract to our backend is
written once, in Dart, with types that hold from the UI all the way to the wire. When
the gateway gains a capability, the SDK gains a method, and every product can use it
without re-implementing the request.

Then there's the highest-leverage move. A service writer in ServicePro, building a
repair ticket, needs to find and order parts. Instead of bolting on a second parts UI,
we're **embedding Catalog directly into ServicePro** — the whole app, as a Flutter
component, authenticated through the same shared GroupVAN client. The writer will
search, price, and order without ever leaving the ticket — reusing the whole app rather
than rebuilding it. That's the thing you can only do when your products are the same
kind of thing all the way down: you reuse a *whole application*, not a handful of
snippets.

The scan module is on the same trajectory — it's moving into Widgets next, so every
product can drop in registration and VIN scanning the way it drops in a button.

## One opinionated way to hold state

None of this composes if each product invents its own way of managing state. So we
don't. Across all four, state lives in **flutter_bloc**, arranged the same way every
time:

- A **feature-first** layout — each feature is a self-contained folder of *state*,
  *view*, and *data*.
- **Blocs and Cubits** turn events into states in one direction, so data flow is
  predictable and there's exactly one place a screen's state can change.
- A **repository** wraps the data source — the GroupVAN SDK, or Supabase in ServicePro's
  case — so business logic never touches a raw network call.
- **hydrated_bloc** persists the state worth keeping — preferences, recent work — so an
  app comes back the way you left it, even offline.

flutter_bloc is *opinionated*, and that's the point. Yes, it asks for more ceremony
than tossing state into a widget. In exchange, every feature in every product is built
the same way — which makes the structure itself a kind of documentation. A developer
opening a feature they've never seen already knows where the state lives, where the
data comes from, and how to test it (we lean on `bloc_test` to exercise business logic
with no UI at all).

## The small-team dividend

Here's where it cashes out. We're a **small team of cross-functional engineers**, not
four separate product teams — and one stack is what makes that possible.

Because it's the same language, the same state model, the same components, and the same
typed client in every repository, moving from the Console to ServicePro isn't so much a
context switch as a change of scenery. There's no new framework to learn on Tuesday, no
second mental model to keep straight. An engineer can fix a bug in Catalog in the
morning and add a feature to ServicePro in the afternoon — and once Catalog is embedded
inside ServicePro, those two efforts literally share code. One team can credibly own a
whole suite, and each product makes the next one cheaper to build.

## Still built for the edge

ServicePro is where the "edge" argument from the first post gets concrete. It runs at
the service counter and in the bay, so it's built to work there:

- **On-device scanning.** It reads the VIN and the barcode on a vehicle registration
  and decodes them **on the device** — no photo uploaded, no server round-trip — so a
  rep can scan a registration and skip the typing. We're extending that with on-device
  **text recognition** for the registrations that don't carry a standard barcode: the
  model runs locally, on the phone, right there at the counter.
- **A managed backend.** ServicePro uses **Supabase** for authentication, its data, and
  live sync between the front desk and the bay — so the team ships shop features instead
  of standing up infrastructure.

All of it in the same Flutter stack, sharing the same components and the same typed
client as everything else we build.

## The interest on the bet

"Why Flutter?" was the principal. This is the interest.

One design system, one typed client, one product embedded whole inside another, one
state model everywhere, and a small team that can build across all of it without ever
leaving the language. We didn't choose Flutter four times. We chose it once — and it
keeps paying.
