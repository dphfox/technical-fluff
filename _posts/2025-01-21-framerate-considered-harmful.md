---
layout: post
title: Framerate considered harmful
image:
  path: /assets/posts/framerate-considered-harmful/thumb.jpg
  width: 256
  height: 256
---

I've spent the past little while moving Daydream away from framerate counters and towards "frame budgets". Here's why.

![Daydream using frame and tick budgets.](/assets/posts/framerate-considered-harmful/frame-budget.png)

The primary reason is that it's hard to properly read frame rates, because they are actually a reciprocal unit; to double your framerate, you must halve the amount of time taken to process a frame.

This can be fine in isolation - defining a "target framerate" is still a valid concept, because it's reasonable to want to serve someone, say, 120 frames per second.

But when you try and compare frame rates with each other, or track frame rate through development, you'll quickly find that framerate can be misleading. Targeting 240 FPS, but your prototype is hitting 2000 FPS? Plenty of headroom, surely - but actually, you're already using 12% of the frame!

That's why I'm moving away from the use of *framerate* as a debug metric. It just isn't relevant during development. Instead, I'm measuring what percentage of the frame budget is being used, which I've set at 240 FPS for Daydream (since I have a good graphics card, and I feel like it should be possible).

Having a linear, easy-to-interpret percentage on-screen at all times makes performance usage visible and understandable in a way that - personally - I find framerate doesn't offer. I derived the idea from a friend that used to work at Unity and Halfbrick; when they talk about performance, they always point to two things: set a budget early, and make expensive things visibly expensive.