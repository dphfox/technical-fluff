---
layout: post
title: Fallibility lints in Luau
image:
  path: /assets/posts/fallibility-lints-in-luau/thumb.jpg
  width: 256
  height: 256
excerpt: Because, like those who write it, code is fallible,
---
Let's play a game of 'spot the error' :)

```lua
local function performRaycast()
	local mousePos = UserInputService:GetMouseLocation()
	local ray = workspace.Camera:ScreenPointToRay(mousePos.X, mousePos.Y)
	return workspace:Raycast(ray.Origin, ray.Direction * 5000)
end
```

Okay, that one was easy. `workspace.Camera` will error if there isn't an instance called `Camera` inside `workspace`, and furthermore, it's not guaranteed to be of type `Camera` - the only way to guarantee that you're working with an actual camera is via the `CurrentCamera` property.

What about this code snippet? Assuming all Luau types are satisfied, will this error?

```lua
local function generateHueGradient(stops: number): ColorSequence
	local keypoints = {}
	for stop = 1, stops do
		local ratio = (stop - 1) / (stops - 1)
		local hue = Color3.fromHSV(ratio, 1, 1)
		keypoints[stop] = ColorSequenceKeypoint.new(ratio, hue)
	end
	return ColorSequence.new(keypoints)
end
```

Yes, it can! If `stops` is more than `20` or less than `2`, then `ColorSequence.new` will error. 

Okay, what about this code snippet? Same rules as above.

```lua
local function areEqual(x, y): boolean
	return x == y
end
```

Yup! If `x` and `y` have metatables with the same `__eq` function, and that `__eq` function errors while running, then the entire function errors! How simply wonderful!

Point is...

## Luau is a minefield, and you suck at navigating it

... I'm sorry, it's just the truth. Whether you're not correctly checking things exist, not checking your arguments are within bounds, or simply not considering the weird incantations we call metatables, there are one million ways to break your code in ways you don't even realise.

I'm of the strong opinion that, to write better code in Luau, we need better ways of dealing with the existence of errors. We might not need to deal with the most esoteric errors at all times, but at the very minimum, simple footguns need to be flagged up, and right now our only line of defence are a few lints that cover a few problem cases. Not to mention, those lints are completely hardcoded and aren't user-extensible, meaning you can kiss goodbye to any useful lints for library code.

The only thing we can do about errors is try and come up with a more sensible standard in our user code, and just be *extra* careful around standard library/Roblox API functions we call. We can define our own `Result<T>` types and try and stick to them for as long as possible, but there's always a boundary beyond which uncaught errors can infect our code. We're never truly safe.

I'm not youthful enough to believe it's a good idea to uproot Luau's fundamentals and make everything `Result<T>` types or whatever. We need to work with what we've got and maintain backwards compatibility as much as possible. It's also not really possible to fully statically check for errors in Luau due to the halting problem.

So, I think this is the perfect time to introduce an idea that I've been thinking about for a long time.

## Error annotations

The idea is simple; declare how 'dangerous' functions are using annotations.

Here's a function that is guaranteed to never throw an error:

```lua
@errors("never")
function isPart(x: unknown): boolean
	return typeof(x) == "Instance" and x:IsA("BasePart")
end
```

On the opposite end of things, if you have a function that *unconditionally* fails:

```lua
@errors("always")
function logError(message: string)
	error("[ERROR] " .. message)
end
```

If your function is *expected* to fail sometimes as part of normal operation:

```lua
@errors("expected")
function getUserProfile(username: string)
	return HttpService:GetAsync(`{API_ENDPOINT}/users/{username}.json`)
end
```

Or, if it is *not* expected to fail as part of normal operation:

```lua
@errors("unexpected")
local function getNextUser(userId: string)
	return tonumber(userId) + 1
end
```

If that looks verbose, don't worry! While we can't solve the halting problem, it should still be possible to get quite useful inference for these.

If all code paths of a function call an `@errors("always")` function, then that function itself can be inferred to be `@errors("always")`:

```lua
-- inferred: @errors("always")
local function alwaysErrors(condition: boolean)
	assert(condition, "The condition was false")
	if condition then
		error("The condition was true")
	end
end
```

If `@errors("never")` are the only functions used, and there's no potentially fallible operations (as evaluated based on the static types of the operands), then the function can be inferred to be `@errors("never")`:

```lua
-- inferred: @errors("never")
local function neverErrors(ratio: number): number
	return 3 * x^2 + 2 * x^3
end
```

And anywhere in-between, you can infer `@errors("expected")` (or `@errors("unexpected")` if that's the highest-level annotation you discover).

So, why is this useful? Because you can now add a different annotation to lint for usage of functions with these error annotations.

```lua
@should_error("never")
function getUserProfile(username: string)
	-- warning: GetAsync is expected to error; handle the expected error with 
	-- pcall() or remove @errors("never") annotation to silence 
	return HttpService:GetAsync(`{API_ENDPOINT}/users/{username}.json`)
end
```

And, because you can tune your tolerance level, you can reduce how pedantic the checks are by reducing the level of your annotation:

```lua
@should_error("never")
local function areEqual(x, y): boolean
	-- warning: == with possible metatables can error unexpectedly; handle the 
	-- unexpected error with pcall() or remove @errors("never") to silence 
	return x == y
end

@should_error("unexpected")
local function areEqual(x, y): boolean
	return x == y
end
```

This won't stop all runtime errors, but *it doesn't have to!* This is already infinitely more useful than the current status quo, where most Luau code has at least a few non-obvious places where the error handling isn't *quite* up to scratch, and those places are hard to discover if you aren't carefully looking for them.

Plus, since this is an opt-in per-function system, you can save the pedantic checks for parts of your codebase which are critical, such as saving data in DataStores, without breaking any of your other code. It also gives library authors a way to (finally!) introduce some useful lints for dangerous functions that require special care.

## Callbacks and dynamic function values

When function values are passed around, this causes a few complications with this annotation system.

How should `perform` be annotated?

```lua
@errors("always")
local function badMath(): number
	return 2 + nil
end

@errors("never")
local function goodMath(): number
	return 2 + 2
end

-- inferred: ???
local function perform(math: () -> number): number
	return math()
end
```

The error level of the function can't be resolved until there's an actual function call to analyse:

```lua
local fine = perform(goodMath) -- inferred: @errors("never")
local notFine = perform(badMath) -- inferred: @errors("always")
```

Perhaps error level should actually be modelled with the type system? I don't know, I'm not one of the Luau engineers. I don't have good answers for this, and perhaps I have the wrong system proposed here.
## Real-world example: Fusion

Either way, if you'd like a real-world example of how this sort of annotation would prove useful, look no further than Fusion's update process.

If Fusion's update code errors at any point, then the entire application's state will be left in a dangerous, inconsistent state. Best to not do that, then!

The problem? Well... users can insert their own code directly into this process:

```lua
local calculation = scope:Computed(
	function(use)
		return badMath() -- uh oh
	end
)
```

Wouldn't it be nice if you could just:

```lua
local calculation = scope:Computed(
	@should_error("never") 
	function(use)
		-- warning: badMath will always error, handle the error with pcall() or
		-- remove the badMath call to silence
		return badMath()
	end
)
```

Or even better, if the library could annotate the callback for you, automatically letting you know of the constraints the library wants your code to operate under:

```lua
local calculation = scope:Computed(
	function(use)
		-- warning: badMath will always error, handle the error with pcall() or
		-- remove the badMath call to silence
		-- context: argument 1 to Computed() should never error
		return badMath()
	end
)
```

## Conclusions

Now, I'm not going to sit here and proclaim that this is even the right way to approach this problem. In all likelihood, I've gotten some things wrong.

However, I want to start this sort of discussion. Handling errors is incredibly important, and uncaught errors are one of the most nebulous parts of working with Luau. They've certainly caused me the most stress when working with critical systems dealing with player data, and it's caused no end of trouble for the libraries I've developed (and continue to develop to this day!)

I would be all for alternate explorations of ideas like this. I think the Luau analyser is smart enough to be able to do this in *some* form. If annotations aren't the right mechanism for this, maybe types would work better? Or something else completely? I don't know.

I'd like to know what you think.