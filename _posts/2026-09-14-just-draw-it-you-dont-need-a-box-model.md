---
layout: post
title: "Just draw it: you don't need a box model"
image:
  path: /assets/posts/just-draw-it-you-dont-need-a-box-model/thumb.jpg
  width: 256
  height: 256
  excerpt: Instead of running away from presentation problems and imposing abstractions, we're going to plow right through the heart of the them, and discover a very useful shape algebra along the way.
---

*This is Chapter 2 of [a 3-part blog post series](https://fluff.blog/2026/09/13/i-made-my-perfect-ui-library.html) discussing "Perfection", a greenfield UI library I built for personal use.*

---

In the last blog post, we discovered an equivalence between immediate mode and reactive signal UI styles, and how one is mechanically translatable into the other. At the end of that post, I left you with a teaser for our very first UI presentation problem we would run into.

In this blog post, instead of running away from those problems and imposing abstractions, we're going to plow right through the heart of the problem, and discover a very useful shape algebra along the way.

---
## Hey, let's just draw stuff!

So - we're doing immediate mode. Let's bring it back to [that Clay introduction video](https://www.youtube.com/watch?v=DYWTw19_8r4), and pretend like we're beginners and just wrote a `render_ui()` function with a bunch of calls to directly draw things on the GPU, all with hardcoded coordinates.

```rust
fn render_ui(mut ctx: UiContext) {
	// Render a nice background.
	draw_quad(0, 0, 104, 120, Colour::DARK_BLUE);
	// Then, a bunch of labels stacked vertically.
	draw_text("File", 4, 4, Colour::WHITE);
	draw_text("New", 4, 28, Colour::WHITE);
	draw_text("Open", 4, 52, Colour::WHITE);
	draw_text("Save", 4, 76, Colour::WHITE);
	draw_text("Upload", 4, 100, Colour::WHITE);
}
```

As Nic predicted in the video, we can see this isn't going to work with all these numbers to remember and keep in sync. But instead of bailing out to a library like Clay, let's actually make an effort to do all those calculations ourselves and see how bad it gets.

Since we're just stacking things on top of each other, we could make it a _little_ less impossible-to-modify by keeping a running total of our position and adding onto it. We'll also throw in a few other variable names, too.

```rust
fn render_ui(mut ctx: UiContext) {
	let text_pad = 4;
	let item_height = 16 + text_pad * 2;
	let menu_width = 104;
	let menu_height = 120;
	draw_quad(0, 0, menu_width, menu_height, Colour::DARK_BLUE);
	
	let mut top_of_item = 0;
	draw_text("File", text_pad, top_of_item + text_pad, Colour::WHITE);
	top_of_item += item_height;
	draw_text("New", text_pad, top_of_item + text_pad, Colour::WHITE);
	top_of_item += item_height;
	draw_text("Open", text_pad, top_of_item + text_pad, Colour::WHITE);
	top_of_item += item_height;
	draw_text("Save", text_pad, top_of_item + text_pad, Colour::WHITE);
	top_of_item += item_height;
	draw_text("Upload", text_pad, top_of_item + text_pad, Colour::WHITE);
}
```

And hey, that sure is a bunch of duplication. Maybe we should use a loop.

```rust
fn render_ui(mut ctx: UiContext) {
	let text_pad = 4;
	let item_height = 16 + text_pad * 2;
	let menu_width = 104;
	let menu_height = 120;
	draw_quad(0, 0, menu_width, menu_height, Colour::DARK_BLUE);
	
	let mut top_of_item = 0;
	for item in &["File", "New", "Open", "Save", "Upload"] {
		draw_text(item, text_pad, top_of_item + text_pad, Colour::WHITE);
		top_of_item += item_height;
	}
}
```

Okay, so the code _is_ a little cleaner now. Some people, like Casey, do actually write some UIs pretty much like this - using incrementing variables and drawing as-you-go. 

But by now, you can probably feel what Nic was saying. There's still hardcoded stuff here, and we can't fix that without solving some circular logic puzzles.

For example, it's great that we implemented this automatic for-loop for the items, but we've still hardcoded the background. We _have_ to draw the background before the text, so we need to know the width and height ahead of time. But _also_ - we want the background to fit around the width and height of the text, which we haven't rendered yet!

This circular logic happens because we have mixed up a couple of concepts in this code, which really want to separate out into different layers. If we keep pushing forward, you'll start to see the layers unmix, a bit like oil coming out of water.

## Unmixing layout from rendering

Here's the first move we'll take to unpick this tangle. We'll take some duplicate measurements in a second loop at the start of the function so that we can calculate a sufficient menu size.

This doesn't feel as nice now, because we have to lay out the text once for measuring it, then we have to lay it out again to draw it, but it *does* make the background sizing work, and it _does_ mean we can base the item heights on the actual text bounds, rather than a hardcoded 16 pixels.

```rust
fn render_ui(mut ctx: UiContext) {
	let text_pad = 4;
	let mut menu_width = 0;
	let mut menu_height = 0;
	for item in &["File", "New", "Open", "Save", "Upload"] {
		let laid_out_text = lay_out_text(item);
		menu_width = menu_width.max(laid_out_text.width() + text_pad * 2);
		menu_height += laid_out_text.height() + text_pad * 2;
	}
	
	draw_quad(0, 0, menu_width, menu_height, Colour::DARK_BLUE);
	
	let mut top_of_item = 0;
	for item in &["File", "New", "Open", "Save", "Upload"] {
		let laid_out_text = draw_text(item, text_pad, top_of_item + text_pad, Colour::WHITE);
		top_of_item += laid_out_text.height() + text_pad * 2;
	}
}
```

To feel less bad, let's save that perfectly good `lay_out_text()` result and pass it into `render_text()`, reusing all that work.

```rust
fn render_ui(mut ctx: UiContext) {
	let text_pad = 4;
	let mut menu_width = 0;
	let mut menu_height = 0;
	let mut laid_out_texts = vec![];
	for item in &["File", "New", "Open", "Save", "Upload"] {
		let laid_out_text = lay_out_text(item);
		menu_width = menu_width.max(laid_out_text.width() + text_pad * 2);
		menu_height += laid_out_text.height() + text_pad * 2;
		laid_out_texts.push(laid_out_text);
	}
	
	draw_quad(0, 0, menu_width, menu_height, Colour::DARK_BLUE);
	
	let mut top_of_item = 0;
	for laid_out_text in laid_out_texts.iter() {
		draw_text(laid_out_text, text_pad, top_of_item + text_pad, Colour::WHITE);
		top_of_item += laid_out_text.height() + text_pad * 2;
	}
}
```

Let's move the positioning logic up there as well, so positions and sizes stay together. We'll also move the text colour up there so that text drawing becomes completely mechanical and parameter-free.

```rust
fn render_ui(mut ctx: UiContext) {
	let text_pad = 4;
	let mut menu_width = 0;
	let mut menu_height = 0;
	let mut laid_out_texts = vec![];
	for item in &["File", "New", "Open", "Save", "Upload"] {
		let laid_out_text = lay_out_text(item, text_pad, menu_height + text_pad, Colour::WHITE);
		menu_width = menu_width.max(laid_out_text.width() + text_pad * 2);
		menu_height += laid_out_text.height() + text_pad * 2;
		laid_out_texts.push(laid_out_text);
	}
	
	draw_quad(0, 0, menu_width, menu_height, Colour::DARK_BLUE);
	for laid_out_text in laid_out_texts.iter() {
		draw_text(laid_out_text);
	}
}
```

By separating out _layout_ as a separate stage before rendering, you're free to compute things in whatever order you need. The choice of what appears in-front or behind becomes orthogonal - you just change the order of the render operations at the end, not any of your logic, leaving you free to encode the order of evaluation as dependencies require you to.

## Introducing shapes

Let's now talk about the fact we're calling draw methods at all. If you think back to Chapter 1, you'd remember our caching mechanism allows us to skip running logic on some frames.

However, you can't do that for a draw instruction. If it doesn't run, the shape doesn't show up.

```rust
// Bad! Caching stops draw instructions from running.
ctx.expensive(|get| {
	move || draw_text(laid_out_text);
})
```

So, instead of doing the drawing ourselves, we'll just return a list of shapes - same data, different delivery mechanism. We can kick out the drawing of that list to a different place.

```rust
fn render_ui(mut ctx: UiContext) -> impl Paint {
	let text_pad = 4;
	let mut menu_width = 0;
	let mut menu_height = 0;
	let mut text_shapes = vec![];
	for item in &["File", "New", "Open", "Save", "Upload"] {
		let text_shape = render_text(item, text_pad, menu_height + text_pad, Colour::WHITE);
		menu_width = menu_width.max(text_shape.width() + text_pad * 2);
		menu_height += text_shape.height() + text_pad * 2;
		text_shapes.push(text_shape);
	}
	
	let mut composition = vec![];
	composition.push(Quad::solid(0, 0, menu_width, menu_height, Colour::DARK_BLUE));
	for text_shape in text_shapes {
		composition.push(text_shape);
	}
	composition
}
```

Now, the UI code is completely pure, and shapes are now cacheable data, so they can live inside of caches whenever they need to. That's meaningfully more powerful, bought for very little mental cost!

For convenience, while we're here and talking about mental cost, why not define a trivial `compose!` helper which does the final merge-down into a single flat list? This makes the paint ordering much simpler to see; it'll be ordered how the arguments are ordered.

```rust
fn render_ui(mut ctx: UiContext) -> impl Paint {
	let text_pad = 4;
	let mut menu_width = 0;
	let mut menu_height = 0;
	let mut text_shapes = vec![];
	for item in &["File", "New", "Open", "Save", "Upload"] {
		let text_shape = render_text(item, text_pad, menu_height + text_pad, Colour::WHITE);
		menu_width = menu_width.max(text_shape.width() + text_pad * 2);
		menu_height += text_shape.height() + text_pad * 2;
		text_shapes.push(text_shape);
	}
	
	// Same logic as previously, just nicer to write.
	compose! [
		Quad::solid(0, 0, menu_width, menu_height, Colour::DARK_BLUE),
		text_shapes
	]
}
```

## Operating on shapes

Let's throw a curveball; up until now, we've been rendering relative to `0, 0`. That is to say, we have a top-left-aligned menu.

How would you centre this UI so that the background quad is nicely aligned on the screen?

Your first instinct may be to derive some `centre_x` and `centre_y` variables, and feed them into the positions of `render_text` and `Quad::solid`, so let's try that and see where we end up.

```rust
fn render_ui(
	mut ctx: UiContext,
	screen_bounds: PxRect
) -> impl Paint {
	let text_pad = 4;
	// Try and calculate the top-left corner of the menu.
	// But this depends on data we haven't computed yet...
	let menu_x = (screen_bounds.width() - menu_width) / 2;
	let menu_y = (screen_bounds.height() - menu_height) / 2;
	let mut menu_width = 0;
	let mut menu_height = 0;
	let mut text_shapes = vec![];
	for item in &["File", "New", "Open", "Save", "Upload"] {
		let text_shape = render_text(item, menu_x + text_pad, menu_y + menu_height + text_pad, Colour::WHITE);
		menu_width = menu_width.max(text_shape.width() + text_pad * 2);
		menu_height += text_shape.height() + text_pad * 2;
		text_shapes.push(text_shape);
	}

	compose! [
		Quad::solid(menu_x, menu_y, menu_x + menu_width, menu_y + menu_height, Colour::DARK_BLUE),
		text_shapes
	]
}
```

Just like last time, we've found another cyclic dependency. We want to position the contents of the menu relative to its top-left corner, but we don't know where the menu's top-left corner is until we've finished rendering the menu.

We _could_ compute everything twice - after all, we know it's possible to get the menu width. Let's stomach that for a second and see where it gets us.

```rust
fn render_ui(
	mut ctx: UiContext,
	screen_bounds: PxRect
) -> impl Paint {
	let text_pad = 4;
	// Compute the width of the menu _first_.
	let mut menu_width = 0;
	let mut menu_height = 0;
	for item in &["File", "New", "Open", "Save", "Upload"] {
		// Create a throw-away shape to measure the width.
		let dummy_shape = render_text(item, 0, 0, Colour::WHITE);
		menu_width = menu_width.max(dummy_shape.width() + text_pad * 2);
		menu_height += dummy_shape.height() + text_pad * 2;
	}
	
	// Now we can compute the top-left corner!
	let menu_x = (screen_bounds.width() - menu_width) / 2;
	let menu_y = (screen_bounds.height() - menu_height) / 2;
	// Restart from the top so we can start rendering the text for real.
	menu_height = 0;
	let mut text_shapes = vec![];
	for item in &["File", "New", "Open", "Save", "Upload"] {
		let text_shape = render_text(item, menu_x + text_pad, menu_y + menu_height + text_pad, Colour::WHITE);
		menu_height += text_shape.height() + text_pad * 2;
		text_shapes.push(text_shape);
	}

	compose! [
		Quad::solid(menu_x, menu_y, menu_x + menu_width, menu_y + menu_height, Colour::DARK_BLUE),
		text_shapes
	]
}
```

Digesting this for a second, let's notice we're generating the same text shapes twice:

1. The first shape has its size measured, then is discarded completely. 
2. The second shape has its position set, then gets returned for drawing.

This looks very similar to the move we made before; something wants to be computed earlier, and it's trying to separate out. Specifically, the _text layout loop_ is trying to separate upwards into a pre-calculation, and the _centring logic_ is trying to separate downwards into a post-process.

This should actually make a lot of sense if you think about it! This exactly mirrors the way that most box models or layout engines work; they lay out the contents of children _first_ in a position-invariant way. Once they know what the child looks like, the child can be moved to wherever it belongs in the interface. We've discovered this from first principles by following the dependencies.

So, let's return to our original menu logic. Let's treat this as our "child layout" process - we'll create the entire menu relative to `0, 0`, like it's in "menu space". Then, we can calculate where the pre-laid-out menu needs to go, and move it there.

And it turns out, one of our previous moves helps us do this _super easily_. Since shapes are data instead of immediate draw calls, we can drag the shapes around however we need to, before returning them to be drawn.

```rust
fn render_ui(
	mut ctx: UiContext,
	screen_bounds: PxRect
) -> impl Paint {
	let text_pad = 4;
	// Render the menu in "menu space".
	// Since shapes are just data, we can safely do this
	// without anything showing up on-screen as a side effect.
	let mut menu_width = 0;
	let mut menu_height = 0;
	let mut text_shapes = vec![];
	for item in &["File", "New", "Open", "Save", "Upload"] {
		let text_shape = render_text(item, text_pad, menu_height + text_pad, Colour::WHITE);
		menu_width = menu_width.max(text_shape.width() + text_pad * 2);
		menu_height += text_shape.height() + text_pad * 2;
		text_shapes.push(text_shape);
	}
	let menu_shapes = compose! [
		Quad::solid(0, 0, menu_width, menu_height, Colour::DARK_BLUE),
		text_shapes
	];
	
	// Work out where the top-left of the menu belongs,
	// then offset the shapes so the top-left lands there.
	let menu_x = (screen_bounds.width() - menu_width) / 2;
	let menu_y = (screen_bounds.height() - menu_height) / 2;
	let menu_shapes = offset(menu_shapes, menu_x, menu_y);
	
	menu_shapes // The final centred menu shapes are returned.
}
```

In one motion, we've landed on the core idea behind layout in Perfection: you can operate on UI that's already been "rendered". That isn't possible in a naive renderer, because you emit a draw instruction and it's committed to the frame right away, so computing information about something is coupled with showing it on the screen. 

There are two very specific reasons why this motion is interesting.

## The golden unlock

Most systems - ours included - end up with ways to represent UI as a pile of atomic "building blocks", represented in data and able to be passed around, operated upon and queried.

However, notice something critical: our shape "atoms" are output-side - the by-products of a layout evaluation that has already happened. Our shapes have known positions and sizes.

Most UI abstractions (the DOM, Clay's box model, etc) are input-side - a raw non-evaluated description of how something _will_ be laid out. HTML or Clay elements can have styling rules on them, but they require some other fixed subsystem to evaluate them _for us_ before we can learn anything about them. As an example, asking for a particular DOM element's width may force the layout engine to re-run to give you the answer.

What we've shown from first principles then, is that output-side versus input-side is actually a _choice you can make_. There is no fact of the universe forcing you to _describe_ layouts in order to compute them. You can just compute them and operate on the results after the fact - **layout as the middle of a computation, not the terminal of one.**

That's the golden unlock of this system: the moment you can write layout as a plain computation, it follows all the normal rules of plain code:

- It directly encodes what the computer will _definitely_ do, meaning no hidden performance cliffs from a dynamic subsystem's decision - the procedure is fixed.
- It can be composed together from reusable sub-procedures, meaning you're free to build abstractions as you find convenient, or abandon them as they inconvenience you.
- It has no restrictions on what data or state will be accepted as input, meaning you can base layouts on _any_ positions, sizes, or other metrics you have access to.
- It has no restrictions on what can consume it, meaning you can even perform operations on whole layouts at a time; for example, interpolating rectangles between two disjoint layouts as a way to animate across a reflow.

This is the "layout and placement data flow" that I envisioned while writing [the 2023 blog post from before.](https://fluff.blog/2023/03/16/ui-layout-hypotheses.html) It is the common core of every further single-frame layout paradigm you would want to write. Pretty neat, right?

## A closed algebra of shapes

Now for the second reason this is interesting; there's something about the _way_ we operated on the whole menu UI at once - moving the layout and composition into the centre - that points to a really nice structural truth about operations on shapes.

I'll demonstrate by pushing the code example a little further. First things first, we can compose a new `align` utility from a bunch of the offset logic we just wrote:

```rust
fn render_ui(
	mut ctx: UiContext,
	screen_bounds: PxRect
) -> impl Paint {
	let text_pad = 4;
	let mut menu_width = 0;
	let mut menu_height = 0;
	let mut text_shapes = vec![];
	for item in &["File", "New", "Open", "Save", "Upload"] {
		let text_shape = render_text(item, text_pad, menu_height + text_pad, Colour::WHITE);
		menu_width = menu_width.max(text_shape.width() + text_pad * 2);
		menu_height += text_shape.height() + text_pad * 2;
		text_shapes.push(text_shape);
	}
	let menu_shapes = compose! [
		Quad::solid(0, 0, menu_width, menu_height, Colour::DARK_BLUE),
		text_shapes
	];
	
	// Pull out the offset/alignment logic into a slightly more
	// general utility. Some(0.5) means "align around the 50% mark".
	// We could pass Some(0.0) to start-align, Some(1.0) to end-align,
	// or None to leave that axis alone.
	let menu_shapes = align(menu_shapes, screen_bounds, Some(0.5), Some(0.5));
	
	menu_shapes
}
```

We can also now revisit our text layout logic, too. Instead of hardcoding the placement of each text shape in the list as we go, let's try and reformulate it as a series of small operations operating on the text shapes themselves.

It's a little more verbose for now, but it will be worth it:

```rust
fn render_ui(
	mut ctx: UiContext,
	screen_bounds: PxRect
) -> impl Paint {
	let text_pad = 4;
	
	// Create the text shapes.
	let mut text_shapes = vec![];
	for item in &["File", "New", "Open", "Save", "Upload"] {
		text_shapes.push(render_text(item, 0, 0, Colour::WHITE));
	}
	// Apply the offset that comes from the text padding.
	let mut padded_shapes = vec![];
	for shape in text_shapes.iter() {
		padded_shapes.push(offset(shape, text_pad, text_pad));
	}
	// Stack them up vertically.
	let mut y = 0;
	let mut stacked_shapes = vec![];
	for shape in padded_shapes.iter() {
		stacked_shapes.push(offset(shape, 0, y));
		y += shape.height() + text_pad * 2;
	}
	// Compute the size of the menu.
	let (mut menu_width, mut menu_height) = (0, 0);
	for shape in stacked_shapes.iter() {
		menu_width = menu_width.max(shape.max_x);
		menu_height = menu_height.max(shape.max_y);
	}
	menu_width += text_pad;
	menu_height += text_pad;
	
	let menu_shapes = compose! [
		Quad::solid(0, 0, menu_width, menu_height, Colour::DARK_BLUE),
		stacked_shapes
	];
	let menu_shapes = align(menu_shapes, screen_bounds, Some(0.5), Some(0.5));
	
	menu_shapes
}
```

Notice a few related facts about the direction our code has been heading up to this point:

- Our existing `align()` and `offset()` usages, in addition to the `padded_shapes` case we've just uncovered, are all use cases for applying single-shape operations to whole **groups of shapes** all at once.
- The `stacked_shapes` code looks like the perfect candidate for a specialised `vstack()` abstraction, one that stacks **nested groups of shapes** on top of each other.
- The `menu_width` / `menu_height` code is begging for a `bounds_of()` utility to synthesise what the bounding box of a **group of shapes** should be.

The latent fact we're hitting on here is that - at least conceptually - a group of shapes is _itself_ a shape. Our `compose!` abstraction already leans this way; passing in lists to `compose!` is not just a convenience, but a reflection that our shape algebra wants to be _closed_ and _easily composable_.

Being specific, groups of shapes satisfy a few key criteria:

- A group is a pure composition of other sub-shapes, meaning all of its contents can be uniformly operated on as shapes. (This is why you can `compose!` them, for example).
- You can draw useful* bounds around the group, meaning the group has a single source of truth for how it's measured and how it takes part in layout. (I'll get back to that asterisk.)

So, just like we can manipulate shapes after they're drawn, we should be able to manipulate groups of shapes too. That's the formal motivation for adding `offset()`, `vstack()` and `bounds_of()` to simplify this code.

Let's implement those now.

```rust
fn render_ui(
	mut ctx: UiContext,
	screen_bounds: PxRect
) -> impl Paint {
	let text_pad = 4;
	
	// Create the text shapes.
	let mut text_shapes = vec![];
	for item in &["File", "New", "Open", "Save", "Upload"] {
		// No need for text to be positioned if we can assume
		// it is rendered at 0, 0 - it can be implied
		text_shapes.push(render_text(item, Colour::WHITE));
	}
	// Apply the offset that comes from the text padding.
	let padded_shapes = offset(text_shapes, text_pad, text_pad);
	// Stack them up vertically.
	let stacked_shapes = vstack(padded_shapes, text_pad * 2);
	// Compute the size of the menu.
	let mut menu_bounds = bounds_of(stacked_shapes);
	menu_bounds.min_x -= text_pad;
	menu_bounds.min_y -= text_pad;
	menu_bounds.max_x += text_pad;
	menu_bounds.max_y += text_pad;
	
	let menu_shapes = compose! [
		// Construct our background directly from the menu's size.
		Quad::solid(menu_bounds, Colour::DARK_BLUE),
		stacked_shapes
	];
	let menu_shapes = align(menu_shapes, screen_bounds, Some(0.5), Some(0.5));
	
	menu_shapes
}
```

Greatly simplified! We're almost at a point where you could call this code "clean". 

Thanks to that golden unlock, we've built a straightforward higher-order abstraction where you can compose new shapes by combining others, and where those groups are nouns that behave just like any primitive shape would.

When combined with our immediate mode caching scheme (which is compatible with this whole algebra), and when combined with `compose!` (which can flatten everything down to a single ordered list to be drawn), you end up with a combinatorially rich and uniquely flexible model of UI. If you need to do a specific manoeuvre, no matter where you are or what you're working on, you can just do it.

Everything is data, everything is computation, everything is composition.

## Now, about that asterisk...

I promised you I'd get back to it before we're done. Remember this assertion about groups of shapes?

> You can draw useful* bounds around the group

If you read [my 2024 post about typographic bounding boxes](https://fluff.blog/2024/07/24/understanding-roblox-typography.html), you might have detected where things were going. The slightly messy truth is that _there isn't one single "useful" way to draw bounds around a group!_

Consider the example of a play button in a video preview UI. If you're thinking like a programmer, you might take the vertices of the triangle, find the rectangle that tightly fits around all three vertices, and call it a day. Bounding box achieved, job done, time to pass it onwards to the layout functions.

Your designer will wince. For _those_ shapes, you generally want to use a _bounding circle_ as your alignment guide, because it will look more _optically aligned_ than the mathematical bounding box will. (Ask your friendly neighbourhood designer about it.)

So, maybe you would actually draw a slightly larger, _mathematically off-centre_ bounding box for it to ensure that the bounding circle is neatly aligned with the rest of the UI, even if the vertices have slightly biased x-coordinates.

But that's the _easy_ case. The hard case - as I alluded to - is typography. In that case, your bounding boxes might not even bound the painted "ink" of the shapes at all. It's good typographic practice to align lines of text using various _baselines_; most commonly, the alphabetic baseline, the x-height baseline, or the cap-height baseline. (Though, alternatives are frequently used for Devanagari or ideographic text.)

Oftentimes, the bounding box formed between those baselines is _substantially smaller_ than the quads you're drawing to show the glyphs, and it doesn't even necessarily relate to anything geometric or mathematical _at all_ - it's visually tuned by a font designer in a studio by hand. Disrespecting that bounding box could mean things like a centred text label shifting vertically whenever a "p" or "b" shows up in the text.

This all points towards a surprisingly rarely implemented idea: you need the latitude to decouple rendered geometry **completely** from the bounding boxes used for layout. Whether it's for optically aligning non-rectangular shapes, or for keeping Latin and Devanagari spans of text aligned with each other, there are many, _many_ occurrences of the two perspectives not agreeing.

Unfortunately, most UI systems today pretty much just special case text layout, forcing you into the framework's opinion on how text should be aligned (usually relative to the em box, which has no guaranteed relation to anything useful), and leaving you high and dry for anything requiring optical alignment.

In our system... well, text is just a group of shapes. So, `render_text()` can just return a group of laid-out glyphs or rasterised quads, and _declare_ where the typographically useful bounds are, enhancing the appearance of text everywhere throughout our UI system practically for free - no special case needed.

In Perfection, that's done by calling `.bounded_by()` on a shape (or composition of shapes), which attaches a bounding box that's independent of any rendered content:

```rust
let menu_shapes = compose! [
	Quad::solid(menu_bounds, Colour::DARK_BLUE),
	stacked_shapes
].bounded_by(menu_bounds); // Align these shapes using the menu's outer bounds.
```

There's another benefit to decoupling the layout bounds from the rendered shapes: you can use it for _more_ than just optical alignments. Did you notice all that `text_pad` in our code?

```rust
// Pretend that our text has a padded bounding box.
let padded_shapes = offset(text_shapes, text_pad, text_pad);
// Stack them up vertically, manually maintaining the illusion
// that the bounding boxes are padded by giving double space.
let stacked_shapes = vstack(padded_shapes, text_pad * 2);
// Manually add the padding to the menu bounds so that the
// quad incorporates it.
let mut menu_bounds = bounds_of(stacked_shapes);
menu_bounds.min_x -= text_pad;
menu_bounds.min_y -= text_pad;
menu_bounds.max_x += text_pad;
menu_bounds.max_y += text_pad;
```

Instead of pretending to pad the bounding box of the text... we can recognise that text is a group of shapes, which has a bounding box attached. So, you can just _pad the bounds_:

```rust
fn render_ui(
	mut ctx: UiContext,
	screen_bounds: PxRect
) -> impl Paint {
	let text_pad = 4;
	let mut text_shapes = vec![];
	for item in &["File", "New", "Open", "Save", "Upload"] {
		let text = render_text(item, Colour::WHITE);
		// Pad out the text's bounding box, so the group
		// builds in space for the text to sit in.
		text_shapes.push(pad(text, text_pad));
	}
	// Now the text can be stacked directly.
	let stacked_shapes = vstack(text_shapes, 0);
	// And the menu bounds can be used unmodified!
	let mut menu_bounds = bounds_of(stacked_shapes);
	
	let menu_shapes = compose! [
		Quad::solid(menu_bounds, Colour::DARK_BLUE),
		stacked_shapes
	].bounded_by(menu_bounds);
	let menu_shapes = align(menu_shapes, screen_bounds, Some(0.5), Some(0.5));
	
	menu_shapes
}
```

Bingo. Text padding has a single source of truth, and the rest of the function is better for it: a whole `offset()` loop skipped, a more semantically truthful `vstack()`, and a dead simple `bounds_of()` query for the menu background. 

With that, we've arrived at the end state of our whole journey through presentation.

Let's look back at where we came from near the start of the chapter. If you were drawing UI as you went - like we discussed earlier - this is about as far as you might get while holding to a realistic standard of code quality:

```rust
fn render_ui_old(mut ctx: UiContext) {
	let text_pad = 4;
	let item_height = 16 + text_pad * 2;
	let menu_width = 104;
	let menu_height = 120;
	draw_quad(0, 0, menu_width, menu_height, Colour::DARK_BLUE);
	
	let mut top_of_item = 0;
	for item in &["File", "New", "Open", "Save", "Upload"] {
		draw_text(item, text_pad, top_of_item + text_pad, Colour::WHITE);
		top_of_item += item_height;
	}
}
```

But, with everything we've built up, we can beat it on length while making it semantically even *more* powerful. Thanks to our efforts, we have a concise, consistent vocabulary for expressing our intended layout moves. Plus, our move to make everything into plain computations on data means we get to use Rust's powerful iterators to go fully declarative. And hey - we're not even hardcoding menu dimensions anymore, and it's trivial to re-align if we so choose.

```rust
fn render_ui_new(mut ctx: UiContext) -> impl Paint {
	let items = ["File", "New", "Open", "Save", "Upload"].into_iter()
		.map(|item| pad(render_text(item, Colour::WHITE), 4))
		.collect_vec();
	let items = vstack(items, 0);
	let bounds = bounds_of(items);
	compose! [
		Quad::solid(bounds, Colour::DARK_BLUE),
		items
	].bounded_by(bounds)
}
```

How's that for a semantically rich yet syntax-efficient system? :)

---

## A teaser for interaction handling

Now that we're presenting the user with shapes on the screen, the next natural step is to get responses _back_ from the user and make the UI interactive.

To start, let's whip up a button with our freshly discovered presentation primitives. The button will have different colours when it's being hovered or pressed down.

The code should be readable to you by now:

```rust
fn render_button(
	mut ctx: UiContext,
	label: String
) -> impl Paint {
	// We'll wire this up in a second.
	let state = PointerState::Rest;
	
	// Using the hover/press state, decide on button colours.
	let (bg_colour, fg_colour) = match state {
		PointerState::Rest => (Colour::DARK_BLUE, Colour::WHITE),
		PointerState::Hover => (Colour::DARK_BLUE, Colour::YELLOW),
		PointerState::Press => (Colour::WHITE, Colour::BLACK)
	};
	
	// Emit the button, ready to be included elsewhere.
	let text = render_text(label, fg_colour);
	let button_bounds = pad(text.bounds(), 4);
	
	compose! [
		Quad::solid(button_bounds, bg_colour),
		text
	].bounded_by(button_bounds)
}
```

Let's start by trying to make these hover effects work. We'll accept a `mouse_pressed: bool` directly, and let's assume we have access to a `mouse_position: Vec2` argument, to compute whether the mouse is in bounds for hovering. 

This requires a little bit of rearrangement, but you can in theory make it work. Here's what the rearrangement looks like:

```rust
fn render_button(
	mut ctx: UiContext,
	mouse_position: Vec2,
	mouse_pressed: bool,
	label: String
) -> impl Paint {
	// Render the text first so we can measure the button bounds
	// for hit testing. Punt the text colour; it has separate dependencies.
	let text = render_text(label);
	let button_bounds = pad(text.bounds(), 4);

	// Don't respond to the cursor if it's somewhere else.
	// Otherwise, set the state based on what the mouse is doing.
	let state = 
		if !button_bounds.contains(mouse_position) { PointerState::Rest }
		else if mouse_pressed { PointerState::Press }
		else { PointerState::Hover };

	// This is where we punted the colour computation to.
	// We now compute it separately from text layout.
	let (bg_colour, fg_colour) = match state {
		PointerState::Rest => (Colour::DARK_BLUE, Colour::WHITE),
		PointerState::Hover => (Colour::DARK_BLUE, Colour::YELLOW),
		PointerState::Press => (Colour::WHITE, Colour::BLACK)
	};
	let text = recolour(text, fg_colour); // New operation!
	
	compose! [
		Quad::solid(button_bounds, bg_colour),
		text
	].bounded_by(button_bounds)
}
```

Okay, so this _seems_ reasonable, except... what mouse position do you pass to this function? 

Remember - back when we were dealing with layout, we posited that the button's layout doesn't depend on where it's positioned. That meant that we intentionally constructed the button outside of the normal coordinate system, with the understanding that the button would be moved into place by the caller at some point.

That means the caller would need to somehow pass in a mouse position in this button's own coordinate system. But the caller won't even know where that bounding box ends up _itself_ - the final position of things isn't fixed until the end of `render_ui()` as a whole.

Just like in Chapter 2, we have a cyclic dependency - except this time, the way the component renders is cyclically dependent on _the whole render process_.

Join me in [part 3 of this blog post series](https://fluff.blog/2026/09/15/the-ui-frame-loop-and-why-input-handling-is-ui.html), and we'll start unpicking exactly how our UI system _ought to_ handle inputs, and draw some surprising parallels to game engine systems you might already have built yourself.