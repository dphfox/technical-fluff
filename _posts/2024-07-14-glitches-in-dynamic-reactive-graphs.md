---
layout: post
title: Glitches in dynamic reactive graphs
image:
  path: /assets/posts/glitches-in-dynamic-reactive-graphs/thumb.jpg
  width: 256
  height: 256
---

My reactive library, Fusion, is built around the idea of assembling reactive objects into reactive graphs for processing data and sending instructions. 

In most cases, whole self-contained graphs will be completely static; built at once and destroyed at once. However, problems start to arise when graphs are dynamically constructed and destroyed - specifically, when a sub-graph tries to observe state that indirectly determines the shape of the sub-graph itself.

Consider this graph.

```lua
local hasSubGraph = scope:Value(true)

local outerComputed = scope:Computed(function(use, scope)
	if use(hasSubGraph) then
		return scope:Computed(function(use)
			return use(hasSubGraph)
		end)
	else
		return nil
	end
end)

scope:Observer(outerComputed):onBind(function()
	local innerComputed = peek(outerComputed)
	if innerComputed == nil then
		print("outerComputed = nil")
	else
		if peek(innerComptued) == true then
			print("innerComputed = true")
		else
			print("innerComputed = false")
		end
	end
end)
```

Intuitively, only two things should happen as you flick `hasSubGraph` on and off:

- when it's off, the observer should print `outerComputed = nil`
- when it's on, the observer should print `innerComputed = true`

Again, *intuitively*, the code should not ever print `innerComputed = false`, because that implies that the inner computed object sees `hasSubGraph == false` while the outer computed object sees `hasSubGraph == true`.

However, depending on execution order, this is entirely possible. Both computed objects are direct dependents of `hasSubGraph`. If the order of updates is not well defined, it's entirely possible that the inner computed will update before the outer computed, leading to the contradiction we thought was impossible just one paragraph ago.

The solution here is pretty simple; outer state objects *must* always run before inner state objects. And as luck would have it, you don't need a complex scope system to achieve that; inner state objects must by necessity be made *after* the outer state object who controls them, so the most practical way of implementing this is by running the oldest objects first, and the newest objects later. This adequately resolves the glitch for all well-formed examples I can think of.