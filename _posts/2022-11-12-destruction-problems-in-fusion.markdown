---
layout: post
title:  "Destruction problems in Fusion"
image:
  path: "/assets/posts/destruction-problems-in-fusion/thumb.jpg"
  width: 256
  height: 256
comments_url: "https://twitter.com/Elttob_/status/1591318536223608832"
---

Roblox made two frustrating API decisions that are actively hampering Fusion's
ability to provide a clean way of managing destruction.

1. Roblox APIs, such as instances and event connections, require manual
destruction.
2. Roblox does not provide a viable way to run code when an object is garbage
collected.

Luckily, [as many high-profile rants have highlighted](https://youtu.be/18n6Oh160YQ),
Fusion's finally gotten around to solving half of this problem. With the
introduction of destructors to Fusion 0.2, it's now possible for state objects
to specify how contained values should be destroyed, meaning instances and event
connections play nicely with state objects now.

The problem is that we only run that destruction code when values are being
*replaced*. When a state object falls out of scope and becomes completely
inaccessible, though, we still don't destroy the value inside the state object.
In fact, the entire object gets garbage collected with the live value still
inside. For obvious reasons, this is a soundness issue; if we can't guarantee
that your destructors will run, then the door is wide open for memory leaks.

Raw, unsandboxed Lua 5.1 actually *does* have a way of supporting exactly this.
If you apply a metatable with a `__gc` method inside, then that `__gc` method is
invoked when a table is garbage collected. This is the key ingredient that lets
you build completely garbage-collected APIs, even if they still fundamentally
need to talk to non-garbage-collected APIs.

However, for very good security reasons no doubt, Roblox's sandbox disables this
metamethod. Unfortunately though, it's *disablement without replacement*, the
absolute worst kind of disablement; quite literally the only way you can detect
an object has been cleaned up is by polling a weak table. So unless you want a
background process running all of the time, scanning your objects for liveness,
*we simply can't provide a garbage collected API at any level*, because Fusion
absolutely must support instances and event connections, and those things don't
garbage collect themselves. We need to run that destruction code manually.

To be fair, we could absolutely optimise that background process. If a state
object has been interacted with recently, we know it's been in scope recently,
so we don't need to check for liveness unless state objects get left alone for a
long time. We also don't have to poll frequently; based on how quickly state
objects are being created, we can easily scale up polling when lots of creation
or destruction is occuring, and reduce it down to perhaps zero polling for
static or idle interfaces. No matter how it's sliced though, polling sucks and
it is to be avoided where possible.

So, what else is there? We *could* just tell you to figure it out yourself,
essentially just lobbing a destroy method in every state object and providing no
extra API support at all. This would obviously change how you write components;
take this button component, for example:

```Lua
local function Button(props)
    local isHovering = Value(false)
    local isPressed = Value(false)

    return New "TextButton" {
        -- ... props ...
    }
end)
```

Perhaps the simplest pattern would be to store all your state objects in a
table, and pass that table to a cleanup key. You could do this today if you
wanted to:

```Lua
local function Button(props)
    local state = {
        isHovering = Value(false)
        isPressed = Value(false)
    }

    return New "TextButton" {
        -- ... props ...

        [Cleanup] = function()
            for _, object in state do
                object:destroy()
            end
        end
    }
end)
```

This has one fatal downside though; every single state object must be directly
inside of that `state` table. This means no more inline computeds in property
tables, and no more nesting state objects inside of each other. That's of
particular relevance for users of this common pattern of creating a computed
value for the sole purpose of animating it:

```Lua
local transparency = Spring(Computed(function()
    return if isHovering:get() then 0.5 else 1
end), 25)
```

Looking at these examples raises an interesting observation though, and brings
into focus why garbage collection really *was* the idiomatic, correct solution
here. The thing that matters in these examples is the *owner* of the state
object. 

- That nested computed object above is *owned* by the spring object. We want it
to be destroyed by the spring when we're done.
- The `isHovering` and `isPressed` values are *owned* by the `Button` component.
We want those to be destroyed when the component is destroyed.
- State objects created at a global level (e.g. theme values) are *unowned*, so
they should never be destroyed. They'll live for all the lifetime of the script.

To put it generally, we need some way to distinguish between obtaining a value
we *own* and must destroy, versus obtaining a value someone else owns and we
musn't touch. We don't want to destroy a global state object when we assign it
to our temporary button, but any inline computed objects *do* need to be
destroyed.

***Garbage collection is supposed to take care of that! That's literally its
entire value proposition - that you need not think about that at all!***

(also coincidentally, Rust solved this problem without a garbage collector by
using a borrow checker to allow for really awesome static memory management with
no garbage collector or complex destruction process at all, just scoping.)

Anyway, we need to get pragmatic and realistic. The chances of Roblox changing
this are close to zero right now, so we need to build something ourselves.
Whatever solution we come up with needs to represent ownership of *values* in
some way. [Some work I did on 'managed types' a while back might be relevant
here.](https://youtu.be/atQv2N9eFps)

The nice thing is, if we find a solution for the above, it by definition should
also be applicable to the prior issues with instances inside of state objects,
so we're virtually guaranteed a one-size-fits-all solution that could replace
our current destructor callbacks on state objects, *if* we find a solution.