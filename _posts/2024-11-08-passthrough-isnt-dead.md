---
layout: post
title: Passthrough isn't dead
image:
  path: /assets/posts/passthrough-isnt-dead/thumb.jpg
  width: 256
  height: 256
---
Meta's AR tech demo made the Vision Pro look ancient - but that doesn't mean Apple bet on the wrong horse.

I believe that passthrough VR might not be so far off what we end up doing with AR glasses in the future, to address one of their fundamental weaknesses.

Let me explain.

## Normal optics

I won't spend too long on this part, because it's well-trodden territory. Let's start with a normal VR device, and look at how it gets light into your eyes.

![Wearing the Quest Pro.](/assets/posts/passthrough-isnt-dead/quest-pro.jpg)

In front of each of your eyes, you have a tiny display, which emits light to form an image, just like any other display you've ever used on a computer.

However, the display will be much too close to your eyes for you to get a clear image on your retina, because your eyes expect light to be coming in from much further away. So, a lens is used to shape the light rays so that the image is clear when it hits your retina.

![A simplified diagram of how VR optics work.](/assets/posts/passthrough-isnt-dead/vr-diagram.png)

That's it... minus all the tiny details required to land you a nice job as a hardware designer.

Fundamentally, every headset on the market - including the Vision Pro - work this way. There's no way for you to look "through" the headset at all; you are looking at a display.

So how come you can see the world outside of your Vision Pro?

## How passthrough works

The answer is a technology called "passthrough". The idea is simple; use cameras to show what the outside world looks like on the displays. If you do it right, then the light going to your eyes should look the same as the normal light that'd have come from the room.

![A simplified diagram of how VR passthrough works.](/assets/posts/passthrough-isnt-dead/vr-passthrough-diagram.png)
This is how the Vision Pro works. It uses a bunch of camera feeds to figure out what light would normally be reaching your eyes. Then, it uses some reverse logic to figure out what part of the display should emit that light. When the display shines that light, it hits your eye from the correct angle and makes it look like you're staring at your room, rather than a headset.

This approach has many upsides - mainly, you get full control over what light goes to your eyes. Because the light is blocked by the headset, it's up to the display to recreate all the light that should be coming in. But of course, that also means you can modify the light, painting on your own objects and scenes to make them look real.

The Vision Pro decides to paint on a bunch of glassy apps and 3D objects to make it look like you have multiple large displays floating in your room.

![A photo of visionOS.](/assets/posts/passthrough-isnt-dead/vision-os.jpg)

Still with me so far?

## How AR glasses work

Let's now talk about how Meta's AR glasses work. Meta would like you to think that nothing else like them exists, but there's a few companies making AR glasses today.

Here's my favourite pair, the XReal Air 2 Pro glasses.

![Wearing the XReal Air 2 Pro.](/assets/posts/passthrough-isnt-dead/xreal-air.jpg)

<figure class="aside">
    <img src="/assets/posts/passthrough-isnt-dead/xreal-underside.jpg" alt="The underside of the XReal glasses.">
</figure>
These are actually based on a similar concept to VR headsets. The goal is to shine light from a display into your eyes, with the light coming in at just the right angle to look like it's coming from inside of the room.

If you look at what's behind the front glass, you'll see a pair of transparent glass prisms. These perform a similar task to the VR headset lenses from before; they shape the light coming into your eyes from a display.

A display? What display?

With enough effort, you can find it. Look up under the glasses while you're sending an image to them, and you'll see a pair of tiny OLED displays embedded in the top of the frame, seen through the prism. 

These displays shine downwards, reflecting off of the slanted glass prism and shooting off towards your eyes, with the same carefully-crafted shape you'd get from a VR headset.

If you get up close with a camera, you can see just how small these displays really are - yet they're 1080p!

![Wearing the XReal Air 2 Pro.](/assets/posts/passthrough-isnt-dead/xreal-display.jpg)

This approach has many downsides, which I won't elaborate on for now. However, it has one very large upside - there is nothing in front of you except for a transparent glass prism, so you can let in *real* light. You can see the room you're in *as well as* the light coming from the displays!

![A simplified diagram of how AR optics work.](/assets/posts/passthrough-isnt-dead/ar-diagram.png)

## The occlusion problem

This setup works marvellously, except for one slight issue.

Because we are letting light in from the room, that light is mixing with the light from the displays. More specifically, the display can only add light on top - there's no way to block any of the light that's coming in from the room. This means that everything you show on the display will look "ghostly". 

As an example, here's what I see when I open up XReal's app.

![Ghostly UI elements in XReal's app.](/assets/posts/passthrough-isnt-dead/ghosting.jpg)

Look behind the large tiles, and you'll notice that you can see my lights and wall decorations through them. Everything is slightly semi-transparent, because the light from the display is mixing with the light from the room, a bit like when you look through a window and see your reflection mix with the room on the other side of the window.

This is what I call the "occlusion problem" - you can't block out any of the light from the room. In particular, this means that optical AR tends to struggle outdoors in sunlight or in bright rooms.

Currently, XReal's solution to this is to use electrochromic dimming on the front glass, blocking the light to create a dark backdrop so you can isolate the digital content. This kills the utility of these things as AR glasses, but it means you can see your content clearly. It's basically a VR mode.

![XReal glasses at minimum and maximum dimming.](/assets/posts/passthrough-isnt-dead/xreal-dimming.jpg)

Would it be possible to dim only parts of the glass, perhaps like an LCD display? Yes, but there's another problem at play here - the glass is so close to you that it will be out of focus. 

Your eyes will be focused on the contents of the room (and the light coming from the display as if it was in the room), so the glasses themselves will appear blurry. That includes the front glass that'll be doing the dimming.

![The glasses look blurry when you look through them.](/assets/posts/passthrough-isnt-dead/blurry.jpg)

This means, at best, you'll only be able to do very approximate local dimming. This might be enough for improving legibility, but it won't give you any crisp edges on objects. Everything would either look like it has an aura of darkness, or it'd have a soft, ghostly faded edge.

Solving this problem likely requires somehow blocking or interfering with the incoming light in some really precise, tightly-calibrated way. Nobody really knows how to do this yet. 

And yet, If we don't solve this problem, then the quality of optical AR will never match the experience of a headset using passthrough.

So, what to do?

## Solving the occlusion problem

High-end AR glasses, like the more modern XReal Air 2 Ultra glasses, now come with integrated cameras for position and hand tracking. It's not hard to imagine that these cameras could one day capture the appearance of your surroundings; after all, we have great cameras in our phones right now, and we're surprisingly good at miniaturising technology.

![The XReal Air 2 Ultra and Meta Orion smart glasses.](/assets/posts/passthrough-isnt-dead/new-ar-glasses.jpg)

So, my proposal is this: when we have the technology to provide a large-FOV pair of glasses with a dimmable front surface, and when we have the technology to capture what the environment looks like, what's stopping us from doing passthrough like VR headsets do?

![A simplified diagram of how AR passthrough could work.](/assets/posts/passthrough-isnt-dead/ar-passthrough.png)
To me, blending "natural light" and "display light" is a pragmatic way to solve the problem. While you would need well-calibrated and decent-looking cameras, and you'd need to make sure that things align decently well between the two sources of light, this could end up being an interesting solution to the occlusion problem.

AR passthrough is strong because software is strong. With passthrough VR, we have experience stitching together images on-the-fly. The Vision Pro already shows good competency with image processing here.

Passthrough also avoids the trap of special hardware. While it may be possible to solve this problem with novel methods, those solutions will tend to be expensive and bespoke. Meanwhile, camera systems and dimmable glass are both transferable across product categories, and likely will be included with the product anyway. You don't need specialised support beyond perhaps having a better array of cameras.

The best part is that the problems with approximate local dimming can be compensated for. Instead of having a shadowy aura around virtual objects, input from the cameras can be used to compensate for the decrease in brightness around objects. The end result is a crisp, sharp edge.

Admittedly, I'm biased as a computer graphics guy, but having thought about this for a long time, I don't see any simpler solution without inventing wild new hardware.

## Conclusion

So, was Apple wrong to invest so much into the Vision Pro? Surprisingly, I think it was a good idea.

By going with a passthrough system from the beginning, Apple can now invest in their passthrough processing pipeline and refine the quality of the output. Simultaneously, they can move towards smaller and smaller camera hardware. And when they do release glasses, they will have a head start on solving the occlusion problem.

There are some caveats to this idea, of course. For one, it would require that you can display a high-FOV image, which seems to be within the realm of possibility at this point given the recent advances by Meta. 

It'd also require good quality camera feeds, which is both a hardware and image processing challenge. It's possible that dedicated image processing silicon could make the leap to AR glasses, but it's not a certainty given the space restraints, even if/when a computing puck is introduced.

And of course, there's always the possibility that something genuinely new comes along and makes this all possible. Optics isn't a solved field, after all.

But overall? I would not be surprised if passthrough AR was the direction things went, even for optical AR glasses. At least, that's my take on it as a nerd. 

I'm eager to hear other ideas.