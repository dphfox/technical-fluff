---
layout: post
title: In love with Roblox audio
image:
  path: /assets/posts/in-love-with-roblox-audio/thumb.jpg
  width: 256
  height: 256
---

I'm currently building something in Roblox, and I can say with confidence that the new Audio API just totally changed the sound game.

Throughout my place, I have speakers set up on the walls.

![Wall mounted speakers in an interior.](/assets/posts/in-love-with-roblox-audio/speakersys.jpg)

Normally on Roblox, these sorts of details are essentially just visual flair - you'd have a globally playing music playlist, with perhaps a touch of reverb or EQ if you're *faaaancy*.

<figure class="aside">
    <img src="/assets/posts/in-love-with-roblox-audio/auxsetup.png" alt="The Explorer view of the speaker setup.">
</figure>

But, get this - these speakers are now totally functional. Every speaker is equipped with an `AudioEmitter` and a `Wire` tagged specially to indicate that it should be plugged into the music system.

<figure class="aside">
    <img src="/assets/posts/in-love-with-roblox-audio/listensetup.png" alt="The Explorer view of the listener setup.">
</figure>

Elsewhere, I have a script that manages the global music playlist, and a separate script that inserts listeners for different sound channels and runs them through a bunch of DSP.

All of that is wired up at runtime with the super neat modular `Wire` system that Roblox introduced. The music player script need not concern itself with the details of how the sound will be listened to; it just manages a music player and plugs it in to all of the tagged wires. Likewise, the listening system doesn't care about where sounds are coming from at all. It's incredibly well thought through, and makes it very easy to route audio through a minimal number of places!

Having speakers accept a `Wire` input also allows for individual speakers to add on their own custom DSP incredibly simply! For example, I have speakers in elevators which I want to sound tinny and small. I can pull audio from a tagged wire just as any other speaker can, but before piping it through to an emitter, I can process it with some EQ directly inside the speaker; completely encapsulated from everything else. Modular speaker systems where speakers can have personalities!

And of course, the best part of all of this is that you end up with perfect audio sync - zero flanging or other artifacts stemming from having multiple players out of sync.

Can we have more like this, please? It's delicious.