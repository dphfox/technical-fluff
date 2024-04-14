---
layout: post
title: Evolving Fusion's execution model
image:
  path: /assets/posts/evolving-fusions-execution-model/thumb.jpg
  width: 256
  height: 256
---

Fusion's moving to a hybrid push-pull system. What will that look like?

## How Fusion works today

A while back I derived the [SETS algorithm](https://elttob.uk/go/sets/) which Fusion uses to push updates through all of the objects in the reactive graph. As luck would have it, I simultaneously discovered this technique alongside a few other reactive frameworks for the web, so that's fun.

However, what's not fun is that this style of updating is very prone to issues. In particular, it's incredibly hard to guarantee that the system is *glitch-free*. While Fusion implements a lot of heuristics to prevent this, your code could still accidentally access dependencies before they're ready, causing you to read stale data, *and there's really not a lot we can do about this.*

Not without a radical rethinking of the idea, at least.

## How Fusion will work tomorrow

What if accessing a dependency caused it to become ready? If you happened to access something out of order, why don't we just shimmy around the order of things and force that dependency to quickly prepare itself at the last minute? That'd *provably* solve every problem Fusion has with dependencies not being ready.

But wait - if that *alone* guarantees that dependencies will be ready, do we even *need* all that other update-pushing stuff? Well, yeah, but only for the simple reason that, if you don't push *some* kind of update down the graph, you'll never know when to run your observers and whatnot. However, you don't need to push *computation-spawning* updates down the graph anymore, because those will happen just in time whenever you ask for the result of the computation.

That's the premise of *pull-based reactive systems*. I've been experimenting with the idea for a while as part of my `tokamak` research library. 

The first part of the idea is *invalidation*; when you make a change to a dependent, push an 'invalidation' update down the graph. That invalidation update essentially paints all the state objects it touches with a message: "I need to be re-run!". If you happen to paint something like an observer, then you can fire those off once you've finished painting everything.

The second part of the idea is *revalidation* (checks out!); when you ask for a value from something that's been "painted" with that invalidation, you go run the calculation, then "unpaint" yourself and cache the result for next time. Now, you can return that result in the future, and all is well.

Invalidation cascades *down* the graph. Revalidation cascades *up* the graph. Easy.

[You can trivially update this to include equality checks, as described here](https://github.com/modderme123/reactively/blob/main/Reactive-algorithms.md) - but we'll talk more about prior art in a sec.

## The benefits of push-pull

So, what does this buy us?

 - **No calculation goes unused** - because you only calculate what you ask for.
 - **Batch changes for free** - because changing multiple things in quick succession only does invalidation. You only ever revalidate once, when you ask for the new value for the first time.
 - **Topological sort as an inherent property** - instead of having to implement a dedicated sorting algorithm with confusing heuristics and niche failure cases, the definition of how this system works guarantees that dependencies run before dependents.

## Is this novel?

I came up with the idea independently. However, the creator of [SolidJS](https://www.solidjs.com/) seems to also have had the same idea, and they *technically* beat me to the punch by releasing [Reactively](https://github.com/modderme123/reactively).

Not that this is a competition! It's cool to see that the entire industry is converging on the same ideas. Fusion's design, in retrospect, is damn near prophetic of what was about to happen to the entire world of web development. I'm happy enough with that achievement on it's own.

## Some other changes

As part of this whole rewrite, I'm planning to more explicitly separate out graph objects from state objects. I'll likely keep the ability for one object to hold both traits - no need to make computed objects more complex - but I'm thinking of extending the current `For` implementation to use dedicated graph objects at some point in the future, so it's a good thing to think about.

I'm going to start work on this imminently, and I'm excited to see what the future holds for Fusion - and for the industry at large. [The future is indeed declarative, young Daniel ;)](https://www.youtube.com/watch?v=WWecENWZpy4&t=580s)