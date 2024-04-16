---
layout: post
title: Monotonic painting
image:
  path: /assets/posts/monotonic-painting/thumb.jpg
  width: 256
  height: 256
---

Introducing the **monotonic painting** algorithm, a lazy evaluation algorithm that can skip evaluating updates originating from meaningless changes.

## Context

I maintain my own reactive signal library for Luau, called [Fusion](https://elttob.uk/go/fusion/).

Fusion has historically used an *eager* execution model. When you change something in Fusion, it immediately finds all affected objects and updates them. Specifically, it uses an algorithm I derived, called [SETS](https://elttob.uk/go/sets/). I've discovered recently that it has striking similarities to MobX's internal algorithm.

However, it's not good enough, and there's a few fundamental reasons why. 

### Wasted work
Firstly, you can end up doing a lot of unnecessary work because you're trying to keep up to date all of the time, even when the results are never used. That's the primary reason people don't use eager algorithms.

However, I'm not super concerned about this, and I won't talk much about it.

### Update order
Eager algorithms are burdened with the difficult task of ensuring you never see stale dependencies in your reactive code, with only explicitly encoded knowledge of which dependencies you're going to try and access. It turns out that this is not simple to do, and even when you employ techniques such as topological sort/search, it's still possible that there was some knowledge that you didn't encode in your reactive graph about the order in which things will be accessed. 

Eager updates work well when the only thing you're concerned about is accesses happening between reactive graph objects, but the moment you have to be concerned about the wider world, you either need to pull more things into your graph or accept that things will break. In case any of this sounds hypothetical, many of Fusion's most difficult real-world problems come from exactly this kind of disconnect.

That's the real reason I'm now reaching for a *lazy* execution model. Lazy execution does computations "just in time" when you go to use your dependencies. Because the computation has to occur before the result can be read out (that's how functions work), it's guaranteed that your dependencies will have been computed by the time you read the result, and as such, all of the complex considerations around update order are pretty much solved as a property of the system.

### Meaningful changes
Fusion also has a notion of 'meaningful changes'. If you have a computation that outputs the number 2, and it re-runs and still produces the number 2, then you don't want to update anything that depended on that computation, because they'll just be given the same number 2 as input. 

> Editor's note: Reactively calls this the 'equality problem', but I prefer to talk about 'meaningful changes'. For some values, equality doesn't always align well with our intention: `NaN != NaN`, but the two are not meaningfully different. For other values, it's expensive to compute equality: a mutable table may be equal to itself referentially, but it could still have mutated contents, so instead of doing an expensive structural equality check, you probably want to substitute in some other indicator.

When you're doing *eager* execution, it's really easy to stop updating further objects when you encounter a meaningless change. Just... don't propagate the change beyond that object. You're calculating the changes as you go anyway, so it's not like you have zero access to the information during the update process...

...unlike *lazy* execution, which indeed has zero access to the information during the update process.

When a change occurs in a lazy system, you *have* to mark all of the transitive dependencies, because they might have changed. But you don't know if they're *meaningful* changes, because you need to run the computation to unlock that information, and if you run the computation, then this isn't a lazy process anymore. Instead, you'll have to do extra work when you're doing the calculation, at a completely different point in time, and possibly in a different order.

-----

## Introducing monotonic painting

That's where **monotonic painting** comes in. I developed this algorithm independently, but it bears some resemblance to [other reactive algorithms](https://github.com/modderme123/reactively/blob/main/Reactive-algorithms.md).

Monotonic painting combines graph-painting-based lazy evaluation algorithms with a monotonically-increasing number (which I call `time`) to measure the order that changes occur. This allows monotonic painting to efficiently skip over meaningless computation work. Monotonic painting can also be trivially extended to detect cyclic dependencies - even those not previously encoded in the reactive graph - and halt the algorithm with an error.

To understand the monotonic painting algorithm, we will start with an incredibly simple algorithm, and build on top of it with extra features.

### Level 1: basic painting

The first algorithm we will look at is a dead simple painting algorithm. Consider a tree of objects, where every object can be painted one of two colours; either `valid` or `invalid`.

<figure>
    <img src="/assets/posts/monotonic-painting/level-1-intro.svg" alt="A directed acyclic graph of objects painted 'valid'">
</figure>

An object broadcasts an update which invalidates its value. This paints the object and all of its direct and transitive dependents as `invalid`.

<figure>
    <img src="/assets/posts/monotonic-painting/level-1-invalidate.svg" alt="An object in the middle is invalidated, painting itself and all dependents as 'invalid'">
</figure>

The user asks for the value of an invalid object. This object will now need to run its computation.

<figure>
    <img src="/assets/posts/monotonic-painting/level-1-peek.svg" alt="An invalid object at the bottom is queried.">
</figure>

The dependencies of the object are cleared.

<figure>
    <img src="/assets/posts/monotonic-painting/level-1-clear.svg" alt="The bottom object clears its dependencies.">
</figure>

During the calculation, the object can choose which other objects it now wants to depend upon by looking at their values in turn - essentially, a recursive call.

<figure>
    <img src="/assets/posts/monotonic-painting/level-1-dependency.svg" alt="The bottom object forms a connection with the object above. The above object is queried as the bottom object was.">
</figure>

This recursion continues all the way up until a computation completes without looking at an invalid object's value.

<figure>
    <img src="/assets/posts/monotonic-painting/level-1-recurse.svg" alt="Objects recursively query their dependencies up until the last invalid object.">
</figure>

When that computation completes, the object is painted as `valid` and returns the result of its computation.

<figure>
    <img src="/assets/posts/monotonic-painting/level-1-validate.svg" alt="The last invalid object validates itself and returns.">
</figure>

This cascades back through the chain of recursions all the way until the original object returns from its computation. The queried object is valid again, and can return its value.

<figure>
    <img src="/assets/posts/monotonic-painting/level-1-complete.svg" alt="The recursively accessed objects validate and return, and the original queried object can validate and return.">
</figure>

This simple algorithm has many desirable characteristics, such as naturally performing computations in the right order to prevent glitches, and not computing values that are unrelated to the result that's desired.

### Level 2: catching cycles

This simple algorithm can now be extended to catch cyclic dependencies. We now have three colours: `valid`, `invalid`, and `busy`.

Invalidation looks the same, so this explanation will start after an object has been invalidated in the middle, in the same situation as before.

<figure>
    <img src="/assets/posts/monotonic-painting/level-2-intro.svg" alt="The graph looks as it did originally, except an object in the middle was invalidated at some point and this has cascaded down.">
</figure>

Again, an invalid object is queried for its value, and it clears out its dependencies and runs its computation. However, it paints itself `busy` while this is happening.

<figure>
    <img src="/assets/posts/monotonic-painting/level-2-peek.svg" alt="An object at the bottom of the graph is queried. It has no dependencies and is painted as busy.">
</figure>


It decides to depend on (and query) a nearby object.

<figure>
    <img src="/assets/posts/monotonic-painting/level-2-dependency.svg" alt="An object at the bottom of the graph is queried, and has one dependency on an object above it, which is now being queried.">
</figure>

However, this nearby object decides to depend on (any query) the original object again. This creates a cycle.

<figure>
    <img src="/assets/posts/monotonic-painting/level-2-cycle.svg" alt="The dependency has queried and formed a dependency upon the original object.">
</figure>

This is detectable because the query occurred on an object that was already `busy`. You can output an error here and reject the dependency.

For completeness, you can also reject invalidations which discover a `busy` painted object, though your implementation doesn't break if you don't reject them. The advantage of rejecting busy objects on invalidation is that you can reject more subtle cycles that aren't encoded in the reactive graph directly, but generally it doesn't make too much difference, and you can live without it.

### Level 3: monotonic painting

With this background, we can now introduce the 'monotonic' element. All that you need for this is a global number that increases every single time an object meaningfully changes. This monotonically increasing number is hereafter referred to as `time`, because it can be used to compare which changes occurred earlier or later than other changes.

Each object in the graph will store the `time` when it last meaningfully changed, so in a freshly recomputed graph, dependencies will always have a smaller `time` than dependents. This fact will become important later.

<figure>
    <img src="/assets/posts/monotonic-painting/level-3-intro.svg" alt="The original graph, except each dependent object has a greater 'time' associated with it.">
</figure>

>  Editor's note: For the rest of the explanation, the `time` will only be shown for changes that occur during the explanation, to keep the visuals clear. 

>  If it helps, imagine the older `time` values as being too small to draw, because they're significantly older.

To begin with, an object is invalidated, as before. However, there's a subtle difference here. Because we need to know if this was a meaningful change or not, the targeted object has to be computed. 

If the computation is meaningful, the dependents are painted `invalid` and the targeted object gets assigned the `time` that it was meaningfully changed. If the computation is meaningless, then of course nothing happens.

This is fine, because pretty much all of the time, the computations here are trivially simple.

<figure>
    <img src="/assets/posts/monotonic-painting/level-3-invalidate.svg" alt="An object in the middle transitively invalidates itself and its dependents. The original object is assigned a new time.">
</figure>

When an invalid object is queried, the object is painted as "busy", but does *not* immediately clear its dependencies. 

<figure>
    <img src="/assets/posts/monotonic-painting/level-3-peek.svg" alt="An invalid object at the bottom of the graph is queried. It is painted busy.">
</figure>

Before any dependencies are touched or computation is done, the object must first prove that one of its direct dependencies has been meaningfully changed.

To do this, remember the earlier principle. If this object's `time` is larger than its direct dependencies' `time`s, then no meaningful change took place and no work needs to be done. Inversely, if a direct dependency is found with a larger `time`, then some meaningful change has took place and the computation must proceed.

So, each dependency is visited in turn. 

This object's first dependency is valid (remember this principle only holds for valid objects) so its `time` is checked. It's a smaller `time`, so we move on to the next one.

<figure>
    <img src="/assets/posts/monotonic-painting/level-3-older.svg" alt="The first dependency of the object is valid and older.">
</figure>

This object's next dependency is invalid. Invalid objects don't fit our previous principle, because their `time` might not have been updated yet.

<figure>
    <img src="/assets/posts/monotonic-painting/level-3-unsure.svg" alt="The next dependency of the object is invalid, so the time isn't accurate.">
</figure>

So, the dependency must have its further dependencies visited.

<figure>
    <img src="/assets/posts/monotonic-painting/level-3-dependency.svg" alt="The dependency is queried and marked as busy.">
</figure>

The dependency itself only has one dependency - the object that was invalidated earlier. 

This object will have a newer `time`, which means the dependency - and by extension - the original queried object - has experienced a meaningful change and will have to recompute.

We don't have to recompute the dependency right now, because the original object might not choose to depend on this dependency anymore when it recomputes. However, we *are* now aware that it will need to be recomputed for sure if it *is* queried. You should temporarily cache this information so you can skip re-doing these checks in case this dependency is queried during the computation.

Anyway, the original object has the go-ahead to recompute, so it clears its dependencies and runs its computation. It decides to depend on a nearby object, and the usual recursion occurs.

<figure>
    <img src="/assets/posts/monotonic-painting/level-3-recurse.svg" alt="A chain of dependencies are queried originating from the bottom object.">
</figure>

And, as the computations complete, each object is painted as valid. If the change was meaningful, it's also given that monotonically increasing `time`.

If all of the changes are meaningful, you get the familiar picture:

<figure>
    <img src="/assets/posts/monotonic-painting/level-3-complete.svg" alt="The dependencies and bottom object are painted valid, with increasing times.">
</figure>

But if all of the dependencies didn't meaningfully change, you get this wonderful massive cascade of objects skipping their computations and becoming valid very cheaply.

<figure>
    <img src="/assets/posts/monotonic-painting/level-3-skip.svg" alt="Everything is painted as valid because no dependencies had meaningfully changed.">
</figure>

This is what makes monotonic painting awesome - it has all of the properties of the previous algorithm that Fusion used, but it tackles the problem completely differently - and arguably, more elegantly, given that no topological sorting had to be done at all.

-----

## Conclusion

Going forward, I plan to use monotonic painting as the update algorithm for Fusion. I'm working on it right now, actually, and writing this blog post helped me prove the details to myself in my head. I even found ways of making the algorithm more efficient.

If you're wondering how to implement `time` by the way, there's two options:

- You can measure the time since an epoch, such as Luau's `os.clock()`
- Alternatively, if you're fine with global state, you can maintain a counter. Just increment the counter every time you assign a `time`.

Fusion uses the first method so as to avoid the need to synchronise a counter between multiple copies of the library.

Anyway, I'm looking forward to getting this rolled out to Fusion's main branch. I already had a research version up and running in `tokamak` (my toy reactive library for research) since last November, but it's going to be incredible to see it supporting some truly huge projects.

I love that kind of thing.