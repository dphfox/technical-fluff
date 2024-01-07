---
layout: post
title: A reactive smart home
image:
  path: /assets/posts/a-reactive-smart-home/thumb.jpg
  width: 256
  height: 256
---

Event-based programming is known to cause obscure complications in systems design - so why do we expose it to users setting up smart homes?

## Understanding the problem

Right now, smart home systems expose a pretty direct interface to users; devices emit triggers, which are sent to your hub. Then, in response to those triggers, events are invoked on other devices. This directly models the flow of messages between devices happening under the hood.

As a result of this structure, every device in the system doesn't just hold its own state internally; it exposes its internal state to users. Imagine if every light bulb in your house had buttons on it to turn on/off or change colour. All a smart home does is let you push those buttons invisibly using another device, such as a wireless remote. It doesn't actually simplify the concept.

For the simplest setups, this is no problem; so long you have a single controller, it's trivial to push all of their 'invisible buttons' at the same time and end up with a sensible result. However, the moment you need anything more complex than that, the scattered-ness of the system becomes a real, tangible problem.

To illustrate the problem, consider this [Reddit post where a user tries to control a light with both a motion sensor and a light switch](https://www.reddit.com/r/homeassistant/comments/tdyiv0/reactive_automations/). 

They want the following logical statement to always hold true (this isn't real code):
```
brightness OF light 
	IS 100% WHILE 
		light switch IS activated
		OR
		motion sensor IS activated
	IS 0% OTHERWISE
```
Using the language of events, how would *you* do this?

You might naïvely try this set of triggers and events. (again, not real code):
```
WHEN motion sensor SENDS TRIGGER activated
	SET brightness OF light TO 100%
WHEN motion sensor SENDS TRIGGER deactivated
	SET brightness OF light TO 0%
-------------------------------
WHEN light switch SENDS TRIGGER activated
	SET brightness OF light TO 100%
WHEN motion sensor SENDS TRIGGER deactivated
	SET brightness OF light TO 0%
```
If you took just the top half or the bottom half on its own, this would work perfectly fine.  The problem is that these two halves can't be combined; I can use the switch to turn off the light while motion is still being sensed, or (more problematically) the motion sensor can turn off the light while the switch is still on. The state of the light becomes a 'race' between the switch and the motion sensor.

If you keep track of the state of devices, you *could* just about make it work:
```
WHEN motion sensor SENDS TRIGGER activated
	SET brightness OF light TO 100%
WHEN motion sensor SENDS TRIGGER deactivated
	SET brightness OF light TO 0%
		ONLY IF light switch IS NOT activated
-------------------------------
WHEN light switch SENDS TRIGGER activated
	SET brightness OF light TO 100%
WHEN motion sensor SENDS TRIGGER deactivated
	SET brightness OF light TO 0%
		ONLY IF motion sensor IS NOT activated
```
However, this is the start of something gross and unmaintainable. Every time we add some new device to this system, or want to add some new condition, you have to add extra logic *in multiple distant places*.

What we are doing here is re-treading the exact same path that programming itself has walked. Smart home automation is currently somewhere in the 1980s. If we want the smart home to do more, we should skip through a few decades of invention and get to the good part.
## A solution from UI systems

Most of my computer science work up to this point has been researching the theory underlying UI systems. These are incredibly complex and hard-to-work-with systems, owing to the massively distributed nature of individual UI components, the blending of local and global responsibilities, errors cascading out of control from distant parts of the system, and the need for a fast and frictionless developer workflow to keep up with ever-increasing feature demands.

All of that is to say, if we're going to learn anything about managing complex systems, it is a fantastic idea to apply what we have learned about managing user interfaces.

The golden rule is that we should program systems with the *results* that we want to see, and allow the computer to infer *what exactly* we need to do to get there. Computers are good at figuring out efficient paths from A to B, so we can trust them to construct a path from our current state to the final state that we want.

Most commonly, this principle is implemented using *reactive programming*. Reactive programming defines everything with logic *exclusively*. In particular, you do not "set" things at a point in time; instead, you define some kind of logic. When you need to know what the thing is, you run through the logic to determine it.

The very first logical statement in this blog post can be considered a form of reactive programming. Here it is again:

```
brightness OF light 
	IS 100% WHILE 
		light switch IS activated
		OR
		motion sensor IS activated
	IS 0% OTHERWISE
```

Whenever the system needs to know what the brightness of the light is, it can run through that logic to determine it. In particular, the computer knows that the brightness of the light depends on both the light switch and the motion sensor, so it knows to run through this logic whenever either of them change state.

Of course, there are ways of making that much more efficient, such as caching previous values, but fundamentally, this is a much more intuitive mental model. We use this all of the time in UI system programming to great effect; some of the most complex systems run on logic implemented exactly like this, and we've built incredible ways of accelerating this work to run within milliseconds.

Why aren't we building our smart home software to work like this? In the Reddit post I originally linked, the author agrees that this is a much easier way to work. It's impossible for devices to "race" each other because the logic runs every single time, giving one consistent answer. In general, reactive programming eliminates several whole classes of contrived edge cases, which is why we use it in the construction of those UI systems in the first place.