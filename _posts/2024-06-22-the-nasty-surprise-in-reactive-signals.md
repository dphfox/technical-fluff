---
layout: post
title: The nasty surprise in reactive signals
image:
  path: /assets/posts/the-nasty-surprise-in-reactive-signals/thumb.jpg
  width: 256
  height: 256
excerpt: If your reactive framework isn't doing this, then you're in for a nasty surprise.
---

Reactive signals are often a nice concept. You can write calculations as you'd expect, and they'll re-run automatically whenever *anything* you use changes.

```lua
-- Fusion 0.3 syntax
local numCoins = scope:Value(50)
local itemPrice = scope:Value(10)

local finalCoins = scope:Computed(function(use)
	return use(numCoins) - use(itemPrice)
end)
```

Notice something about the way I just wrote that example. See that `Computed` I wrote? Notice how the parameters of the calculation are wrapped in this `use` call, and how that `use` function is passed in from the `Computed`?

If your reactive framework isn't doing this, then you're in for a nasty surprise.

## Unsexy code saves lives

Fusion used to use an entirely different syntax for this. Due to rampant problems with code maintainability, we had to switch.

The old syntax looks innocent enough. Don't mind the lack of `scope:` - that's unrelated. Pay attention to the inside of the `Computed`.

```lua
-- Fusion 0.2 syntax
local numCoins = Value(50)
local itemPrice = Value(10)

local finalCoins = Computed(function()
	return numCoins:get() - itemPrice:get()
end)
```

Magical `:get()` calls. Older versions of Fusion had *quite complex* machinery inside of them to track whenever you called these `:get()` methods, and form dependencies between the objects automatically. We used to call it the *automatic dependency manager*.

Many people who try and improve on Fusion's design often reach for an automatic dependency manager, because it's what everyone else is doing, and it looks nicer than having to accept a parameter. Here's Vide, which has been sent to my inbox more times than I can remember;

```lua
-- Vide syntax
local numCoins = source(50)
local itemPrice = source(10)

local finalCoins = derive(function()
	return numCoins() - itemPrice()
end)
```

I'll give Vide credit, they would beat Fusion if this was a beauty contest. However, in the pursuit of beauty, they've made the exact same showstopping issue that Fusion made in its early development, and the exact same showstopping issue that basically the whole web ecosystem is walking into blindly.

Magic dependencies *never* scale.

## Button pushing problems

Consider this [standard implementation of an Event primitive in Luau](https://github.com/Elttob/LibOpen/blob/main/LibOpen/Event/init.lua). Don't consider it for too long though, it isn't particularly interesting.

```lua
--!strict
-- (c) Daniel P H Fox, licensed under MIT

export type Connect<Args...> = (callback: (Args...) -> ()) -> () -> ()

local function Event<Args...>(): (
	Connect<Args...>, 
	(Args...) -> ()
)
	local listeners: {[{}]: (Args...) -> ()} = {}
	local function connect(
		callback: (Args...) -> ()
	): () -> ()
		local id = {}
		listeners[id] = callback
		return function(): ()
			listeners[id] = nil
		end
	end
	local function fire(
		...: Args...
	): ()
		for _, listener in listeners do
			task.spawn(listener, ...)
		end
	end
	return connect, fire
end

return Event
```

I'll set up a button push event, which identifies the button that's just been pushed by passing a string value. I'll set up some Vide sources that will count how many times we push a certain button in a row.

```lua
local doButtonPushed, onButtonPushed: (string) -> () = Event()

local lastButton = source(nil)
local numPushes = source(0)

onButtonPushed(function(whichButton)
    if whichButton ~= lastButton() then
		numPushes(1)
		lastButton(whichButton)
	else
		numPushes(numPushes() + 1)
	end
	print("Pushed", whichButton, numPushes(), "times")
end)
```

Finally, I'll simulate some button clicks.

```lua
doButtonPushed("red") --> Pushed red 1 times
doButtonPushed("red") --> Pushed red 2 times
doButtonPushed("red") --> Pushed red 3 times
doButtonPushed("green") --> Pushed green 1 times
doButtonPushed("red") --> Pushed red 1 times
doButtonPushed("green") --> Pushed green 1 times
doButtonPushed("green") --> Pushed green 2 times
doButtonPushed("blue") --> Pushed blue 1 times
```

It works perfectly fine. If you were prototyping this system in a blank file, this would probably be enough to convince you to integrate this with your larger project.

Except - whoops - now for some reason, all of your UI state gets reset whenever a button gets pushed. What happened?

## Effectful reads

Consider this code that's creating some UI in a reactive context. There's many real world places where this sort of thing would happen, for example if you were creating one UI for all of the button colours that appear in a list.

```lua
derive(function()
	local buttonUI = create "TextButton" {
	
	-- ... some time later ...
	
	doButtonPushed(buttonColour)
end)
```

The details of what's happening here aren't important. What *is* important, however, is that `doButtonPushed` has been called from within a reactive context, for some reason or another.

So what happens next? The internal machinery of `Event` whirrs, and calls our `onButtonPushed` handler we defined before.

Which *juuuust* so happens to be where we're reading `lastButton()` and `numPushes()`.

Which adds a dependency quietly that we had no idea about.

Which will re-run the reactive block where we're building our UI and storing that UI's state.

Shit.

## This isn't hypothetical

I've seen this play out over and over again whenever someone using Fusion 0.2 complains that their UI is re-rendering or losing private state whenever some random thing occurs. More often than not, the problem is simple, and always the same: their code was reading a state value from deep within some stack of functions that was originally called from a reactive context.

To be clear, it's not like Vide has forgotten this issue exists. They have an `untrack()` primitive for exactly this sort of use case. But how are you supposed to use a mitigation when you can't even see where the problem is coming from? Do we have to modify all of our code to defend against reactive dependencies by default, out of an abundance of caution? What about external libraries whose source code we might not understand, or code not owned by a development team which they're not allowed to modify? What about cases where you want to track *some* uses but not others? What if you want to intercept and mock out those uses? What if you're building your own object and want to get a bit more unique with your dependency management?

Vide has no answers for these questions. Neither did Fusion 0.2. Neither will nearly any library that elects to use automatic dependency management as a syntax crutch.

Fusion learned its lesson. Instead of clever magic, Fusion 0.3 takes things back to basics. Computed objects give you a function you can call to add a dependency. That's our dependency manager. If you don't call it, you don't add a dependency.

```lua
local finalCoins = scope:Computed(function(use)
	-- This will not add a dependency, ever, regardless of context.
	-- `peek()` is a top-level function in Fusion and can be used anywhere.
	local x = peek(numCoins)
	-- This will always add a dependency to exactly this computed, regardless of
	-- context. It's only available right here, right now.
	local x = use(numCoins)
end)
```

This makes the intention clear, first of all. If you don't intend to form a reactive dependency, but are instead writing one-shot code, then you can use `peek()` and be done with it. On the flip side, if you're trying to write reactive code, then you can switch over to `use()` to indicate that's what you want to do.

On top of clear intentions, having separate functions for these things, and having `use()` be explicitly lexically scoped to `Computed` callbacks like this, means that it is a compile-time error to try and `use()` something outside of a scope where such a `use()` function is available. That's just leveraging the basics of how Luau works.

And finally, this is most important because it allows you to define the *capabilities* of functions you call into. If you don't pass `use` to a function you call, then it is *guaranteed impossible* for that function to *ever* form a dependency you didn't expect. In this way, the places where dependencies can come from are *always* explicitly marked and visible, which pretty much eliminates the problem of dependency hunting. You know where it comes from, because you know where you gave permission.

It isn't magic. It's predictable. And it isn't even that much more verbose. Was Vide's approach *really* a necessary trade-off?

All of this is before we even begin to consider things like how Fusion's newer `peek()` and `use()` functions are explicitly separate from the objects themselves, so as to enable constant value passthrough, which is incredibly important for scaling up a codebase and building reusable code snippets that are agnostic to reactivity.

## So what am I trying to say?

If you're going to reinvent Fusion, you should be doing so from a place of achieving correct, locally reasonable code. If you choose to pursue minimal syntax and "clever" inference (or what we've historically called "saving keystrokes"), you almost always run into issues with correctness and locality, not to mention issues with preserving user intent. When you elide detail from syntax, you are often throwing away load-bearing detail. 

Vide has made this mistake and touts it as a benefit. Much of the JS ecosystem is making this mistake as we speak, too, and some also tout it as a benefit in exactly the same way. I'm seeing this pattern emerge in other hyped areas of computer science right now, such as algebraic effect systems. Magic never scales - yet we seem to be fond of this rake.

Fusion explicitly takes a Rust-ic approach to syntax. We don't minimise it. Instead, we make it mean something. There are very few parts of Fusion where a piece of syntax *isn't* conveying important information about how the code should be expected to behave - it's all designed very intentionally and carefully so that code runs exactly as it reads.

Vide has many good ideas that I would like to integrate into Fusion. We're currently approaching the end of a months-long internal refactor that should allow us to integrate most, if not all, of the good ideas that Vide has brought to the table. I'm thankful for their work. 

But as equally as I would praise some of their choices, I must criticise the parts of their design that re-tread known dangerous ground, in the hopes that others don't follow. I would steer clear of using Vide at scale, because you *will* be bitten by dependency issues sooner or later.

Ask this Fusion 0.2 user how they know that.