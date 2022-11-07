---
layout: post
title:  "You don't know performance"
thumbnail: "/assets/posts/you-dont-know-performance/thumb.jpg"
---
> When a measure becomes a target, it ceases to be a good measure.

At my day job, I'm using [Fusion](https://elttob.uk/Fusion/) to build a large,
dynamic education app with large quantities of user-generated content. At night,
I work on Fusion itself, figuring out how to make it easier for people to build
and maintain those kinds of complex interfaces.

Throughout my work on Fusion, even long before it was ever released, people have
cast doubt on Fusion's microperformance. Originally, I ceded to those crowds by
pushing synthetic benchmarks on Fusion and stretching it to the limit, but now I
simply don't think it's worth pursuing. Here's where I disagree about
performance.

## Performance is a problem you have

I think people overplay the performance impact of different frameworks. I know I
certainly used to.

The reality of the situation is that performance issues affect only a subset of
especially complex or pathological UI rendering scenarios. The overwhelming
majority of UI work is boring, simple, uneventful stuff which is easy to do, but
for which you're just looking for a consistent and straightforward development
pattern. This is why I say that framework choice is largely about personal taste;
because you don't want to be fighting against your framework to do 99% of your
work, just because you're trying to eke out a couple milliseconds.

You'll know when you've run into a real performance problem when it shows up in
your tools and tests (you *are* testing, right?) - until then, you should be
focusing on making the thing work in a way which you can maintain.

## Performance is an option you should have

I'm highly sceptical of inflexible frameworks, or frameworks with strong
opinions on how to do things, because restricting what the user can do could
inadvertently lock them out of easily achieving better performance.

A framework worth it's weight should allow you to gradually move towards better
performing code by slightly stepping away from your 'default' solution that
isn't performing well. There should rarely be 'jumping-off' points where you
have to abandon your entire ship just to fix a performance problem, because
usually there's only a specific part of your code that your performance problems
stem from. You should be able to isolate that code and refactor it without
nuking everything it talks to.

This is why I'm such a fan of small, composable APIs that don't live inside of
larger processes; problematic code is far easier to isolate when it's already
completely separable by the design of the API itself.

## Performance is not one thing

There's still something that the above statements don't define. What defines
good performance?

I somewhat hinted at this in the first section, but good performance is any
performance that meets your real-world requirements. There's no absolute
definition, only the definition you decide for yourself based on which platforms
you need to support and what your testing and profiling tools tell you.

Surely, though, there's some absolute way to *compare* performance? If a piece
of code takes 2ms to compute versus 4ms, it's better right?

Again no, because there's different *kinds* of performance. A piece of code can
run fast on the CPU. Perhaps it's better suited to run on the GPU, and will
perform terribly on the CPU. Maybe an algorithm is only sensible to use when
it's being run in parallel threads that can divide and steal work. Or perhaps
you straight up don't care about CPU performance and need something that can run
in a limited memory space, or which writes less information to disk, or reduces
network bandwidth.

Again, if you aren't testing your code and you aren't considering some set of
requirements, *you don't know what kind of performance matters.* Everyone
assumes they need the best CPU time, because framerate, without realising
they've now just eaten up 4GB of memory caching everything.
[Sometimes it's just better to do the maths.](https://twitter.com/transitracer/status/1588316358303010816)

> the knowledge that math is literally significantly faster than L1 cache hits,
> so memoization in many cases is actually a performance /loss/ is making me
> insane

> (btw it's even worse if you bust your L1 cache and end up in L2, or god forbid
> L3 or actual memory, which is, surprise, highly likely because now you're
> storing extra data!)

[Sometimes it's just better to re-fetch the thing.](https://twitter.com/adambroach/status/1588550243196755970)

> More cursed knowledge: at Mozilla, the perf team discovered that it is, in
> double-digit percentage of cases, faster to re-fetch data over the network
> from the original web server than to retrieve it from local disk cache.

It depends on what you need to optimise, and what you're able to give up. So
please stop synthesising arbitrary benchmarks that mean nothing to nobody, and
go figure out what you need before you criticise it for not meeting your
arbitrary metrics.

## What Fusion optimises for

As a framework developer, it's impossible for me to know exactly what kind of
performance you will need. That's why I want to give you flexibility rather than
prescriptions when it comes to Fusion's API surface.

Since Fusion is often used for animation work, I mostly optimise for good frame
times and low CPU overhead, though I keep an eye on memory usage too so as to
not lock you out of using older, lower-end hardware with Fusion. Parallel
execution could be a nice inclusion at some point in the future, but right now
that's something you can express on your own should you need it; I haven't found
a good general pattern for parallel execution that would add enough value to
justify it being there.

I mainly build around the common scenario of random data reads and writes. From
real world UI systems, I know that the vast majority of updates are not total
or huge. Oftentimes, it'll just be tiny things, like typing text into a text box,
scrolling, or single UI elements being toggled on and off. That's because users
don't typically have the power to change everything all at once pathologically.

That's the motivation for things such as memoising individual state objects and
binding to individual properties on instances, rather than using the diffing
processes that are in vogue today. Virtual data models are great when you can't
make strong guarantees, and especially where large volumes of data and change
are concerned, they can absolutely run slimmer than equivalent binding code -
but this neglects the fact that most UIs are heavily limited in the axes they
vary along, which is exactly where diffing performs the worst.

The worst case scenario for binding is when you have a massive number of
instances to process, and every single property of them is changing. The worst
case scenario for diffing is when you have a massive number of instances to
process, and you have to search them all to find and effect a single change. By
the way, when I say 'worst case scenario' here, I'm talking about CPU time that
is wasted above what hand-tuned code could achieve.

There's also the consideration of performance variance. One of the things that
virtual data models can suffer from is wild variance in performance outcomes
from small - perhaps even common - code changes. Diffing processes are easy to
derail performance-wise, because it's easy to express things that require
sweeping recalculations, and it can also be easy to express those things in ways
that shouldn't seem intuitively 'wrong' as far as the user's mental model goes.
I try to keep performance metrics in Fusion relatively stable by making
computations more explicit; Fusion shouldn't go and start eating up resources
unless you've given it some defined reason to do so. 

That affords you the luxury of being more confident in the code that you've
written, which is so much more valuable than anyone gives it credit for. I'm
largely inspired by Rust here, which despite having a very verbose syntax and
slow iteration time, is still my favourite language to work in because when I've
written a piece of code that compiles and runs, I feel like it's already been
battle-tested out of the gate and is absolutely bulletproof. I don't have to
doubt myself after spending two hours on a refactor, because I can trust that
the language isn't hiding nebulous edge cases from me. I want that for Fusion in
some capacity too.

Anyway, maybe this gives you some refreshed ideas on performance. Or maybe it
doesn't. These are just some of the things I've been thinking about lately.
