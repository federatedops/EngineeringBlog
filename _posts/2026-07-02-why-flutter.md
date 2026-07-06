---
title: Why Flutter?
description: >-
  Why we're building Catalog on Flutter and Dart instead of the JavaScript/React
  default — the case for real types, compiled binaries, and an app that holds up at
  the parts counter.
author: federated
date: 2026-07-02 10:00:00 -0400
categories: [Engineering]
tags: [flutter, dart, webassembly, frontend, architecture]
---

When you set out to build a web app in 2026, the path of least resistance is
well-worn: JavaScript, React, today's build tooling, and a few dozen packages to glue
it together. It's a good path. Millions of excellent apps are built
on it every year.

We didn't take it. For **Catalog** — the parts-shopping app we're building on top of
[GroupVAN](/posts/what-is-groupvan/) — we chose **Flutter** and the **Dart** language.
This is the case for that choice, made in good faith, with full credit to the tools we
passed over.

> Our bias, stated plainly: for software that authorizes a purchase, we want the
> compiler to catch our mistakes before a customer does.
{: .prompt-tip }

## The trouble with "just ship JavaScript"

JavaScript is the most successful programming language in history, and it earned the
spot. But it is dynamically typed, and in a system that moves real orders for real
money, dynamic typing is a liability wearing the costume of flexibility. JavaScript
will happily let you add an array to an object, decide that `NaN` isn't equal to
itself, and ship the result — and it won't mention any of it until a customer finds it
for you at the counter.

"So use TypeScript," you say — and you'd be half right. TypeScript is a genuinely
clever negotiation with JavaScript. But it's worth being precise about what it gives
you: TypeScript's types are **erased at compile time**. They're there to help you and
your editor; by design they don't affect the program that actually runs, and
*soundness is an explicit non-goal* of the language. For the web at large, that's a
sensible trade. For software that authorizes a purchase, it's a strange one — a
guarantee that evaporates right before the moment it would matter.

We wanted types that survive to runtime: a compiler on our side, not a linter we can
silence with `any`.

## What a compiler should do for you

[Dart](https://dart.dev/) is statically typed and **sound** — it pairs compile-time
checking with runtime checks so a value can't quietly turn into something its type
says it isn't. Sound null safety is mandatory, which means the classic
"undefined is not a function" failure mode simply isn't in the vocabulary. None of
this is exotic. It's just the compiler doing work we'd otherwise be doing by hand, in
code review, at 5 p.m. on a Friday.

And Dart doesn't stay in the browser. Flutter takes one codebase and compiles it:

- **Ahead-of-time to native machine code** (ARM, x64) for iOS, Android, Windows,
  macOS, and Linux.
- **To the web**, as either JavaScript or **WebAssembly**.

One language, one codebase, every screen a distributor might touch — the counter
desktop, the manager's phone, the tablet in the warehouse — behaving identically,
because Flutter draws its own interface instead of inheriting whatever the local
browser and OS feel like doing that day. If you've ever watched a layout that was
pixel-perfect in Chrome quietly fall apart in an older Safari, you know precisely which
tax we're declining to pay.

React, for its part, is a fantastic way to build for the web, and it deserves its
popularity. It also asks you to assemble a framework out of a few dozen independent
packages, to keep pace with an ecosystem whose conventions move quickly, and to trust
a dependency graph you didn't audit. A single npm install routinely pulls in dozens of
transitive packages maintained by dozens of strangers; the industry once watched an
eleven-line package get unpublished and break thousands of builds around the world.
That's not a jab at anyone in particular — it's the physics of a huge, decentralized
ecosystem. We'd simply rather own our toolchain end to end.

## Betting on typed, compiled WebAssembly

Here's the part we're most excited about. Flutter can compile Dart to
**WebAssembly** — a real, compiled binary that runs in the browser at near-native
speed. Not source the browser has to parse and compile every time it loads. A binary.

We'll be honest about the state of the art, because overselling it would undercut the
point. Flutter's WebAssembly output became stable in 2024, and the browser feature it
depends on is now supported across all three major engines — but only since Safari
joined at the end of 2024, so older browsers still fall back to JavaScript
automatically. It's an emerging foundation, not yet a universal one. That's exactly why
we call it a *bet*: we believe the future of serious web applications is typed source
compiled to a fast binary, and we'd rather build toward it now than retrofit it later.
The target is a Catalog whose type safety runs unbroken — from the Dart model behind
the screen, through the typed GroupVAN client, all the way to the wire — with no soft
spot where a value's shape is left to guesswork.

## Built for the edge

Most "which framework" arguments are settled on fast laptops over fiber. Our users
don't work there. They work at a parts counter: a busy, loud place, on whatever device
is bolted in next to the register, sometimes on wifi that comes and goes. That counter
is the **edge** — where the transaction actually happens, and where a stutter or a
dropped order isn't a bug ticket, it's a lost sale and an annoyed customer.

This is where a compiled, self-contained app earns its keep. Flutter aims for a steady
60 frames per second (more on displays that allow it), precompiles its graphics to
avoid the first-run stutter that punishes lower-end hardware, and renders the same on a
five-year-old counter PC as on a brand-new phone. Installed, these apps run offline; on
the web, they can too. What a counter rep sees at 7:45 on a Monday is the behavior we
tested — not a dice roll against the device in front of them.

And if it helps to know the idea isn't ours alone: Flutter is trusted to run the
infotainment system in 2026 Toyotas. A parts counter is a friendlier place than a
moving car — we just intend to make it feel as solid.

## The honest tradeoffs

None of this is free, and pretending otherwise would be its own kind of dishonesty. A
Flutter web app ships a multi-megabyte engine before your code even runs, so the first
load is heavier than a lean HTML page. Because it paints to a canvas instead of a
document, search-engine indexing suffers, and accessibility — screen-reader support,
keyboard focus, text scaling — is something we have to build in deliberately rather
than inherit from the DOM. For a tool staff use all day, we treat that as an
obligation, not an afterthought. If we were building a content site or a marketing page,
React and the DOM would be the obvious, correct answer — and we'd reach for them without
a second thought.

But Catalog isn't a page to be read. It's a tool to be used — all day, at the edge,
where correctness and consistency beat everything else. For that job we want a real
language with a real compiler, one codebase for every screen, and a path toward typed
binaries running close to the metal.

That's why Flutter.
