---
layout: post
title:  "UI frameworks don't serve designers"
image:
  path: "/assets/posts/ui-frameworks-dont-serve-designers/thumb.jpg"
  width: 256
  height: 256
comments_url: ""
---

The pace of innovation in UI framework design has given an outsized advantage to
those who know code. Specialist UI designers have been left in the dust.

I'm far from the first person to raise this issue, but I may perhaps be the most
prominent in Roblox's UI circles. It seems like it's only recently being taken
seriously at all, given that personally every time I've brought it up in years
prior, it's been met with some variant of these points (of varying extremity):

- We can hire a coder to properly implement the design on their behalf
- They don't need the innovations brought to coders by UI frameworks
- We should re-skill and re-educate them, so they can use code frameworks
- It's their problem for not accepting that progress excludes people sometimes
- We don't need them at all, technology has obsoleted them entirely

As a funny side observation, many of the people who've told me that last point
have some of the most dreadful UX I've ever seen, with very strong correlation.
Obsoleted indeed.

So what do I think about the above points, anecdotes aside? Where are the
problems? What should we focus on?

## Non-designers are terrible at implementing designs

Not for a lack of will, either; even the most careful non-designer does not know
what pitfalls they should be looking out for, because they don't have the
experience or wisdom of the designer they're trying to stand in for.

The typical approach taken here is to make the designers produce artboards,
prototypes, PDFs, PNGs, PSDs, After Effects mockups, basically whatever comes
out of your basic design toolkit. You give those specifications to someone who
knows how to work with UI in code, and they can then build something that tries
to resemble those specifications.

This is a lossy process. If you don't have a designer's mindset, and you're not
trained to identify intentional design choices and principles, you don't know
what parts of the design are flexible or open to interpretation, versus what
parts your designer will murder you for should you change it. Designers are
incredibly specific about things many people don't even think about, such as
the rhythm of negative space, optical sizing and corrections, typography and
iconography, grids, line lengths for paragraphs of text, and even intricate
details about responsive design and appropriate breakpoints. Without that
knowledge, and without the wisdom of the original designer's thought process to
guide you, *you are woefully underinformed to come to any decisions about what
resemblance is deemed adequate.*

So, the solution is to get a designer to evaluate your work and point out all
the things you can't see, right? Maybe even loop them in for some continuous
iteration and quick mistake resolution. Except...

## Getting a designer to evaluate resemblance is a waste of everyone's time

I say this as both a scripter and as a designer. As a scripter, I don't want to
spend time pushing pixels in ways I don't understand to solve problems I can't
see in interfaces I've already spent ages coding. As a designer, I don't want to
spend time telling coders how to re-create the thing I already spent ages
creating myself, and which already exists, but which needs to be transcribed
because for some reason I don't have the tools to do that design job properly
first time.

This is a fundamentally inefficient and frictionful workflow. At best, you end
up attempting to teach the design principles you use to everyone you work with,
to try and ease friction and reduce the number of iterations you have to do. At
worst, someone says 'fuck this shit' and breaks the iteration cycle early, where 
the design is left half-baked and crippled in some number of ways. 

Even worse, that outcome can sometimes be blamed on the designer 'being too
nitpicky'. Trust us, no designer enjoys making more work for themselves, just
the same as any other worker, especially not when most workers aren't even
invested in their jobs. We make our suggestions and points because *we're trying
to guide you towards making the design complete.* A good designer will tell
you that it's okay for some specifics to diverge, but no good designer will ever
tell you that it's okay to skip something principally important.

Fundamentally though, it's just better to have a designer come in and do the job
right first time. Designers are trained to know what's important and what can be
loosened. They know how to build the design in a way that balances function and
form in the right ways, and they know all of the hidden gem tricks that most
overlook. An original work will always be more consistent and more efficient
than a replica.

Okay, so the way we work now - with all of our fancy-pants frameworks like
Fusion and their myriad features - is causing collaboration problems. Does that
mean we should regress to the way things were *before* we complicated the
equation?

Well, no. To put it shortly, I didn't build Fusion because it was useless.

## Designers need modern innovations too

The wonderful conceptual world of components, templating, dynamic content and
responsive design aren't just brainchilds of nerdy programmers. More and more,
designers have been adopting these very same techniques to colloquially
direct their design work too.

The problem is that, as designers, we don't really have good tools to actually
*express* those ideas right now. Code has the luxury of being infinitely
flexible and introspective, allowing you to practically do almost anything
within a very permissive system of expression. On the other hand, unscripted UI
is still fixed in static, atomic building blocks, which are difficult to
compose, template, reuse or adapt to content. The modern day language of UI
design systems and responsive design have missed specialist designers completely.

> So what? As long as the coder can assemble those constructs from the
> designer's work, then the coder gets all the flexibility they need from those
> constructs.

... which is a very code-centric view of these organisational patterns. In
reality, the designers themselves stand to gain *stratospheric* productivity
from these ideas, with many parallels to the ways coders have done just that for
their workflows already. Right now, if you're not willing to adopt code, you're
falling back to the same basic workflow we've had since 2008, one which doesn't
reflect the reality of many UIs built today: you're working with the absolute
atoms of UI all of the time, with very few tools for organisation, even fewer
for adding variations and adjustable properties, and nearly nothing to help
you automate reuse or respond to dynamic circumstances such as intrinsic content
sizing, responsive breakpoints or different viewing conditions.

## Designers shouldn't have to learn code

Faced with all of this, some people are of the opinion that UI designers should
learn how to code UI in some capacity, too. There are a few problems with this
idea in my head.

Firstly is that 'coding UI' is *not* straightforward for someone who has never
written a line of code before. Unlike other places in computer science, where
you might at least get some nice friendly templating language, in Roblox you're
basically required to have a firm understanding of programming principles
before you're even ready to touch something like Fusion or Roact, because these
frameworks talk in the language of Lua's fundamental constructs. You *could* try
to build a templating language on top of that, but that seems excessive and will
probably annoy any coders working on the project. I certainly don't know of
anyone who would recommend that.

It's worth mentioning here that, as Fusion's designer, I've been trying to do my
part to intentionally keep Fusion's complexity down, unlike something like Roact
which almost presents itself like mathematics in comparison to regular scripting.
That doesn't change the fact that Fusion still assumes you know the ropes of Lua.
Perhaps better tooling to guide non-proficient users could help out here, though
I'm not precisely sure how you even begin to approach that, or why it'd even be
a good solution.

The second problem with this idea of 'designers who code' is that it's simply
not in a designer's job description to think about code and codebases. All that
a designer wants to do is *design*. It's indeed correct that code could help you
get to a design more efficiently, but it seems pretty excessive to tell someone
to learn an entirely new - *and entirely different* - skill, just to do what
they used to be able to naturally achieve. I think there must at least be some
happier mediums somewhere else, where re-education is not required to such an
extreme degree.

What a Gordian knot of constraints this is turning out to be.

## Is this fixable?

After all this writing, you're probably expecting me to posit that we're all
doing this wrong, and there's a better way to do this that will make your
designer happier. Oh, and it probably involves Fusion too, because I'm the
Fusion guy, right?

Unfortunately I come bearing no such gift. Instead, this is indeed a completely
unactionable whingepost which will contribute absolutely nothing practical to
your day-to-day workflow.

I think Roblox want packages to be the answer to this, but honestly I can't see
that properly addressing much of anything here anytime soon - packages have
historically been untenably inflexible, buggy to boot, and tie you into a very
specific way of working which I can't see catching on quickly with anyone that
doesn't conform exactly to Roblox's vision. Here's hoping I'm wrong on that.

I'm also personally thinking hard about these problems, too. Most people I talk
to think some of these ideas are fundamentally irreconcilable, and that it's not
ever going to be practical for designers and scripters to work 'without contact'
in the way I want to advocate for. That's totally okay - I'm willing to accept
if it's impossible! That's not going to stop me from treating this as a
deliciously difficult design problem though, and I've already been prototyping
and feeling out the space. Even if nobody can come up with an answer for these
issues, I feel like there are at least steps we can take to reduce the friction
of this process that we haven't yet invented, and fundamental principles at play
that we could harness to direct the co-working experience towards greater
productivity.

If the chance of improvement is possible, then it's already worth it to me.