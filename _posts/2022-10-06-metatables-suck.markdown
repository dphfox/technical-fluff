---
layout: post
title:  "Metatables suck."
thumbnail: "/assets/posts/metatables-suck/thumb.jpg"
excerpt: "You might observe that I try pretty hard to avoid metatables in my own code. Here's every use case I've seen for metatables and why they all suck."
---

*I originally posted this on Medium, before Technical Fluff was a thing. I'm
crossposting it here for completeness.*

You might observe that I try pretty hard to avoid metatables in my own code.
Here's every use case I've seen for metatables and why they all suck.

## Don't redefine well-defined operations

One of the biggest draws of metatables is their ability to replace core language
features with custom implementations. In total, you can provide implementations
(or sometimes values) for:

- Retrieving values of keys that don't exist
- Writing values to keys
- Calling the table like a function
- Using basic math and comparison operators on the table
- Converting a table to a string
- Retrieving the metatable of the table
- Table finalisation/destruction [^1]
- Measuring the length of tables
- Using a table as an iterator

This allows you to 'rewrite the rules' of the language. You can completely
redefine operations that are otherwise well-defined, and you can even go as far
as to introduce far-reaching side effects and performance penalties in far-away
idiomatic code. I do not consider this to be a good thing.

Code maintainers have expectations about operations and language features. They
have expectations about their performance overhead, their memory usage
characteristics, and their side effects. One of my goals is to turn those
commonly held expectations into guarantees - when you're trying to assess where
memory leaks or excessive CPU usage are originating from, you should never have
to doubt the core features of the language. Metatables can, and often do, break
this.

Indexing a table rapidly should never be unduly slow, nor should it bloat the
memory space. Adding two things together should never accidentally connect to a
database in Melbourne. The problems in your code's logic should always be in
what is explicitly written, not hidden by the building blocks of your code,
because that reduces the surface area for bug hunting and allows you to make
proper rigorous assertations about what a passage of code will do and how it
should perform while doing so.

Correct, idiomatic code should never be made incorrect or have atypical
performance or memory characteristics due to non-local code. It should always
behave the way that it appears, regardless of context. The only way to guarantee
that is to make sure that code never deals with metatables that flout the rules
of the language.

Indeed, you often find these arguments presented as refutations of operator
overloading in other programming languages. It's simply too easy to introduce
subtle bugs and dig yourself into nightmarish holes when this kind of usage is
encouraged, and this has been evidenced in practice time and time again.

However, that is not to say that all uses of operator overloading are
necessarily doomed to fail - when there is a widely-agreed-upon consensus for
how two data types can be operated upon, and when that operation lines up
naturally with Lua's own operations, then it perhaps makes sense. This is of
particular relevance to mathematical packages, which often need to provide
operations for working with vector, matrix, complex, infinite-precision or
geometric data types. This is clearly something worth preserving, because it
makes sense.

Overall, though, metatables seem to over-provision language-altering features,
and culturally we do not have the right mindset when working with them. We
should treat operator overloading as something to complement the operations
we already have, not to redefine them.

Don't redefine well-defined operations. Use plain functions for those.

## Inheritance is internally inconsistent

One such egregious redefinition happens when using metatables to inherit pairs
from a secondary table, essentially providing fallback values when a value is
not provided by the primary table.

To the user of such tables, this brings into existence a kind of 'phantom value' -
values which you can index by key, but which don't appear in any other kind of
operation. They don't contribute to the length of a table, nor can you iterate
over them, nor can you erase them, because what appears as a single table to
your code is actually just the primary half of a table split into two parts.
This breaks the mental model of tables as simple key-value pair collections.

That is to say, inheritance unreasonably blurs the line between the primary and
secondary tables. Most operations directly deal with only the primary table -
you measure the length of the primary table, you write values into the primary
table, you cast the primary table into a string, et cetera. However,
inexplicably, the indexing operation itself is replaced with a kind of
'fallback routine' which merges both tables into one, eliminating the
possibility of telling the two tables apart with normal practices. If you want
to index the primary table only, you're forced to resort to specialised in-built
'raw' functions to bypass the metatable. This is exactly backwards to the way
the rest of the language is set up - indexing should operate on the primary
table only, and we should be using functions to read from the table with
fallback values. That keeps internal consistency and makes it explicit when
you need special case behaviour. More importantly, it means that indexing
behaves consistently and predictably in all cases.

Furthermore, inheritance invites non-local consequences into your codebase. You
can privately hold a table with perfect control of all access points, and yet if
it inherits from an external table, you can no longer make strong guarantees
about how it behaves. In that way, inheritance breaks encapsulation and
introduces coupling, as external code modifying external data can break
privately held code without warning or even the slightest indication of
something going wrong.

As with class inheritance, data inheritance is simply not a good idea. Where
external data needs to be incorporated, it is better to store a plain reference
to it and to make the dependency explicit and visible. No 'phantom values', no
secondary tables, no encapsulation breaking.

Inheritance is internally inconsistent and an over-complication of tables.

## OOP puts the cart before the horse

The only problem with dismissing inheritance is that Roblox's most popular OOP
pattern is built on it.

The One True Pattern involves creating a class in two parts. Firstly, you create
a common table which holds the methods of the object as functions. Then, to
create a new object, you create a table to hold the object's data, and make it
inherit the functions from the common table:

```lua
local class = {}

function class:sayMyName()
    print("My name is", self.name)
end

local me = setmetatable({
    name = "Elttob"
}, {
    __index = class
})
```

This is, of course, over-engineered. You don't need metatables to do OOP at all.
Just store your functions in your objects:

```lua
local function sayMyName(self)
    print("My name is", self.name)
end

local me = {
    name = "Elttob",
    sayMyName = sayMyName
}

me:sayMyName()
```

This is functionally identical to metatable-based OOP in every single way you
care about. [^2] Beyond that, since it behaves nicely with other language
features too, the possibility space for what you can do with your objects opens
up further. For example, you can now easily clone these objects, methods
included, without having to remember to re-assign a metatable every time. *They
just work, no matter what you're doing with them, because they're simple.*

The only thing you have to do is assign the functions yourself, which some
people might find tedious. In fact I'm pretty sure most people use metatables
just because they're too lazy to write out some assignments.

Luckily, there's a way out. To save your poor fingers from withering up after
writing one line of code, you can combine the function declaration and
assignment into one:

```lua
local me = {
    name = "Elttob"
}

function me:sayMyName(self)
    print("My name is", self.name)
end

me:sayMyName()
```

This is the approach for OOP that I would *want* to advocate for. It's simple,
clean, and does exactly what it says on the tin - no weird metatable shenanigans
to trip up novice developers. In fact, I would go as far as to say that *this
should have been the idiomatic way to do OOP.*

Before we get carried away, though, this third approach is *not* as performant
as the previous approach. Previously, we stored a reference to an existing
function inside the object. However, in this approach, we're constructing the
function alongside the object. If you care about the micro-performance of your
code, that means you'd technically be taking up more memory and spending more
time initialising functions.

That's where we derive the common justification people give for not using these
simpler, metatable-free variants of OOP.

> The engine doesn't optimise them as well as the metatable based objects!

This is where the cart has been put before the horse. Part of the job of Luau
maintainers is specifically to optimise the coding patterns we depend on every
day. If Luau's code analysis can determine that we're constructing objects with
methods that don't change, then why shouldn't it be able to de-duplicate them in
memory? We're already using code analysis to improve autocomplete and linting -
the tools to fix the problem are right there!

In my opinion, we should ideally be building our engines around the coding
patterns that make the most sense and are most consistent with the way idiomatic
code works. The hyperfocus around metatable-based OOP does not line up with this
school of thought. We should instead be hyperfocusing on making OOP more
practical without resorting to operation-altering, guarantee-breaking metatable
magic.

So yes, those people are right. It's marginally slower. But you shouldn't have
to care, and in fact for most users of OOP, they don't have to care. That's not
going to be anything close to their true bottleneck. Just use the third approach,
and don't worry about the marginal cost.

However, if for whatever reason, the primary resource hog in your game is
constructing crap-tons of objects, you still don't need metatables. Use the
second approach, and you skip the construction costs, at the expense of having
to do one single assignment. Sorry for the carpal tunnel.

Fundamentally, though, Luau just needs to put the horse before the cart.

## Scopes are the encapsulation primitive

Objects bring us nicely on to the topic of encapsulation. Oftentimes we want our
objects to be associated with privately held data.

The simplest (and weakest) way to hold private data is to just make it public
data, but prefix it with an underscore so people realise it's not supposed to be
used by them:

```lua
local me = {
    name = "Elttob",
    _secretDesire = "eat pizza"
}

function me:spillDarkSecret()
    print("I really want to", self._secretDesire)
end

me:spillDarkSecret()
me._secretDesire = "do maths" -- booooo 👎🏻
```

Now, I know what you're thinking. Public private data is a bad joke. Let's hide
the private data. If you're metatable-brained, you might be inclined to make a
'proxy object' that shows you a controlled view of the object underneath, and
prevents you from bypassing that view.

```lua
local publicMe = setmetatable({}, {
    __index = function(_, key)
        if string.sub(key, 1, 1) == "_" then
            return nil
        end
        return me[key]
    end,
    __newindex = function(_, key, newValue)
        if string.sub(key, 1, 1) == "_" then
            error("Can't write private properties")
        end
        me[key] = newValue
    end,
    __metatable = "This metatable is locked."
})

publicMe:spillDarkSecret()

publicMe._secretDesire = "do maths" -- errors! ✅
```

This is gross though. By using a proxy object, we're now elaborately lying to
the rest of our code. We're running way more code than we need to for regular
accesses to methods, and generally everything is overengineered to a significant
degree. Even for advocates of metatable-based OOP, this is understandably too
much to swallow, and you generally don't see people doing this in professional
codebases.

The simpler solution, naturally, is to not mix up the public and private data.
Store the private data externally, in some other variable that the rest of the
code can't access. Scopes were quite literally made for this:

```lua
local me = {
    name = "Elttob"
}

do
    local secretDesire = "eat pizza"
    function me:spillDarkSecret()
        print("I really want to", secretDesire)
    end
end

me:spillDarkSecret()
```

Beautiful, isn't it? No lies, just elegantly encapsulating the data. This
approach is highly compatible with the previously proposed third way of doing
OOP, since constructors are the perfect place to privately scope data:

```lua
local function new()
    local self = {
        name = "Elttob"
    }

    local secretDesire = "eat pizza"

    function self:spillDarkSecret()
        print("I really want to", secretDesire)
    end

    return self
end
```

You can even completely forgo storing any data inside the table at all, instead
just using the table to store the functions, and keeping all of the data
perfectly private.

If you're not writing your constructors like that (for example, if you're using
the second way of doing OOP), then you can instead store private data in a table
and index it with the public data:

```lua
local privateObjs = {}

function spillDarkSecret(self)
    local private = privateObjs[self]
    print("I really want to", private.secretDesire)
end

local me = {
    name = "Elttob",
    spillDarkSecret = spillDarkSecret
}

privateObjs[me] = {
    secretDesire = "eat pizza"
}

me:spillDarkSecret()
```

As long as the table of private data is kept encapsulated, the rest of the code
is secured against accessing any of the private data.

While it's definitely interesting to see that encapsulation is easily done
without metatables, we should keep in mind that objects with data stored outside
of itself aren't cloneable or serialisable by external code. If that's important
to you, then you need to keep all of your object's data publicly accessible.
Otherwise, just be aware that you're losing the ability to do those things
without bugs.

Overall, though, scopes are the simplest, cleanest encapsulation primitive.

## Garbage collection is incomplete

This is the one that not many people talk about. It turns out that, at least for
Roblox Luau, there's a key garbage collection feature missing that makes any
custom garbage collection concepts pretty much useless - destructors.

Imagine you want to gain exclusive control over a data store for some amount of
time while you do some writing operations, then free up control when you're done.
Since Lua is designed around a garbage collector, you could try to hook into the
garbage collector for your data store API. When you create an instance of the
API, you open the connection, and when that instance is garbage collected, you
close the connection and free up control.

This would work fantastically in base Lua 5.1, but there's a serious roadblock
preventing us from doing it in Roblox. The metamethod that gets called on
destruction is disabled, and there's no replacement API. Roblox cites security
reasons to do with how they assign permission levels to threads, but you don't
need to understand what that means.

The entire point of garbage collection is to run destructors automatically.
Without that critical piece of the puzzle, we regress to the default: manual
memory freeing. That's right! If you're writing Roblox scripts, you're doing
manual memory management. Don't believe me? Try removing every call to destroy
methods in your game and see how fast you run out of memory.

In the context of manual memory management, you always have to call some
procedure somewhere to destroy your objects. That basically makes all of Lua's
garbage collection features useless for most people. After all, why resolve a
circular reference with weak tables when you could just set the reference to
nil in the destroy method and keep your sanity?

The only time you ever need to work with weak tables in Roblox is when you don't
need to destruct anything, and even then, that's pretty tenuous. There's no
guarantee that future versions of your API won't need to destruct anything, so
you can easily end up in a pickle if you build something around garbage
collection, and later need to run some code on destruction. Fusion is currently
running into exactly these growing pains, and it hurts.

Luckily, things aren't quite as dire as they are in, say, C. You're not going to
get a segfault when you read from an instance after you've destroyed it, and at
least one or two fundamental things from Lua still garbage-collect nicely. You
can still run into a few mismanagement issues pretty easily if you're not
careful though, which makes it a dreadful shame that Roblox doesn't help you
defend against that using the garbage collector that's right there.

Garbage collection on Roblox is incomplete. It sucks. Don't depend on it.

## Closing thoughts

You may not be entirely convinced that metatables suck yet. Perhaps you think
they're suboptimal, but it's not that bad, right? "My code uses metatable OOP,
and I think it's fine."

Perhaps it's somewhat an overreaction to abandon metatables entirely. I'm still
in two minds about it myself. In either case, I generally don't find myself
reaching for them anymore as a first solution - there are just too many good
alternatives to metatable solutions that are simpler, more readable and easier
to get novice developers onboard with.

In particular, I really hope that the Luau team get closure-based OOP to be
competitive performance-wise with metatable-based OOP. As small as the overhead
is, I do think that it's unfortunate it's been neglected for so long, and I
really do truly believe it's the proper, idiomatic, simplest way to do objects.
I suspect some not-so-recent changes to function identity in Luau might be
related to this somewhat.

And even if I've not convinced you at all, hopefully it at least gets you to
think about metatables a bit more critically.

[^1]: Roblox's Luau implementation disables destructors.
[^2]: There is a tiny technical difference, but it's extreme micro-optimisation territory. With metatable-based OOP, you're storing a reference to a table of function references, but here you're storing the function references directly. That does mean that you'll technically be allocating a couple bytes per function reference in the table, but if that's a problem then your name is probably James and you're making everything in your game an Rx observable.