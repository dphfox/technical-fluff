---
layout: post
title: Please, no more "UI frameworks"
image:
  path: /assets/posts/please-no-more-ui-frameworks/thumb.jpg
  width: 256
  height: 256
---

Roblox UI is an island far removed from the rest of the engine. This is my manifesto: focusing so heavily on user-space client-centric UI-first frameworks has damaged how we talk about the future of Roblox and the Luau ecosystem.

I'll be tackling this post from a technical perspective, but [I've written a few years ago about even more problems with these frameworks from a low-code/designer perspective too.](https://fluff.blog/2022/11/10/ui-frameworks-dont-serve-designers.html)

## UI is not a unique problem

We developed UI frameworks to allow us to define a simple source of truth, and derive a more complex set of consequences from it. Fundamentally, this is all that React, Fusion, and other frameworks try to do. Yet, it's hardly the first time we've been confronted with this problem in the engine.

Consider the example of reusable model packages. Say I want to distribute a car on the Creator Store as a 3D model, built of multiple meshes. I want to give users an artist-friendly "paint colour" attribute so that they can configure the colour of the car's meshes without reaching inside and making invasive modifications to each mesh.

Other benefits include (non-exhaustively):
- Efficient replication - a minimal amount of state can be driven from the server, and clients can infer the rest, reducing bandwidth usage
- Frictionless updates - the attributes can be configured externally without touching any internal instances, so there are no conflicting changes when upgrading to a new package version
- Fluent controls - a low-code user of this package can easily discover all of the axes it can be customised along, and can immediately tweak and see the effect of those controls, improving iteration times and encouraging asset variation

I'd feel pretty confident in saying that packages would benefit from being able to derive at least some state from a simpler set of inputs. The car would obviously still have other internal state, such as the velocity of the different assemblies, but as the user, we don't really need to concern ourselves with that.

If this sounds familiar, it's because I believe that packages with attributes are isomorphic to modern-day UI components - it's almost exactly the same state derivation problem, and is amenable to equivalent solutions. 

UI is not special - it's merely the same problem at scale. Any well-done solution that solves one of these problems should solve the other, but no attention has been given to this at all. 

## Luau is fundamentally compromised

One of the most difficult problems in Fusion's history has been the memory management problem; how can we make sure that resources allocated in state objects get destroyed properly when they're recomputed, without destroying resources that are still in use? In particular, this is relevant for resources like instances which require explicit manual cleanup (e.g. unparenting or destruction).

Since the `__gc` metamethod is absent, the only thing Luau can do is respond to the absence of a resource handle (e.g. by polling a weak table), at which point it's too late to perform logic involving that resource. You could log a generic message, perhaps, but the data stored inside that handle is gone, often making it impossible to take a new reference to whatever was on the other side (with the notable exception of instances, where the data model allows fresh handles to be obtained if you're *really* persistent).

This always-early-or-late problem haunted Fusion for a very long time. No matter how hard I looked, I couldn't find a way to square this circle. It's a fundamental compromise of Luau, and I have left many notes on this blog over the years to warn you that you won't solve it either.

In the end, I worked around it by introducing structured object lifetimes to Fusion, in the form of scopes. Famously, scopes are very easy to teach and nobody asks questions about them.

But all I can think of when I reflect on this problem, is just how entirely unnecessary it is. It's an invented problem caused by a limited API surface. I have good reason to believe that we could offer a simpler and more understandable solution to this kind of problem if only the implementation lived lower down the tech stack, where the resources actually live.

This is just one of the downsides of building complex UI frameworks in user space, but it's especially salient whenever people talk about post-React framework design. I don't think it's possible to combine granular reactivity with managed resources like instances without running headfirst into exactly this problem, and we will only be left with unsatisfying solutions.

## Manual replication is un-Robloxian

We appropriated today's attitude towards building Roblox UI directly from web conventions around single-page app (SPA) architecture. As the web world has learned over time though, there are tangible benefits to leveraging the server side more fully, especially when dealing with state that needs to be protected by server authority, and there is a lot of room to make the client/server divide more transparent and easier to work with.

UI shouldn't require a special kind of script to be properly constructed. I think Roblox had something good going with server-side UI control, because it naturally allows the game state to live on the server, where there is one authoritative world view, making it easy to integrate with other game state and easy to control securely. It avoids the mental overhead of having to manage a client/server divide, and it's my belief that we missed out on a better API design by heavily restricting the server side from being involved in user input and rendering.

The only reason we need client-side UI in 90% of use cases is to improve responsiveness in certain cases; for example low-latency responses to user input or smooth animation sequences. My view is that those are solvable design problems; even without adopting some advanced server authority system, you could *at least* replicate animations more smartly than how `TweenService` does things today, and you'd see meaningful payoff even if more complex actions would still have high latency.

Ultimately, I just think the engine needs to get smart about scheduling work to be done on clients at a fundamental level. Remotes and local scripts were a mistake in my eyes - sure, they're a good power user tool for those who really need them, but we shouldn't have ever shuttled novice developers through these mechanisms by force.

Would this require abandoning some purity around consistent worldviews? Would it mean that clients could get out of sync because they played animations at slightly different times, or processed inputs slightly differently from the server? Sure! But let's not pretend like we ever had that - the speed of light already guarantees that clients have inconsistent worldviews due to networking. The sooner we stop trying to be precious about something we don't have, and just implement proper rollback and server authority mechanisms, the better for the user.

Roblox UI should be following the example of the web and better learn how to leverage Roblox's always-available server authority and cloud compute. We have a great replication and networking model with the potential to make server authority easy to reason about. Let's lean into that rather than building yet another framework for SPAs with a complex manual networking layer.

## It is time for a better way

It's now my full belief that we should be spending less time ogling at Luau frameworks and more time investing into the fundamental design of the Roblox engine. That's not to say that Luau frameworks have no place in a future Roblox, but they should not be a prerequisite tool to achieve reasonable UI goals.

We should take what we've learned about the way we want to build UI, and apply it by introducing key engine features to make the fundamental experience of building UI simpler. We are long overdue proper tools for componentisation and for managing the client/server boundary.

It's also my belief that many of the modern paradigms we want to adopt, such as granular reactivity, would benefit from having lower-level implementations - not merely for some mythical language speedup (runtime complexity matters more), but because they would benefit from proper engine coordination in a way that would meaningfully reduce user-space mental overhead.

I'm thinking about following up this post with some thoughts about a mental model for UI that extends the skeuomorphic idea of bindings into a proper engine-level primitive, in a way that might even be compatible with actor-style parallelism. I'll probably stew on it some more though, because I'm torn on a few design problems at the moment.