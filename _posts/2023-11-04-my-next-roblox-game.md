---
layout: post
title: My next Roblox game
image:
  path: /assets/posts/my-next-puzzle-game/thumb.jpg
  width: 256
  height: 256
---

Ever since working on Agnosia and Interval, I've been burning to make a sequel. I could make it as a traditional game, but I'm somehow drawn to platforms like Roblox. Here's some thoughts about how I might make it work.

## Singleplayer fights the social world platform

Both Agnosia and Interval are first-person, singleplayer, desktop-oriented puzzle platformers. I'm ever so proud of how they turned out, and I would absolutely love to go ham on a singleplayer title, but this is precisely the sort of game category that makes zero sense on social world platforms such as Roblox, VRChat and Resonite. 

And yes, I'm coining 'social world platform' in this post, because I don't actually think anyone knows what 'metaverse' is anymore (thanks, marketing). I think this is a concise and accurate way to communicate the common thread I see in these platforms.

Anyway, the name clues us into a truth about these platforms. People *can* explore them alone, but often they *prefer* to do it with friends. In fact, there's often a whole layer of emergent gameplay from social interaction. A while ago, I said to someone "you could describe a surprising number of Roblox experiences as overgrown party videogames" - think Plates of Fate or Freeze Tag. The social experience is often the point of these games; nobody would play singleplayer Plates of Fate, because the point of that game is to flail in the chaos alongside everyone else.

So I think it makes the most sense that my next Roblox game incorporates this 'social gameplay' aspect. It shouldn't be a singleplayer isolated experience, even if that's what Agnosia and Interval were, because it wouldn't be as compatible with the medium as it could be. A singleplayer game may impress some, and perhaps it would satiate own desires, but ultimately it probably wouldn't gel well with players. I think it's a much more interesting and fruitful design space to try and merge multiplayer with puzzle mechanics. The question then becomes "how exactly do you merge these concepts?".

It's an incredibly interesting design space because of the kind of multiplayer gameplay that these platforms offer. Unlike traditional puzzle games, which usually only offer 2- to 4-player co-op at most, we're talking about a sliding scale that could be anywhere from two or three close friends getting together in a private server, to 50 strangers with no previous affiliation and no common baseline being assigned to each other by a public server matchmaking algorithm. You need to somehow protect against disinterested, ephemeral strangers who may troll, may have different skill levels, or who might just enter and leave at any time due to different schedules and timezones. At the same time, you don't want to protect so hard against these things that the game gets in the way of truly co-operative, satisfying multiplayer gameplay. You can't make players depend on each other, but the more they do depend on each other, the more truly social it is.

This is part of why NPCs can work so well for these games. I've been playing The Talos Principle 2 recently, and it is *really nice* to be able to depend on the other characters in that game to help you out, and really helps bring the characters alive. Perhaps NPCs can be part of the solution, but ultimately they'd have to complement whatever actual solution I come up with for dealing with unreliable players.

## A puzzling content problem

There's a further issue with multiplayer puzzle games; they work optimally when there is minimum pre-existing knowledge. Puzzles don't have much replay value, because the process of figuring them out is largely a one-way street, meaning you don't usually get the satisfaction of completing them more than once. Instead, you would have to derive joy elsewhere, for example using your knowledge to help someone else solve it. 

This does not bode well for the 'multiplayer puzzle' idea. When the appeal of puzzle games comes from an irreversible process, how can a finite body of content keep generating new appeal? One answer would be contingent on some emergent social gameplay, where value is derived from inter-player interactions rather than the mechanics of the game itself. This could work, but I feel like it might dilute the game down too much for my liking. Another solution is to just keep churning out new content to throw on top of the content pile, but I don't want to become enslaved to a tight content update cycle.

An alternate option - and hear me out, because this is going to sound batshit insane - would be to make the content pool fundamentally infinite using procedural generation. It's a proven tool used to fill content gaps and solve re-playability problems in other genres. What would it look like here?

Let's go back to me playing The Talos Principle. I had a recurring thought as I was solving some of the puzzles. I realised that, even though the positions of these puzzle pieces were represented with non-discrete positions, fundamentally the puzzles had a finite, discrete set of states, represented by certain objects being in certain regions and having certain configurations. You could describe the different states of a puzzle like a sort of state machine, where the possible movements between adjacent states would represent possible courses of action to mutate your current state into a new one, for example moving an object into a new location, or reconfiguring it in some way. In the case of The Talos Principle, I imagine states being defined by 'visibility regions', the position of the player, laser connections, jammed objects, et cetera.

I hypothesise that it should be possible to 'discretise' almost any kind of player-solvable puzzle into these sorts of discrete state machines, because we depend on players building up such state machines in their head and exploring them to discover solutions. At minimum, this would be great for finding solutions to puzzles, because pathfinding through the different states of the state machine would allow you to find optimal solutions algorithmically.

This doesn't actually address how to procedurally generate puzzles though. For this, I think back to simpler procedurally-generated puzzles I've made before, specifically an implementation of a Lights Out mini-puzzle. I wanted to randomly generate puzzle boards which always had a solution, without just randomly flipping grid cells on and running a solution checking algorithm to verify it was solvable (which would be too expensive to run on the fly). The solution was to work backwards from the 'success state', taking the same actions as would be possible to the player to forge a (hopefully) winding and interesting path through the state machine. This would then manifest as a randomly-generated, always-solvable puzzle ready to present to the player, generated with more controllable and less explosive performance characteristics. If you really wanted to try and ensure an interesting journey from start to finish, you could try employing some kind of intentionally-bastardised pathfinding algorithm and giving it the state machine, to find a sufficiently long and winding path from the starting state to the finished state.

The only prerequisite for this to work is to have a sufficiently interesting state machine to path the puzzle through. For Lights Out, this is just defined by the grid the puzzle sits upon, but there's no reason we couldn't procedurally generate this state machine too. I don't see any reason why you couldn't build one of these state machines pseudo-randomly - almost like putting together pieces algebraically - then converting that into some kind of playable puzzle room or screen based on some layout rules.

Not to say any of this is trivial, by the way! I suspect for something as three-dimensional as Portal, for example, it'd involve a shit ton of geometry. But I don't see any reason for it to be fundamentally unsolvable. Perhaps I haven't thought about it deeply enough. Maybe there's something NP-hard waiting for me around the corner. Who knows. But it's fun to think about.

By the way, I want to take a second to talk about human authorship. I acknowledge the argument that part of what makes puzzles satisfying is that a level designer can carefully craft an 'aha' moment, assess difficulty, stuff like that. I would, in fact, agree with this sentiment. However, I don't think it's necessarily grounds to reject computer-generated puzzles. While computers maybe can't think about puzzles like humans do, maybe that's also a virtue, because it means computers don't fall into the same tendencies as humans do, paving the way for more surprising or lateral solutions, perhaps a different kind of 'aha' moment.

## VR reveals the hidden cost of enjoyable cross-play

I want to take a quick diversion to talk about VR. It's my preferred way to play games by far; if a game offers both VR and non-VR editions, I pick the VR version every time, because it's simply more fun to me. I do think VR has a content problem though; there aren't enough truly compelling VR games out there, so I wanted to try and make one myself. This idea kind of merged itself into my puzzle game ambitions over time.

There's one quote that still resonates with me to this day though. The developers behind Half-Life Alyx said something to the effect of: "VR games need to do things that only VR can do, in order to be truly compelling over traditional games". Viewing the VR medium through this lens, I immediately see what they mean. My best experiences in VR has been with games like The Last Clockwinder, which fit the medium like a glove and subsequently gave me some of the most unique and memorable play sessions I've had.

Fundamentally, a great VR game will probably rely on two things. One is the immersion factor of head mounted displays, and the other is the direct tactility of tracked motion controllers. The former is easy to achieve, because VR has the strongest sense of immersion of any medium practically by default, but the elusive latter helps make for more viscerally enjoyable core gameplay loops in VR. It's what makes both Half-Life: Alyx and The Last Clockwinder so completely amazing to play. They *feel* great in a manner that just couldn't be replicated any other way.

Again, though, this poses problems for this 'multiplayer puzzle game' idea. While VR is amazing as a medium and I would easily recommend everyone try it, the reality is that it's still a small market dominated mostly by small titles. Roblox's VR audience is not very expansive compared to most other demographics. VRChat would be a better fit here since they grew from VR roots, but also Udon is still primitive compared to what Roblox can do, so I probably wouldn't be quite ready to jump ship from Roblox just to pursue the easy cop-out of a VR exclusive.

Instead, I think it'd be interesting to try and solve for good cross-play between VR and non-VR players, so they can still coexist in the same social space. Easier said than done, though, because VR has an entirely different set of strengths and weaknesses to traditional platforms, so I'd need to craft a different VR experience for it to feel truly great, rather than being a mediocre translation of what was already there.

The more cynical reader might, at this point, exclaim: "This is more work than it's worth! Throw away your VR ambitions, because they're too much work!". And perhaps you're right. Maybe I should. But unfortunately, I realised that this isn't actually a VR problem at all. This is just generally a problem with *any* sort of cross-play between different device types. Think about phones, for example; don't those have distinct strengths and weaknesses compared to flatscreen PC, too? First person controls on mobile *suck*, but there's a whole world of interesting possibilities when you have multitouch, gyroscope and accelerometer inputs available. Does that not sound analogous to VR's motion tracked controller strengths, and its locomotion weaknesses?

If I want to make a puzzle game worth playing on every device it's available on, then I need to make sure that every device has gameplay that makes sense for it and plays to the device's strengths while avoiding its pitfalls. I don't really hear this get talked about often; for example, I feel like a lot of people just treat designing for consoles or mobile as nothing more than having your inputs automatically rebind based on the last input type or whatever.

## Closing thoughts

I don't really have any good closing thoughts. I just think these are interesting ideas to ponder. Maybe exploring these design spaces will yield interesting and unique ideas. Or maybe they'll just run into intractable, impossible-to-compute solutions.

But I'm down for a puzzle.