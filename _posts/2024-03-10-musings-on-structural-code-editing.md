---
layout: post
title: "Musings on structural code editing"
image:
  path: /assets/posts/musings-on-structural-code-editing/thumb.jpg
  width: 256
  height: 256
---

One of my long term aspirations is to design the structural code editor that actually sticks, mostly because I don't entirely vibe with text. It's not something I'm working on today, but I have a couple of opinions about how it should be done that I'd like to verify or discard eventually.

## Normalised **formatting**

![Hand-drawn diagram showing different formatting for viewing and saving.](/assets/posts/musings-on-structural-code-editing/formatting.png)

Tabs versus spaces is noise. You should not care about the primitive that's used to move something some number of pixels on the screen. That applies to pretty much all code - you should care about the logic, and the editor should handle the presentation of that logic for you. To some extent, this is the unifying aim of structural code editors as a whole.

You should be able to work in a code base that mandates some style guide, while simultaneously still being able to adjust the way you personally view your code files to suit what works for you as an individual. The format of saved code should not have to equal the visual layout of code on your screen. This should ease debates around things that don't matter by giving individuals agency over their own experience.

Automatically formatting your logic would free developers from having to consider anything that isn't directly contributing to the function of the code. You wouldn't want formatting to change while you're actively editing, of course - you want the code to predictably and consistently respond to your actions - but perhaps whenever it's off-screen or no longer focused for editing. You'd want to take the benefits of structural editing removing presentation concerns, without hampering the editing experience with odd layout changes.

## Text-like editing

![Hand-drawn illustration of text-like multi-cursor editing.](/assets/posts/musings-on-structural-code-editing/textlike.png)

I consider the fundamental property of a structural code editor to be that it internally deals with a structure rather than text. That does not, however, mean that the editor interface has to strictly abide to that same structure. 

I envision a spectrum upon which editor interfaces can lie: on the loosest end is editing source code as pure text, which lets you as the user completely ignore the underlying structure. On the strictest end is editing source code as pure structure - that is, your editing actions will never result in a program that's unrepresentable.

Strict editing definitely has its virtues for ensuring correctness, but I'd argue it's too strict to mandate that programs always conform. Being able to leave a program in an invalid state is valuable because it reduces the number of steps to refactor a piece of code, and lets you do so more directly based on what you're looking at on-screen.

I'd draw a comparison to music notation programs. Strict editors are like Dorico, which enforces that every note is 'just a duration', and formats the output for you. This is convenient unless you want to manipulate the output directly, at which point it introduces a new layer of abstraction that you wouldn't get with a more direct editor, such as Sibelius or MuseScore. Users report that Sibelius and MuseScore can be quicker to produce notation for that reason.

So I'd propose that principle applies here, and that it's most efficient to allow users to manipulate the code *as if* it were text, at least to a large enough degree to allow flexible refactoring options. Programmers flip text far more than other professions do; it's not unheard of to find-and-replace phrases that themselves don't have a neat structural analogue, for example substrings or parts of expressions. I think there is value in transferring at least some of these ideas into the structural world.

Perhaps, like in the browser tools, you could choose to edit a block as text when you want to do some more hardcore refactoring. Perhaps you could 'pin' some blocks as text while you're in the middle of a refactor - bind it to something convenient like triple click for efficiency.

This perhaps contradicts my 'normalised formatting' musings, but hey, I think it's important to still have some element of freeform editing.

## Beyond vertical

![Hand-drawn diagram showing 2D-positioned blocks of code with links between them.](/assets/posts/musings-on-structural-code-editing/beyondvertical.png)

Screens are two dimensional. Code is kind of one dimensional. To be clear, that's totally great so long as you're working with control flow, but when you're dealing with a bunch of mostly-order-independent blocks like function definitions, it's just limiting.

Imagine if you could arrange your function definitions like you'd arrange an artboard. Put related blocks of code next to each other to make their similarities more visible. Or perhaps you could summon the source code of a function you're calling, and have it float to the right of the code block you're currently editing. Perhaps you could visualise how your functions nest inside of each other. Maybe you'd want to bring in a block from another file just to have it for reference. 

This feels like the biggest missed opportunity to me. We spend so much time scrolling up and down code files, while simultaneously singing the praises of those few files that can fit on a single screen. We have huge monitors nowadays, and it feels like wasted potential to not use that space to get a better overview of what we're doing all at once.

-----

I'd probably have more thoughts than this given enough time, but these are the main points of my belief that structural code editors could be great one day. I'd love to explore these directions.