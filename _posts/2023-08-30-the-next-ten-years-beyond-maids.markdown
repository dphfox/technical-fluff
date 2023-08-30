---
layout: post
title:  "The next 10 years, beyond Maids"
image:
  path: "/assets/posts/the-next-ten-years-beyond-maids/thumb.jpg"
  width: 256
  height: 256
---

Maids have been the dominant destruction pattern in Lua(u) for a decade - but
new possibilities await beyond them.

## Understanding the context

The original motivation for Maids is what I will call 'the destruction problem'.
Generally, we are working with two kinds of value in Luau:

- garbage collected values, such as numbers, bools, etc.
- manually managed values, such as Instances, OOP objects, etc.

Garbage collected values aren't too much hassle to work with, as long as you're
careful not to accidentally hold on to references forever, e.g. via event
callbacks. These naturally flow with Luau's grain and work well.

Manually managed values, however, are a notorious pain in the ass. Parallels
can easily be drawn between these values and good old `malloc()` and `free()`.
Every object creation must be paired with exactly one object destruction. As it
turns out, *developers kind of suck at this.*

It's easy enough when you only have a few manually managed values. When only a
few values are in play, it's easy to remember and easy to audit, because it all
fits roughly in working memory. The problem comes when you're wading in a small
forest of objects, all of which *have* to be handled correctly. As the number of
objects goes up, your capability to catch destruction errors goes down. It's all
too easy for a `:destroy()` to be forgotton somewhere.

This is the destruction problem. How can we ensure objects are destroyed when we
are done with them?

## See the forest, not the trees

Let's recognise a powerful insight. Broadly, lots of values are created and
destroyed alongside each other. Therefore, it makes sense to *consolidate* them.
We should be able to group the values into a larger structure that can be
destroyed all at once.

This is what Maids do. When you pass a value into a Maid, you are adding that
value into a 'cleanup table' that keeps track of everything that's been made so
far. When the Maid is destroyed, it runs through that list and destroys
everything inside. This effectively consolidates a bunch of small values into
one larger value; you have reduced the number of things you need to track.

```lua
local function Button(props: Props)
    local scope = Maid()

    maid:destroy()
    
    local animColoured = scope:give(Spring(scope:give(Computed(function(use)
        return if use(props.Coloured) then 1 else 0
    end)), 50))
    
    local isHovering = scope:give(Value(false))
    local isPressed = scope:give(Value(false))
    
    local textColour = scope:give(Computed(function(use)
        local isDark = isColourDark(use(props.Colour)) 
        local atopColour = if isDark then use(Theme.textAtopDark) else use(Theme.textAtopLight)
        return use(Theme.text):Lerp(atopColour, use(animColoured))
    end))

    -- ... snip ...

    scope:destroy()
end
```

This is such a good idea that it has thrust Maids into almost every top game's
codebase, and for a long time has been a staple of developer toolkits. To be
clear, it doesn't *completely* solve the destruction problem - it's effectively
kicking the can down the road - but it reduces the number of moving parts,
making the problem more tractable.

## Growing pains

All this being said, I think there are some notable unsolved problems around
Maids. They have a tendency to bloat line length with extra method calls and
pairs of parentheses, which typically means code using Maids is harder to
understand.

Maids are also fundamentally an opt-in paradigm. If you write code without using
Maids, it will look completely normal and unassuming. It won't error when you
try and run it. The only indication that something may be wrong is that your
memory usage slowly creeps up over time. All of this is to say: the API design
of Maids does nothing to steer you towards using them, and away from writing
dangerous code in their absence.

This feeds into another problem with Maids. It is very difficult - perhaps even
impossible - to comprehensively statically check that they're being used for all
of your objects. By being opt-in and light-touch, Maids don't have a clear story
for integrating with the increasingly useful static analysis tools Luau
provides. Sure, you can type-check that you're putting the right kind of things
into them, but can you check that you're putting things into them in the first
place?

Maids can't answer these problems. They weren't designed to do that.

We can.

## Make invalid usage unrepresentable

The way to solve this problem is to stop thinking so much about how you're
collecting objects together. Instead, you have to flip the problem on its head,
and think about where all that garbage is being generated - in constructors.

As long as constructors don't have memory management rules baked into them, you
will always be able to get around whatever external memory management techniques
you're using, and you'll never be able to properly enforce them through static
analysis. This is how we will move beyond what Maids could ever hope to do.

Let's assert that objects *must* be added to a cleanup table somewhere. It is
simply invalid to create an object without doing that. 9 times out of 10, this
is going to be what you want, and for those few occasions where it isn't, it's
easy to eat the cost of creating a list just for one object.

Now, take a hint from Rust - make invalid usage unrepresentable through the type
system and the API surfaces you expose. For us, that means that constructors
must always accept a cleanup table, non-optionally. Whatever they construct,
they must add into that table.

```lua
local function Button(props: Props)
    local scope = {}
    
    local animColoured = Spring(scope, Computed(scope, function(use)
        return if use(props.Coloured) then 1 else 0
    end), 50)
    
    local isHovering = Value(scope, false)
    local isPressed = Value(scope, false)
    
    local textColour = Computed(scope, function(use)
        local isDark = isColourDark(use(props.Colour)) 
        local atopColour = if isDark then use(Theme.textAtopDark) else use(Theme.textAtopLight)
        return use(Theme.text):Lerp(atopColour, use(animColoured))
    end)

    -- ... snip ...

    doCleanup(scope)
end
```

This gives us something that we can annotate with Luau types; as a result, we
can now statically analyse our code to check that we are adding our objects to
these tables for destruction. **If your code passes type checking, you can now
strongly guarantee that all of your objects are using cleanup tables.** There is
simply no other way around it.

Of course, checking whether those tables are ever destroyed *themselves* is a
different question. Remember - consolidation only kicks the can down the road,
but it's a damn good kick.

## Finishing touches

At this point, you've got a pretty good system going. It's brilliantly simple to
use, and you don't need OOP fluff just to get clean syntax for adding stuff
to your cleanup tables. It happens automatically with very little line bloat,
and it's statically analysable no matter what odd situation you're stuck in.

If you want to get a truly deluxe developer experience though, you can go one
step further and think of constructors as `associated methods operating on
cleanup tables`. That is to say, constructors don't live *outside* of cleanup
tables, but are the API surface *of* the cleanup table.

```lua
local function Button(props: Props)
    local scope = scoped(Fusion)
    
    local animColoured = scope:Spring(scope:Computed(function(use)
        return if use(props.Coloured) then 1 else 0
    end), 50)
    
    local isHovering = scope:Value(false)
    local isPressed = scope:Value(false)
    
    local textColour = scope:Computed(function(use)
        local isDark = isColourDark(use(props.Colour)) 
        local atopColour = if isDark then use(Theme.textAtopDark) else use(Theme.textAtopLight)
        return use(Theme.text):Lerp(atopColour, use(animColoured))
    end)

    -- ... snip ...

    doCleanup(scope)
end
```

This is as simple as getting the cleanup table's metatable, and pointing its
`__index` at a table of constructors, where the first parameter is the cleanup
table itself. This means Luau can interpret those constructors as method calls
on the cleanup table, allowing you to use very tidy method call notation.

This is about the minimum level of syntax you could possibly have here, outside
of using awkward global state to track stuff. Your argument lists are kept tidy.
You can still create objects inline without nesting them in parentheses. It even
still works beautifully with curried functions and table/string literal calls.
In fact, the *single* syntax addition over plain code is the name of the cleanup
table in front for the method call. It is exceptionally clean.

All of this is achieved with almost *zero* actual code, too. You only need two
functions. One is just shorthand for creating cleanup tables:

```lua
local function scoped(constructors)
    return setmetatable({}, {__index = constructors})
end
```

The other is used for destroying those cleanup tables, practically the same as
what Maids already implement:

```lua
local function doCleanup(task)
	local taskType = typeof(task)
	if taskType == "Instance" then
		task:Destroy()
	elseif taskType == "RBXScriptConnection" then
		task:Disconnect()
	elseif taskType == "function" then
		task()
	elseif taskType == "table" then
		if typeof(task.destroy) == "function" then
			task:destroy()
		elseif typeof(task.Destroy) == "function" then
			task:Destroy()
		elseif task[1] ~= nil then
			for _, subtask in ipairs(task) do
				doCleanup(subtask)
			end
		end
	end
end
```

23 lines of code for a simple, watertight consolidation solution. Not bad!

## Closing thoughts

This is the fruit of some long-running research I've been doing for Fusion. I
was never truly satisfied with Maids as the 'one solution' for destruction, and
now that I have the words to explain why, it's led me to these key insights.

Make no mistake; this isn't a complete picture, merely one of the puzzle pieces.
I haven't got satisfactory answers for the whole of Fusion's destruction story,
and I'm planning to look into other parts of the destruction problem I can
tackle.

All I know is that, if consolidation is what you're after, this will serve you
for the next 10 years, beyond Maids.