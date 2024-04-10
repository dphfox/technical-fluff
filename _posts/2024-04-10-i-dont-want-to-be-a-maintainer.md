---
layout: post
title: I don't want to be a maintainer
image:
  path: /assets/posts/i-dont-want-to-be-a-maintainer/thumb.jpg
  width: 256
  height: 256
---

In a world where [that xz incident](https://en.wikipedia.org/wiki/XZ_Utils_backdoor) occurred, I've been thinking about my relationship to OS.

## I still want to share my work with the world

Because that's the best part of building things - seeing what others do with them. I really do enjoy being part of a broader system and seeing how other people's imagination and vision can build atop the little pieces I contribute.

## But I don't want the world to depend on me

At least, not every single time I share something. I take most projects public because "there's no reason to keep this to myself". That's not the same as "I want to maintain this for thousands".

To be clear - I do still want to maintain *some* things! Fusion is a joy to work on. But do I have that same enthusiasm for all the dozens of tiny utilitarian code modules that I put out there?

## Here's how I see things

The open source movement has conflated open source licencing with open source maintenance. I think there should be room for one without the other, a more tepid level of commitment where the user is expected to bear the cost of maintenance rather than the original creator. If information is to be truly free, then the cost of publishing it should be free, too. **There should be no default expectation that [a random person in Nebraska](https://xkcd.com/2347/) will maintain infrastructure you chose to depend on.**

If you want to bear the cost of maintenance for the world, then good on you, and go full steam ahead. If you don't want that, then that should be okay too. We need our 'snippet sharing' culture to grow beyond low-quality StackOverflow answers - free experimentation and information sharing should be the default way we do things, with the best discoveries transformed into maintained projects either internally at companies or out in the open by truly motivated maintainers. To continue on our current trajectory of unpaid, borderline-exploitative open source culture is to be on a collision course with disaster.

## So here's what I'm doing about that

I promised that the [next-generation Elttob Suite](https://elttob.uk/go/suite/) would open up some of its modules to the world with an MIT licence. You might think that means I'm about to upload a bunch of new GitHub repositories, but I have different plans.

I want to experiment with 'mirroring' these modules instead. I don't want to spend time managing an open source community around each and every one - I maintain these modules in-house anyway as part of my own work, so the most likely outcome would be that dealing with contributions would involve a lot of work in enforcing guidelines and specifications, and not a lot of work actually getting committed in from the outside. Fusion already struggles with that same problem, to a certain degree.

So I'm just going to make them available, because that's the part I care about. I want you to be able to take them, fashion them into your own, and use them to build systems of your own. But I don't want to be implicated in that process, because it will be less efficient for the both of us, and it'll create an unhealthy dynamic leading down the road of burnout and increased vulnerability.

If I'm happy with the outcomes from this experiment, then I'll likely start moving other projects in this direction too. I'm idly thinking about whether I should continue to accept code contributions for Fusion, or whether the real value is derived from talking to the community and authoring specifications in collaboration instead. I don't accept a lot of code contributions because I want to enforce a specific style and quality of code such that I'm not yet comfortable delegating to others, but I love talking about potential API surfaces and working through users' problems with them in text form.

We will see whether this works.