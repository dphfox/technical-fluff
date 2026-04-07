---
layout: post
title: Flip caches & information loss
image:
  path: /assets/posts/flip-caches-and-information-loss/thumb.jpg
  width: 256
  height: 256
---

I've been exploring ways of introducing caching and incremental recomputation to immediate-mode UI systems, in line with a new information loss principle I'm thinking about.

## VDOM

For a decade now, I haven't been a fan of UI approaches that involve comparing trees of elements. Often, these are referred to as "virtual DOMs" or VDOMs (this is terminology inherited from the web).

My main criticism of these approaches is that they throw away granularity information that exists at the time of building the VDOM. By design, the VDOM assumes that everything can change between re-renders of UI, and so must compare everything to retrieve the actual minimal set of changes desired.

If the original builder of the VDOM specified the minimal change they wanted, this work is totally avoidable. In its absence, even a simple property change can balloon into a lot of work done by the CPU just to re-infer the information.

## Fine-grained reactivity

My goal when building Fusion (my fine-grained reactivity framework) five years ago was to eliminate this re-inference step. Instead of comparing trees of elements, my idea was to explicitly allow bindings to be made between properties of elements and objects in the code. If the object receives a change, the property is updated.

The key innovation of the system was to allow objects to bind to other objects. You could define computations that take the current state of some objects and returns a newly bindable state. When used in a UI, this established a granular chain of cause-and-effect through the UI, and allowed for changes to be applied directly without re-inferring anything.

However, this still didn't satisfy me completely. The most obvious issue is a colouring problem; code that is written "reactively" has more ceremony than code which is written "traditionally". The cost to the programmer of managing a fleet of objects required supporting concepts in memory management and helper utilities for improving ergonomics.

There's also a more subtle cost in frameworks like this; they tend to over-memoise information by their nature. Since every object is a binding, every object has a state that needs to be synchronised to its users. This typically means the object is a state container. As a result, you effectively memoise every causal node in the system, which often means memoising every piece of dynamic state, even trivially inferrable ones. The cost of this memory has been shown to be notable in practice - it would be more ideal to create a scheme that more naturally encourages recomputation of trivial states rather than universal memoisation.

## The principle of information loss

From my experience, using fine-grained reactivity has led to almost universally more predictable and favourable performance compared to VDOM systems. Many of the highest-performance frameworks today adopt similar principles.

But why is it better? In my eyes, it's because it moves memoisation work closer to the subject being memoised.

Since the memoisation done by the reactive objects can take advantage of information available in the local context, they're able to track changes granularly without re-inferring any missing information. The bindings and objects involved are directly and trivially known.

Attempting to apply this principle, I would extrapolate that the best memoisation techniques are likely ones which deduplicate calculations "as they happen" - parts of the computation which wish to reuse historical data can elect to do so on their own, in the same path of computation that returns the result of the computation. To the end user, the computation simply provides a result - caching details are encapsulated in some kind of context side-effect.

That's what I've been exploring next, as part of a reductive approach I call "flip caches".

## Flip caches

Let's consider an example of a gallery app. We want to show a bunch of photos with different widths and heights in a list view, similar to Google Images.

Our state may look like this:

```rust
struct Photo {
	width: f32,
	height: f32,
	image_content: Texture
}

struct UIState {
	photos: Vec<Photo>
}
```

Let's write a naive immediate-mode function that implements a wrapping layout.

```rust
fn build(state: &mut UIState, renderer: &mut Renderer) {
	const MIN_ROW_HEIGHT: f32 = 100.0;
	const GAP: f32 = 4.0;

	 // Break photos onto rows bounded by screen width
	let photo_rows = {
		let mut x = 0;
		let mut index = 0;
		let mut photo_row = vec![];
		let mut photo_rows = vec![];

		for photo in state.photos {
			let height = MIN_ROW_HEIGHT;
			let width = (photo.width / photo.height) * height;
			if x + width > imgui.screen_width {
				photo_rows.push(photo_row);
				photo_row = vec![];
			}
			photo_row.push(photo);
			x += width + GAP;
		}
		if !photo_row.is_empty() {
			photo_rows.push(photo_row);
		}
		photo_rows
	}

	// Render rows of photos scaled to fill screen width
	let mut y = 0.0;
	for photo_row in photo_rows {
		let total_gap_widths = (photo_row.len() - 1) as f32 * GAP;
		let total_photo_widths = photo_row.iter().map(|p| p.width).sum();
		let scale_factor = (imgui.screen_width - total_gap_widths) / total_photo_widths;
		let height = MIN_ROW_HEIGHT * scale_factor;
		let mut x = 0.0;
		for photo in photo_row {
			let width = photo.width * scale_factor;
			renderer.draw(Draw::Texture { x, y, width, height, texture: photo.image_content });
			x += width + GAP;
		}
		y += height;
	}
}
```

Every redraw event, we compute the layout in full, then directly send instructions to the renderer to draw textures where we want them. This is stateless and does not memoise, which makes it very easy to work with, but discards a lot of information. That's especially bad when working with expensive layouts that involve many elements.

Let's assume for demonstration that the `photo_rows` computation is a hotspot (perhaps it's allocating too much, don't ask!). Instead of immediately reaching for a VDOM or fine-grained reactivity, we could instead keep a cache between rebuilds. Maybe it looks like this:


```rust
struct PhotoRows(Vec<Vec<Photo>>);

fn build(state: &mut UIState, renderer: &mut Renderer, cache: &mut Cache) {
	const MIN_ROW_HEIGHT: f32 = 100.0;
	const GAP: f32 = 4.0;

	 // Break photos onto rows bounded by screen width
	let photo_rows = cache.get_or_init::<PhotoRows>(|| {
		let mut x = 0;
		let mut index = 0;
		let mut photo_row = vec![];
		let mut photo_rows = vec![];
		for photo in state.photos {
			let height = MIN_ROW_HEIGHT;
			let width = (photo.width / photo.height) * height;
			if x + width > imgui.screen_width {
				photo_rows.push(photo_row);
				photo_row = vec![];
			}
			photo_row.push(photo);
			x += width + GAP;
		}
		if !photo_row.is_empty() {
			photo_rows.push(photo_row);
		}
		PhotoRows(photo_rows)
	});

	// Render rows of photos scaled to fill screen width
	let mut y = 0.0;
	for photo_row in photo_rows {
		let total_gap_widths = (photo_row.len() - 1) as f32 * GAP;
		let total_photo_widths = photo_row.iter().map(|p| p.width).sum();
		let scale_factor = (imgui.screen_width - total_gap_widths) / total_photo_widths;
		let height = MIN_ROW_HEIGHT * scale_factor;
		let mut x = 0.0;
		for photo in photo_row {
			let width = photo.width * scale_factor;
			renderer.draw(Draw::Texture { x, y, width, height, texture: photo.image_content });
			x += width + GAP;
		}
		y += height;
	}
}
```

The idea here is that `cache.get_or_init::<T>` only runs the inner closure once, then it saves the returned `T` and uses it for all subsequent calls. (The type exists to allow for multiple different pieces of data to be independently cached in a type-inferable way.)

This cache solves our memoisation problem for this expensive layout computation without requiring a deep structural change to any code. Since the mechanism is generic, it can encapsulate as much or as little of the code as profiling indicates is necessary.

For example, if constructing commands for the renderer was also expensive, that could be memoised instead:

```rust
struct PhotoDrawInstructions(Vec<Draw>);

fn build(state: &mut UIState, renderer: &mut Renderer, cache: &mut Cache) {
	const MIN_ROW_HEIGHT: f32 = 100.0;
	const GAP: f32 = 4.0;

	let instructions = cache.get_or_init::<PhotoDrawInstructions>(|| {
		// Break photos onto rows bounded by screen width
		let photo_rows = {
			let mut x = 0;
			let mut index = 0;
			let mut photo_row = vec![];
			let mut photo_rows = vec![];

			for photo in state.photos {
				let height = MIN_ROW_HEIGHT;
				let width = (photo.width / photo.height) * height;
				if x + width > imgui.screen_width {
					photo_rows.push(photo_row);
					photo_row = vec![];
				}
				photo_row.push(photo);
				x += width + GAP;
			}
			if !photo_row.is_empty() {
				photo_rows.push(photo_row);
			}
			photo_rows
		}

		// Render rows of photos scaled to fill screen width
		let mut instructions = vec![];
		let mut y = 0.0;
		for photo_row in photo_rows {
			let total_gap_widths = (photo_row.len() - 1) as f32 * GAP;
			let total_photo_widths = photo_row.iter().map(|p| p.width).sum();
			let scale_factor = (imgui.screen_width - total_gap_widths) / total_photo_widths;
			let height = MIN_ROW_HEIGHT * scale_factor;
			let mut x = 0.0;
			for photo in photo_row {
				let width = photo.width * scale_factor;
				instructions.push(Draw::Texture { x, y, width, height, texture: photo.image_content });
				x += width + GAP;
			}
			y += height;
		}

		PhotoDrawInstructions(instructions)
	});

	for inst in instructions {
		renderer.draw(inst);
	}
}
```

The desirable property of this solution is that it obeys Tennant's Correspondence Principle: memoising a passage of code does not require restructuring as fine-grained reactivity does. The default assumption made is that computations are trivial (which they often are).

Of course, the problem now is cache invalidation, but we have existing techniques to address this.

```rust
impl UIState {
	fn set_photos(&mut self, photos: Vec<Photo>) {
		self.photos = photos;
		self.photos_changed = true;
	}
}

// ... later ...

if self.photos_changed {
	cache.clear::<PhotoDrawInstructions>();
}
```

To implement such a cache, I would use a "flipping" method to allow read access to the previous version of the cache while allowing write access to a fresh version of the cache. This is why I use "flip caches" to describe this technique; it only requires space for two `T` allocations, plus a little book-keeping in the cache object itself, and flips between these allocations for predictable performance.

## Why I think this could be useful

I haven't yet implemented a fuller UI system using flip caches, but I like the solution because it essentially filters rebuild events for closures which still rebuild in an immediate-mode fashion. There is no colouring problem like fine-grained reactive signals have.

Additionally, flip caches avoid the information loss problem of VDOM techniques; since the caching happens "inline" in the build function, diffing is unnecessary - the renderer receives the draw instructions it needs directly.

Of course, there's one caveat to all of this, which is that re-constructing trees of components every rebuild could become expensive; a forest of small components can add up in a way that isn't true of fine-grained reactive systems, which construct their trees once.

I'm interested to explore this system further to see whether higher-level abstractions can be built from this, to reach similar ergonomics to fine-grained reactive systems. In particular, I'm interested in orthogonal techniques to do with cache invalidation patterns, and how these caches could perhaps carry incidental UI state such as animation timelines.