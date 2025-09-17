---
layout: post
title: Why I'm writing a programming language
image:
  path: /assets/posts/why-im-writing-a-programming-language/thumb.jpg
  width: 256
  height: 256
---

I'm doing the thing people tell you not to do. Wolf is my small, smart programming language.

You'll likely start with a very valid question...

## Why?

I don't believe we're at the end of programming language history. We haven't found the perfect language yet (does that even exist?) so why not spend time exploring more of the space? Sure, I'm not an academic, and absolutely daft, but I figure there's still fun to be had in the process of exploring the possibility space.

In particular, there's a few ideas that excite me terribly:

### True consistency

Most programming languages are at least influenced by the syntax and semantics of C.

I think that C is a perfectly fine programming language! Basic perhaps, but somewhat fine. However, I think that most languages designed in C-style aren't really _consistently designed_ languages. Things that are ostensibly similar look different on the page, and things that are ostensibly different look similar on the page.

After a decade+ of working with the Lua language, I know how much I love a simple mental model. I wonder what that could look like when applied to a language's syntax.

I want a language where you can point to language features _visually_, by pattern matching with your eyes.  Instead of building towering complexes of features, you should be able to hold a model of the whole language in your head, and it should map 1:1 with each syntax item.

### Incremental recomputation

This is probably the headline thing I think about these days. After years working on reactive signal libraries, I'm desperately tired of all the extra syntax cruft.

I often think about immediate-mode programming. Conceptually, it's far and away the best choice for expressing logic clearly. The only drawback of it is that blowing away a bunch of computation just to do it all again can be incredibly wasteful, especially when you need to perform more complex layout reflow for example.

Why can't a compiler optimise that?

Of course, the answer defies simple explanation - side effects, mutation, blah blah. But that's just why we can't do it _in existing languages_. What would a language built for that look like?

Imagine you didn't just run through a whole function every time you needed to update the result of a previous calculation. What if you could just re-run the part that's actually relevant?

Right now, I fully believe that something like this is the bridge we need between immediate-mode and retained-mode logic.

### Memory management

After implementing scopes in Fusion, I'm pretty furious we ever spent so much time on garbage collection. Structured memory management is so obviously superior it's not even funny.

The problem is that doing structured memory management in Lua is also so obviously against the grain. Lua has no facilities for doing it ergonomically, because it's built on the philosophy that garbage collection will handle almost everything for you.

Simultaneously, I'm a pretty happy Rust user, but I'm not about to defend Rust's memory management approach either. Lifetimes are hell, and the language as a whole is confusing and unintuitive to learn if you're not familiar with its concepts.

When I listen to Graydon Hoare and Casey Muratori, I wonder just how many trivial memory management techniques we're missing when we think in terms of RAII and lifetimes.

Could you make a language that trivialises memory management by taking a broader view of the problem?

### Open objects

It seems really common these days to define objects as rigid nominal types with _only_ a defined set of data on them.

Perhaps you're making a 3D modelling program and you want to store some vertex data:

```rust
struct Mesh {
	vertices: Vec<Vertex>
}

struct Vertex {
	position: Vec3,
	normal: Vec3,
	uv: Vec2
}
```

And then perhaps, the tool has a vertex painting mode, where you want each vertex to _additionally_ have a vertex colour and alpha.

These things logically live on the vertex, so you _could_ put them there:

```rust
struct Vertex {
	position: Vec3,
	normal: Vec3,
	uv: Vec2,
	colour: Vec3,
	alpha: f32
}
```

But what if you wanted that vertex colour and alpha to come from an interchangeable "vertex colour map"? You'd have to store them externally:

```rust
struct VertexColourMap {
	vertices: Vec<VertexColour>
}

struct VertexColour {
	colour: Vec3,
	alpha: f32
}
```

I'm going to apply a somewhat-arbitrary transformation to this code. I'm going to switch the data around to be stored in SOA format, rather than AOS.

```rust
struct Mesh {
	positions: Vec<Vec3>
	normals: Vec<Vec3>,
	uvs: Vec<Vec2>
}

struct VertexColourMap {
	colours: Vec<Vec3>,
	alphas: Vec<f32>
}
```

Notice that, by performing this switch-around, we've actually flattened down one of the dimensions of this problem. No matter whether we store the data in `Mesh` or `VertexColourMap`, we do not need to change the data structure at all:

```rust
// The data is stored the same way (!)
struct Mesh {
	positions: Vec<Vec3>
	normals: Vec<Vec3>,
	uvs: Vec<Vec2>,
	colours: Vec<Vec3>,
	alphas: Vec<f32>
}
```

This led me to a _really cool_ insight - what if you could build objects by joining together fields like this? 

Instead of having a "mesh struct" with a rigid structure, what if an object could be formed from any fields you wanted to view together? In a system like that, you could easily define extra fields for an object in your own namespace, and transparently associate them with data that comes from elsewhere in the codebase. 

Instead of "closed objects" that can't be extended by others, you'd have "open objects" that are just syntax sugar for array accesses anywhere in memory.

I view this as the ultimate expression of the open-closed principle, or perhaps just the logical conclusion of entities from ECS. Objects become mere IDs - things you use to access fields. 

What would a language built around "open objects" look like? Could open objects be made efficient? Where would they fall short?

## Introducing Wolf

To pursue these questions, late last year I decided to pull the trigger and push the first commit to a repository called `wolf`.

![The initial commit for the Wolf repository.](/assets/posts/why-im-writing-a-programming-language/initial-commit.png)

Back then, I didn't intend to actually implement the language I was designing. All I wanted was to _explore_ what me and my good friend Trey thought "our ideal language" could look like.

In that time though, I've ruthlessly scrapped and rethought a ton of ideas. In the time that some people may have completed a whole traditional language, I've just barely figured out how the basic constructs of the language should even work, and even then I'm not entirely sure.

![The "Design" page on the Wolf website. It only covers basic language features](/assets/posts/why-im-writing-a-programming-language/i-havent-written-much.png)

That said, beyond the pages, there's a few truths I seem to have landed on in my head:

### Wolf works like maths

Wolf is designed along the grain of mathematics. Equals is equals - there's no assignment or mutation by default.

In that sense, Wolf is firmly placed in the camp of immutable functional languages built atop the lambda calculus, but not for any of the traditional reasons - I didn't even know what the lambda calculus was before I started.

I don't necessarily agree that mutation and assignment are fundamental operations that should appear in normal logic. I think most logic should be stateless, and when you're writing stateless logic, you _shouldn't_ need mutation in theory. (The lambda calculus backs this up!)

The real reason I think we reach for mutation is because functional syntax sucks. Recursion is not an ergonomic replacement for a `for` loop in _many_ cases. Why haven't we tried to close that gap?

So, to the maximal extent possible, Wolf should let you express stateless functions the same way you express maths - as substitutable expressions.

### Wolf mutates, but carefully

The last section may have given you the impression that I'm a Haskell enjoyer (more of an F# guy tbh), but I've omitted most of the story.

You see, I don't think mutation is something to be avoided. After all, our computers are _basically_ just overgrown Turing machines. To deny mutation in a programming language is to invent a new alternate reality.

Instead, I simply view things in reverse to most languages. In my head, mathematics is the base of logic, and mutation/imperative style is an _extension_ to basic logic that allows it to interact with an event-based model of the outside world.

From that angle, I think it is perfectly rational for Wolf to support mutable dynamic objects, because they're a reality of programming a Turing machine. The only key difference is that I think it should be tightly managed, so that both humans and machines can figure out how these dynamic objects and mutation events affect the execution of the program.

One particular mistake I'm not keen to re-enact is the idea of allowing unmanaged memory allocations and frees. I see no reason for there to be a "global allocator", and have no desire to implement things like borrow checking or RAII. 

Why are we trying so hard to make objects manage themselves internally, when we could just manage them better from the outside? We already know about concepts like arenas, object pools and slab allocators. These things trivialise memory management, but I don't blame people for not using them when many vogue languages don't elevate them to a prime position.

In Wolf, I want all dynamic features to route through structured memory management techniques like these, rather than random vibes-based resource allocations or garbage collectors. 

But importantly, it shouldn't feel like you're managing memory at all - it should be transparent, almost implicit in what you're writing. After all, it's maths, right?

### Wolf is ruthlessly smart

I think static analysis is non-negotiable in this day and age for good autocomplete and program optimisation. I don't want another dynamic scripting language.

So, Wolf is designed to be 100% statically analysed, type-checked and optimised. The language avoids features that would require dynamic or run-time analysis to check the correctness of, or which lead to optimisation footguns.

Instead of methods, Wolf prefers pipelining and freestanding functions. When you can pipe an object into a function, you don't need methods anymore. I already write most of my code using freestanding functions today in all programming languages I work in, and it makes modularity so much easier to achieve because it obeys the open-closed principle naturally.

Instead of polymorphism, Wolf prefers switching based on statically-available type information. This sort of thing should work a lot better in Wolf, because it's built without a lot of the concepts that make indirection necessary in other languages. We know that random memory access is especially expensive when factoring in things like cache misses, so why depend so heavily on it?

Instead of dynamic types, Wolf prefers inferred types. If you define a minimum amount of information about your interfaces, that the compiler should be able to propagate through your implementations. Inference should be a forwards process, since code is written forwards, and that unlocks the best possible autocomplete and error messages for unfinished code.

## Conclusions

Wolf is still extremely early in its life. It doesn't even have a reference implementation, or even a completed design. Mostly, it's just ideas, but I have some conviction that these ideas are good to pursue further.

Long-term, I want to bridge the gap between lightweight scripting languages and powerful systems languages - a kind of systems-language Lua that threads the needle between expressivity and performance.

I don't make any promises that this project goes anywhere, but if you see me making commits, then you know what I'm up to. Wish me luck on my misguided programming language adventures :)