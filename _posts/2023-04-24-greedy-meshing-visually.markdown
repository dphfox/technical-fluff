---
layout: post
title:  "Greedy meshing, visually"
image:
  path: "/assets/posts/greedy-meshing-visually/thumb.jpg"
  width: 256
  height: 256
comments_url: "https://elk.zone/tech.lgbt/@Elttob/110254765144551186"
---

Greedy meshing blocks in a voxel game doesn't have to be hard. Here's a simple
explanation for people who prefer seeing how the algorithm progresses.

*This post is the newer version of my popular tutorial from the Roblox DevForum:*
[Consume everything - how greedy meshing works](https://devforum.roblox.com/t/consume-everything-how-greedy-meshing-works/452717)

## The algorithm

- Find a starting point, that isn't empty, and hasn't already been merged.
- For each axis, one at a time (X, Y, Z)
	- Look one coordinate ahead. Is there a block?
	- If so, merge it with the previous block and step forward.
	- If not, then move on to the next axis.
- Repeat the above steps until you have merged every block.

By the way, finding a starting point isn't too hard. It's pretty sufficient to
just loop over them in the normal way - you'll always have merged every block by
the end, and it's relatively efficient at merging things into large groups.

```lua
for x = 1, size.x do
    for y = 1, size.y do
        for z = 1, size.z do
			-- if this is a starting point, start merging stuff...
		end
	end
end
```

## A simple visual example along one axis

<img style="display: block; max-width: 25rem; margin: 6rem auto;" src="/assets/posts/greedy-meshing-visually/visual1.svg" alt="A row of 5 blocks in a straight line is greedy meshed with the algorithm above, producing one group containing the whole row.">

## The same example, with two axes (2D)

<img style="display: block; max-width: 25rem; margin: 6rem auto;" src="/assets/posts/greedy-meshing-visually/visual2.svg" alt="A 5 by 5 area of blocks is greedy meshed with the algorithm above, producing one group containing the whole area.">

## A more complex demonstration

<img style="display: block; max-width: 25rem; margin: 6rem auto;" src="/assets/posts/greedy-meshing-visually/visual3.svg" alt="Two 3 by 3 areas of blocks, which share only one of their corner blocks. When greedy meshed, a 3 by 3 group is created for one of the areas, and to fill the remaining L shape, a 3 by 2 group and a 2 by 1 group are created.">