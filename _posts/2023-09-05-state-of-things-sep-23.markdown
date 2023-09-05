---
layout: post
title:  "State of things (Sep 2023)"
image:
  path: "/assets/posts/state-of-things-sep-23/thumb.jpg"
  width: 256
  height: 256
---

I'm unmuddling my brain by laying out the status of everything I'm working on,
just before I pop out to San Francisco for a bit.

## Fusion

This is the main focus of my life at the moment, and one of my more popular
projects, so I'll be generous with word count here.

Work on 0.3 is well underway. I'm currently focusing on how best to move away
from garbage collection to manual, explicit destruction that helps statically
enforce more guarantees about code, and helps manage the heaps of objects that
Fusion code creates without passing on that burden to developers.

In the background, I've got other things on my mind - in order of priority:

- unifying ForKeys/Values/Pairs into one object again, with a new design that
encodes reuse behaviour in the reactive graph, hopefully simplifying the
implementation and allowing the other three objects to act as syntax sugar

- separating out the reactive graph from state objects to allow better modelling
of currently-hellish objects (like the aforementioned For objects)

- changing to lazy evaluation to reduce work done in subgraphs that aren't ever
read before being updated again - esp. pertinent for later animation work

- replacing Hydrate with 'cloning New' and dedicated state/graph objects for
hooking into existing instances more ergonomically

- animation updates that introduce better timing mechanics and are more generic
over different kinds of animation curve (e.g. for transitions)

- moving towards Fusion being platform-agnostic ("Luau-first, Roblox second")
so it can be used without Roblox present (great for unit testing)

- establishing a proper style guide again and splitting the project into better-
scoped modules to better fit developers with different needs or working in
different environments

I'll elaborate on the last point a bit. I could imagine Fusion, instead of being
a monolithic project, instead being a bundle of related modules. Pehaps at the
extreme end of modularisation (and including a few hypothetical additions), it'd
look like this as it grows:

- `fusion-core` - reactive graph implementation, state object API definition
- `fusion-compute` - state objects for basic compute (Value/Computed/Observer)
- `fusion-iter` - state objects for iteration (ForKeys/Values/Pairs)
- `fusion-time` - state objects related to timing and time measurement
- `fusion-motion` - animation & motion design state objects
- `fusion-component` - component convention interfaces, types, shared APIs
- `fusion-layout` - state objects for 2D/3D layout
- `fusion-roblox` - Roblox-specific integrations
- `fusion-love` - Love2D-specific integrations
- `fusion-godot` - Godot-specific integrations

I won't comment on how many of those hypothetical modules are unannounced
research projects of mine. :)

## Vanilla

Work on 3.1 is mostly paused at the moment. I'm still not entirely sure what the
best colour scheme for icons is right now. I'd like to get around to releasing
the commercial version eventually, but I'm sorting through legal/licensing
stuff and need to figure out how I should be massaging the raw project files
into something more distributable and useful.

## Elttob Suite

A while back I made the call to tie the Elttob Suite into my Fusion and Vanilla
work. It's currently waiting on some of the Fusion tech to mature before I start
work on updated versions of my plugins. They'll be more polished and interactive
than ever since I hope to no longer waste so much time on logistical work.

I have some really cool prototypes for new plugins that I would love to share in
a fully fledged product soon. I have ideas for many more. I look forward to the
day that this new technology stack is mature enough that I can easily work on
all of them with unprecedented new velocity, but right now it's still a touch
too underbaked for the level of tomfoolery I'm going to be doing. I've got some
visionary stuff in the works here, at risk of hyping it up too much.

I'm thinking of making dev tools for Fusion part of the Elttob Suite. They'd
share the same tech as the other Suite plugins, and be specifically focused on
adding DX quality-of-life features for working with Fusion code. I've already
hinted at a few tools I've been working on in the past, but nothing to report
here on those efforts.

In parallel, I've been working on other stuff for the Suite:

- building a comprehensive design system - I call this effort 'SuiteUI'. It
builds from first principles much like the original Material Design did, to
build a paradigm that is generic over viewport types and input methods. It's
less about pixels on a screen and more about building physical things. This
perhaps hints at some future intentions I have for where I want to take this
design system after I'm done building plugin widgets with it.

- working on a bunch of solid infrastructure for plugins to handle multi-
instance better than they do today, specifically looking to protect against data
races in Roblox's APIs and provide more ergonomic interfaces for inter-process
communication.

- support for plugin integrations - I would love a future where Suite plugins
can co-operate together and share functionality. Imagine if you could access
Reclass APIs inside InCommand scripts.

We'll see how those things go.