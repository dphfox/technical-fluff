---
layout: post
title: math.round considered harmful
image:
  path: /assets/posts/math-round-considered-harmful/thumb.jpg
  width: 256
  height: 256
---

`math.round(x)` is the same as `math.floor(x + 0.5)`, *riiiiiiiiiiiiight?*

***...riiiiiiiiiiight?***

We'll get back to that. In the meantime, here's a fun icebreaker for an especially dull party: what way should the floor operation round?

Some people think it should round towards zero. That is to say, `floor(-0.9) == 0`. This definition has the property of being symmetric around zero, meaning negative numbers round the same way as their positive number counterparts.

Other people think that it should round towards negative infinity. That is to say, `floor(-0.9) == -1`. This definition will always round down any non-integers.

Ditto for `ceil()` functions. Some people think they round away from zero, and others think they round always-upwards towards positive infinity.

Importantly, ditto for `round()` functions. When you `round(0.5)` (or any other perfectly-in-between number), it can either round towards zero, it can round up, or it can round down.

In Lua 5.1, `math.floor()` always rounds down. Similarly, `math.ceil()` always rounds up. Luau later implemented the `math.round()` function, which users *widely believe* to be equivalent to `math.floor(x + 0.5)`. That is to say, it rounds `0.5` down.

If you believe this, you are wrong. 

It *actually* rounds away from zero.

Yes, I was just as surprised as you are.

So, why is this harmful? Well, it turns out that people actually *depend* on that rounding-down behaviour implicitly in a bunch of code, because *they think it works the way floor does*.

Por ejemplo;

```lua
local CELL_SIZE = 16

-- Finds the nearest 'dual' grid cell for this position.
local function nearestDualCell(pos: Vector3): Vector3
	return Vector3.new(
		math.round(pos / CELL_SIZE),
		math.round(pos / CELL_SIZE),
		math.round(pos / CELL_SIZE)
	)
end
```

If the above code encounters a position with any negative .5 ordinates/components, it will give you the wrong grid cell. Specifically, as you cross the number zero, you will end up with inconsistent behaviour, because the direction of rounding changes as you pass over zero.

The only way to fix the code snippet is to have a consistent rounding direction. That is to say, you either use `math.floor(x + 0.5)` (which rounds any .5 down every time) or `math.ceil(x - 0.5)` (which rounds any .5 every time).

```lua
local CELL_SIZE = 16

-- Finds the nearest 'dual' grid cell for this position.
local function nearestDualCell(pos: Vector3): Vector3
	return Vector3.new(
		math.floor(pos / CELL_SIZE + 0.5),
		math.floor(pos / CELL_SIZE + 0.5),
		math.floor(pos / CELL_SIZE + 0.5)
	)
end
```

To summarise:

- `math.round(x)` rounds any .5 inconsistently - upwards for negatives, downwards for positives.
- `math.floor(x + 0.5)` rounds any .5 downwards, consistently
- `math.ceil(x - 0.5)` rounds any .5 upwards, consistently

The difference matters. Choose an opinion and document why in your code.

I would strongly recommend setting up a lint for `math.round(x)` to make sure your team members (and you in six months time are aware of the issue and are thinking actively about it. There is little use case for having inconsistent behaviour, and if you do by chance need to round towards zero you can suppress the lint for that one use and document why you're suppressing it. Otherwise, flagging it up is a good idea, and using a more consistently behaving alternative is best.