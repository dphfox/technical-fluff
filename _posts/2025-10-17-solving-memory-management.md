---
layout: post
title: Solving memory management
image:
  path: /assets/posts/solving-memory-management/thumb.jpg
  width: 256
  height: 256
---
I have devised a pretty exciting memory model for [Wolf](https://wolf.phfox.net/). If it works, it should be provably safe, scale close-to-optimally, and add zero language complexity.

I recommend you [go watch this revelatory talk by Ryan Fleury.](https://www.rfleury.com/p/enter-the-arena-talk) The chain of logic they describe in that talk reflects pretty much my entire history with memory management up to today. I'll continue this post assuming you know what I'm talking about.

## The joy of arenas

Back in 2023, while working on Fusion, [I rediscovered arena allocation independently in a GC language.](https://fluff.blog/2023/08/30/the-next-ten-years-beyond-maids.html) Without anyone else ever having explained the concept to me, I immediately saw just how much more powerful structured memory management is, and how it leads to programs that are both trivial to reason about, and also fully deterministic. 

It's so utterly powerful that I managed to implement (what Fusion calls) *time-travelling checks* to predict use-after-frees, which have been a revelation for catching edge cases that I otherwise would have missed.

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/QTTRtzZxew4?si=lDU5XbiDLXRWNVKH" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

It turns out these ideas are not original in the slightest! Ryan Fleury, Casey Muratori, Johnathan Blow and the rest of the gang have been out there for years singing the praises on arena allocators, and my lived experience dealing with all of this is all the proof I need that these ideas are every bit as good as each one of them said they were.

I love arena allocators. They're everything I've loved about "maids" in the Roblox ecosystem for years, and they're almost word-for-word identical to "scopes" (which are really just properly-done arenas that run destructors instead of freeing memory).

## The rub

There's only one problem with arena allocators. It's not even really _that much_ of a problem, but it's still worth mentioning IMO.

Arenas are still visible in your code. (Boo hoo, right? _Unusable._)

When I designed "scopes" for Fusion, that was a feature - it declared a code path's intent to allocate, and allowed you to declare their lifetime explicitly in your code, preventing whole classes of bugs. For any language that exists today, I think it's the right approach.

But really, I've been looking for a way to make code _look_ every bit as clean as garbage collection, while using a deterministic structured approach to memory management that can be analysed ahead of time and trivially checked for safety. I don't know if today's languages are up to that task without doing some gross things that I'd never recommend over simply _writing the damn thing_.

## Wait, I'm writing a language already

Yeah, turns out I'm the benevolent dictator of my own still-in-flux language. And, turns out I've already been stewing on this problem in a different _context_.

One of the top features I've wanted in Wolf (my language) are implicit function parameters, aka _context_, so that you don't have to do "prop drilling" to transiently pass relevant wide-area information down to deeply nested functions. I got the idea from my frontend UI experience, ironically enough - it's a feature I implemented and loved in Fusion, which itself was somewhat stolen from React. (Hey - React's useful for something!)

<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/dOkoXYCpxMM?si=KCSyNQ2fVZeIASlH" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

My _original_ line of thinking here was that it'd be _really nice_ if you could contextually define an arena allocator to use inside of a function, a bit like how Jai (Johnathan Blow's language) lets you do the same thing. That would make it a lot more ergonomic to have dynamically-sized literal constructors in Wolf - no need for any special syntax to tie them to a specific arena allocator. It also makes all functions generic over allocation strategy by default, which is _rad_.

However, I realised something pretty nice about Wolf. Since it's a highly declarative programming language without things like mutation, pointers or references baked deeply into it, everything is lexically scoped. In theory, that makes the whole language pretty compatible with stack allocation, as you might find by default in C. It's not a 100% solution, but a good 80% solution.

I was... half satisfied. Then I thought a little harder about what stack allocation _really_ is.

## Oops! All arenas

Yeah, turns out stack allocators are just mini little arena allocators, aren't they?

This is about when I realised that Wolf could have a really nice generic allocation strategy that does the same thing as a stack allocator, but doesn't prescribe use of a stack per se. You just implicitly create a scratch allocator for each block, and if it passes any values out of itself, then you implicitly parameterise the block with an allocator to use for those values.

```
-- Pseudocode
fn [.return_alloc : allocator] (
	let local_alloc = allocator
	-- Allocated temporarily for the duration of the block
	let needs_memory = malloc [.name "steve", .age 27] in local_alloc
	-- Important: use the return allocator here
	let will_return_this = malloc [.name "baxter", .age 5] in return_alloc
	free local_alloc -> will_return_this
)
```

Via induction, since all Wolf programs are composed from blocks, you can now guarantee that all memory is allocated and deallocated correctly (if sometimes conservatively, and with some extra nuances/implementation details that aren't relevant here).

There's even more nice things where that came from, too. You don't strictly have to use one allocator per block; you could use as many as you like, wherever you like, because none of those details actually appear anywhere in the syntax.

For example, you can determine when the last time a value from an allocator is used in a block, and at that point, you can choose to drop the allocation right away. Alternatively, you could choose _not to_ if you locally know it'll get dropped by an allocator in a higher lexical scope. In other words, you can fit your arena allocator lifetimes more tightly or loosely to your program depending on context, all completely automatically.

What's more, you could decide to use a right-sized allocator just for the parts of the block where the sizes of values are statically known, and use a more flexible allocator for the unbounded dynamically-sized things like user strings. (The fact that Wolf intentionally tries to be as compile-time-analysable as possible helps massively here.)

Oh right, flexible allocators for dynamically sized things... shoot.

## Who manages the manager?

This model of memory management in Wolf works perfectly well _in theory_, when you have arenas that are _assumed to be_ growable and flexible, and when you have infinite memory space.

In reality, you get a finite amount of virtual memory address space, and of that, a dismal fraction that's available as physical memory.

At this point, I was thinking about allocation strategies for arena allocators that needing to be dynamically growable, or perhaps having a dedicated allocator _just_ for growable arrays of some kind, which could then be used as a springboard for implementing dynamically sized arena allocators.

I was decently happy with that idea. That's about when I discovered Ryan's talk.

## There's hardware for that, dummy

I had overcomplicated the whole thing because I wasn't thinking about how the OS actually manages virtual address space. You can use the hardware's MMU to just... reserve tons of contiguous memory and forget about it. It's not like it's going to be backed by any physical memory until you actually use it, so it's free real estate.

So, the way Ryan describes the technique: make every arena allocator gigantic in virtual memory, then just forget about it. Sure, you'll have a maximum arena size, but it'll be something like 64 GB. No need to sweat over it when it's orders of magnitude larger than the data your program is dealing with, and when you have 256 TB of virtual address space to carve up for arenas. Plus, this is where statically knowing the size of things can really come in clutch.

Ryan says that this gives you ~O(1) allocation and deallocation, all super close to the metal. I'm inclined to believe it, though I'm not about to repeat it without citing Ryan as I haven't run through the whole analysis myself.

Zero overhead allocation and deallocation with fully implicit memory management that's as clean as garbage collected code. I'm dreaming, right?

## Remaining considerations

So obviously, I'm super jazzed about this. From my very brief searches online, I haven't really found a language that does all of this together, in the way I'm describing here. Wolf may well be (one of) the first languages setting foot in this territory.

However, I still have a few lingering thoughts on my mind that I haven't yet cleared up:

-  I suspect that I shouldn't _fully_ rely on the MMU to reserve gobs of virtual address space. I might implement a hybrid solution that chains _large-enough_ blocks of contiguous space instead (something like 1GB chunks?) which might play better on platforms that don't have as many bits of virtual address space.
- I'm also not entirely convinced yet that I can escape doing allocator management this way. I'm still thinking about ways of potentially carving up the address space in a way that can be used for allocating arenas of various sizes, just in case. I don't really know.
- I'm also clear-eyed that people may still want to implement more specialised allocation techniques. My idea is that arena allocators are a perfectly good implicit language features, and that through arena allocation, it should be trivial to manage the lifetime of custom allocators and their reserved memory. So I'm not thinking about them too hard right now.

Overall though? I might have just _solved memory management_.

I've defeated that damn garbage collector.