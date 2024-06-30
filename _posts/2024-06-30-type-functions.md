---
layout: post
title: Type functions
image:
  path: /assets/posts/type-functions/thumb.jpg
  width: 256
  height: 256
---

I just merged a new feature into Fusion. It implements nothing new at all. 

It's literally an identity function:

```lua
local function Child(
	x: Child
): Child
	return x
end
```

And yet, this code serves a pretty important purpose! With this function, Fusion's users will now experience fewer false type checking errors, and get more useful static diagnostics while they're writing code.

I've written identity functions like these many times in many codebases now, so I think it's time to give a name to them. These are *type functions*.

## What is a type function?

Type functions are identity functions that constrain the type of their argument. By convention, the name of the function matches the type that's being constrained to.

## Why would you want to use one?

Type functions can cast the type of values much like `::` can, except it's stronger. It forces the type checker to interpret a value differently, rather than performing usual inference and checking if the inferred and expected types line up at the end.

In the case of `Child`, it prevents Luau from complaining about a mixed list of values, by forcibly constraining the type to be an array of a union type.

```Lua
-- This snippet throws a type error because Luau infers the list is
-- {Instance}, when it's actually {Fusion.Child}.
-- This can't be resolved with :: Child.
local folder = scope:New "Folder" {
    [Children] = {
        scope:New "Part" {
            Name = "Gregory",
            Color = Color3.new(1, 0, 0)
        },
        scope:Computed(function(use)
            return if use(includeModel)
                then modelChildren:GetChildren()
                else nil
        end)
    }
}

-- Using the type function for Child makes this work.
local folder = scope:New "Folder" {
    [Children] = Child {
        scope:New "Part" {
            Name = "Gregory",
            Color = Color3.new(1, 0, 0)
        },
        scope:Computed(function(use)
            return if use(includeModel)
                then modelChildren:GetChildren()
                else nil
        end)
    }
}
```

But I've also used it in other places for different reasons. While working on animated icons for the [Elttob Suite](https://suite.elttob.uk/), I used type functions to declare what type I intended to declare, so that I could get useful autocomplete while I was working, and raise type errors as close to the source of the error as possible by communicating more of my intent to Luau. It also makes the code more readable and understandable for someone who is unfamiliar with what data is being represented in the structure.

```Lua
sun = AnimatedIcon {
	fallbackUrls = {
		primary = "rbxassetid://15057420815"
	},
	duration = 0.4,
	layers = {
		Layer {
			imageUrl = IMAGES.private.sun.sun,
			colour = "primary",
			restTransform = Transform {
				centre = Vector2.new(-0.5, -0.5) / 16,
				size = Vector2.new(15, 15) / 16,
				angle = 0
			},
			motion = Motion {
				type = "simple",
				goalTransform = Transform {
					centre = Vector2.new(-0.5, -0.5) / 16,
					size = Vector2.new(15, 15) / 16,
					angle = TAU / -2
				},
				action = "meet"
			}
		}
	}
}
```

So, if you're struggling with type inference in Luau, perhaps try using type functions to effectively and cleanly constrain your types. You'll get fewer false positives and higher quality autocomplete for practically zero cost.