---
layout: post
title: I made my perfect UI library
image:
  path: /assets/posts/i-made-my-perfect-ui-library/thumb.jpg
  width: 256
  height: 256
---

For the past decade, I've been fighting against _clever_ UI frameworks. Ones that try and be helpful, provide features and conveniences, build in all the features you need. That is to say, ones that tie your hands behind your back the moment you escape their mental model.

Perhaps it's the panel you want to animate from one part of the screen to the other. That "dead-simple" layout engine can't handle it because it's a change of parent.

Perhaps it's the button you _just_ want to place a few pixels over. The "effortless" box model complains because that space was taken by another box.

Or perhaps, splitting a dynamically sized container into fractions. Or, computing multiple layouts and picking them based on your own fitness function. Or, tweening to an element's automatic size. _Or, like, just properly typographically centring text in a button - shouldn't that be simple?_ I could go on and on and on.

Pardon my colourful language, but it's been a decade, and I'm _so fucking done_ with convenience. Even my own "convenient" UI frameworks I've built with my own hands have failed me in exactly the same way. (But I'm happy others loved them.)

So all I ask for in this cursed programming world, is a thin, unclever, unopinionated, first-principles library that can build _whatever_ I want, and _doesn't fight me back_.

I built that library for myself, and I'll show you here how it's put together.

---
![Clouds at dawn.](/assets/posts/i-made-my-perfect-ui-library/dawn-header.jpg)
## Chapter 0: Thesis

You know what they say: perfection is achieved, not when there is nothing more to add, but when there is nothing left to take away. So the basis for my new UI library, "Perfection", revolved around a reductive core philosophy: **everything is data, everything is computation, everything is composition.**

The main UI entry point would return a single list of quads. That list of quads would be rendered first element first, last element last - literal Painter's algorithm. Everything between your application state and what textured quads show up on the screen would be a pipeline you control. Freedom from the tyranny of conveniences at long last.

Want to render a string of text? You call into a function which transforms the string into textured, laid-out quads. Want to stack those strings of text? Pass those rendered quads into a function which puts each group of quads on top of the other. Centre align everything? Pass the whole assembly of quads into a function that moves all of them to the centre of the screen's bounding box.

Everything in the UI code would have the same shape; every primitive would be a transformation from input data to output data. Layout reuse and parameterisation looks exactly the same as UI reuse and parameterisation.

Most importantly, it would satisfy [some constraints I first articulated a few years back](https://fluff.blog/2023/03/16/ui-layout-hypotheses.html): it'd be a unidirectional data flow from state, through to layout, through to deciding where quads were placed, and finally through to rendering - no feedback loops, and no forced hierarchies or hardcoded layout inputs like you'd get with a box model.

But it sounds insane, right? For the past three decades, we adopted DOMs, stylesheets, box models and flex layouts for a reason, right? After all, I remembered watching the [introduction video for Clay](https://www.youtube.com/watch?v=DYWTw19_8r4) - a popular UI library. It literally starts with examples of how emitting plain draw instructions falls apart quickly.

![The Clay presentation showing draw instruction bugs.](/assets/posts/i-made-my-perfect-ui-library/clay.jpg)

This introduction mentions you'd have to adjust everything by hand whenever you input any new data or needed to fit new children - it'd be a bunch of micromanagement for limited gain. But the jump to "flexible boxes" felt ill-motivated; instead of trying some of the fixes, it felt like a post-hoc justification for a particular mental model.

Furthermore, people like Casey Muratori - whatever you think of them - can seemingly produce whole game UIs in side-effectful per-frame renderer code without even creating any UI objects or layout structures. 

It made me wonder if most people had _truly, seriously_ tried to proceed without jumping straight for objectful UI frameworks. Perhaps most of us just assumed that all these indirections were doing us some service, and that they were an established natural solution.

I had a suspicion they weren't, so I threw them out.

And you know what? After spending a few days prototyping it out roughly, I got to a point where I could write code every bit as ergonomic as modern reactive code, with the same compositional beauty and straightforward reasoning about state. It fully satisfies the type checker, it isn't coloured by any particularly crazy syntax, and there's no large data structures being constructed anywhere.

Here's a sneak peek of some of the syntax you'll see later. I promise you, it's not that bad :)

```rust
// Written in Perfection, my new immediate-mode UI library

fn render_item(
	mut ctx: UiContext,
	label: &str,
	on_click: impl FnOnce()
) -> UiBounded {
	let region = ctx.key("hit_region").hit_region();
	if region.was_clicked() { on_click(); }

	let colour = match region.state() {
		PointerState::Press => COLOUR_ACTIVE,
		PointerState::Hover => COLOUR_HOT,
		PointerState::Rest => COLOUR_REST
	};
	let text = render_text(ctx.key("text"), label, SCALE as i32, colour);
	let text_bounds = text.bounds();
	let hit_bounds = pad(text_bounds, 4.0 * SCALE);
	compose! [
		text,
		region.place(hit_bounds)
	].bounded_by(text_bounds)
}

fn render_pause_menu(mut ctx: UiContext) -> impl Paint {
	profiler_span!("ui_menu");
	let screen_bounds = ctx.env::<ScreenBoundsCtx>();
	let actions = ctx.env::<GameLoopActions>();

	let items = vec![
		render_item(ctx.key("resume"), "Resume", || actions.set_paused(false)),
		render_item(ctx.key("exit"), "Exit", || actions.close_game())
	];
	let items = vstack(items, 8.0 * SCALE);
	let items = {
		let items_bounds = bounds_of(&items);
		items.into_iter()
			.map(|item| align(item, items_bounds, (Some(0.5), None)))
			.collect::<Vec<_>>()
	};
	let items = align(items, screen_bounds, (Some(0.5), Some(0.5)));
	let items = snap_pixel(items);

	compose! [
		Quad::solid(screen_bounds, Colour::new(0, 0, 0, 225)),
		items
	]
}
```

This is my perfect UI library. It outsources almost everything to abstractions above it. There's nothing left to take away.

Here's how I got there.

----
![Clouds in the morning.](/assets/posts/i-made-my-perfect-ui-library/morning-header.jpg)
## Chapter 1: State management

### But wait, you can't do all that every frame!

It's the most common argument against immediate-mode UI; that the performance is irredeemable. It's only suitable for game engines, it must magically drain battery more, it's some renderer hack, and all that. If you're building "real UI", you just need a traditional retained-mode UI framework, like a reactive signals library. At least, people say.

Let's test that theory. First, let's go over how each approach caches data under the hood.

Reactive signals are simple enough; they're objects that are constructed once when the UI function is run, and then continue existing to respond to events thereafter. The library can store caches and book-keeping internally on the object:

```lua
-- Written in Fusion, my old reactive signals framework

-- How Fusion constructs a simple reactive signal object.
-- Slightly simplified. The object stores everything.
local function Value(initialValue)
	local self = setmetatable(
		{
			createdAt = os.clock(),
			dependentSet = {},
			lastChange = os.clock(),
			validity = "valid",
			value = initialValue
		}, 
		METATABLE
	)
	return self
end
```

On the other hand, in immediate mode UI, the UI code runs all the time, so it's not a stable place to store or remember things. Instead, the main loop of the program manually carries over a "context" object from one frame to the next:

```rust
// Written in Perfection, my new immediate-mode UI library

let ctx = UiContext::new();

loop {
	ctx.next_frame();
	render_ui(ctx);
}
```

Then, each frame, you can refer to stable named storage locations. In the case of my library, Perfection, that's done by building up a "path" to the location using `ctx.key()`.

```rust
// Written in Perfection, my new immediate-mode UI library

fn render_ui(mut ctx: UiContext) {
	// We use .key() to name the storage location first.
	// Then, we use .once() to run a callback just once,
	// and store the value it returns.
	let my_number = ctx.key("my_thing").once(|| {
		print!("I only run once!");
		return 42;
	});
	
	dbg!(my_number) // prints: 42
}
```

`.key()` actually returns a new "nested" `UiContext`, so you can give each sub-function its own clean named `UiContext`. 

```rust
// Written in Perfection, my new immediate-mode UI library

fn render_pause_menu(mut ctx: UiContext) {
	// path: /option_resume/
	render_option(ctx.key("option_resume"));
	render_option(ctx.key("option_exit"));
}

fn render_option(mut ctx: UiContext) {
	// path: /option_resume/button/
	render_button(ctx.key("button"));
}

fn render_button(mut ctx: UiContext) {
    // path: /option_resume/button/hit_region/
	render_hit_region(ctx.key("hit_region")); 
}
```

Internally in the `UiContext`, you have two maps from keys to stored values. On frame N, you can write these named locations into one of the maps. Then, on frame N + 1, you flip the maps over - now you can refer to last frame's map while building up next frame's map, copying over anything you want to retain, and leaving out what you want to purge.

These are what I call "flip caches". [I discuss them more in my January post on them](https://fluff.blog/2026/01/07/flip-caches-and-information-loss.html), as well as some extra background on a concept called "information loss" which motivates the choice of reactive signals and immediate mode as the starting points here. (It's not central to anything in this post, happily.)

So cool, each approach has a way of caching state if it needs to. That's important to establish; immediate mode UI does not _have_ to throw everything away every frame!

### Two sides of one coin

So, back to this claim:

> If you're building "real UI", you just need something retained, like a reactive signals library.

Having built both a reactive signal library and now an immediate mode one, **I disagree.** Both are perfectly acceptable and on even footing with each other.

More foundationally, both approaches are two sides of the exact same coin, assuming both approaches are implemented properly and professionally. Both approaches try to treat UI as a function of state, and in fact, both try and keep your logic stateless, divorced of history. One is mechanically translatable to the other, and the distinction is entirely in the caching approach.

Let's expose this fact by writing the same example code snippet in both.

Every derived reactive signal you write contains little islands of immediate-mode logic. The computation is always stateless and you expect it to always hold true. The framework just chooses to skip the computation when the inputs haven't changed.

We do this because it makes it easy to reason about the chain of events that lead to a final value, without having to hold the whole history of the UI and all the state transitions in your head.

```lua
-- Written in Fusion, my old reactive signals framework

local itemName = scope:Value("Carrots")
local itemPrice = scope:Value(150)

local message = scope:Computed(function(use)
	-- Collect inputs...
	local name = use(itemName)
	local price = use(itemPrice)
	-- Run this when the inputs notify you of a change.
	return "Buy " .. name .. " for " .. price .. " coins"
end)

print(Fusion.peek(message)) -- prints: Buy Carrots for 150 coins
```

The way these reactive frameworks operate under the hood, is that they wire up objects to each other as you `use()` them. Those pathways are used to communicate change events, which are the things that cause the computation callback to re-run. As a result, you can avoid running the computation any more frequently than that.

As it turns out, an immediate-mode library can express exactly the same kind of cached computation. Look, you can even pattern-match it with your eyes.

```rust
// Written in Perfection, my new immediate-mode UI library

let item_name = "Carrots";
let item_price = 150;

// This is a utility function that makes caching
// nice and pleasant to write for us, just like Fusion.
let message = ctx.expensive(|get| {
	// Collect inputs...
	let name = get(item_name);
	let price = get(item_price);
	// Run this when the inputs appear to have changed.
	move |ctx| format!("Buy {name} for {price} coins")
});

dbg!(message) // prints: Buy Carrots for 150 coins
```

There are a few nuances worth digging into here, including a few things that will probably surprise the uninitiated - the first of which being that the inputs (`item_name` and `item_price`) are _not_ objects, but are just plain data.

The reason why reactive frameworks wrap everything in objects, is to form that change-notification network we discussed before - a "reactive graph". That's how you know when to re-run the expensive computation.

Perfection doesn't do this. Instead, it splits expensive computations into two halves; the first callback is called every frame to collect inputs with `get()` (allowing for dynamic/conditional gets), then the expensive part is returned as a _second_ callback which may or may not be executed based on what you got.

```rust
// Expanded version of the above code snippet.
// You wouldn't write it this way for realsies,
// but it's useful for explanation.

// The callback for the "cheap" part.
// Runs every time the UI is evaluated.
let cheap_part = |get| {
	// Collecting all the inputs is super cheap.
	// If we get the same number of inputs, and they have
	// the same values, then assuming our expensive
	// computation is pure and deterministic, it must be
	// identical too.
	let name = get(item_name);
	let price = get(item_price);
	
	// The callback for the "expensive" part.
	// If all the get() stuff is equal, we don't run it,
	// and instead fetch the last result from memory.
	let expensive_part = move |ctx| {
		format!("Buy {name} for {price} coins")
	};
	
	expensive_part // Always returned.
};

let message = ctx.expensive(cheap_part);
```

Rust makes this very cheap to do, and presumably you'd only be caching things that are so expensive that any overhead from this split would be atomically small in comparison. You save the overhead from not doing object allocations, too.

So yeah, you don't need a reactive graph or a bunch of objects in order to get something that feels _exactly like_ lazy reactive signals. Immediate mode and reactive signals are implemented differently, but capable of expressing the exact same concepts.

I would actually argue on syntax alone that immediate mode makes these ideas much nicer to express, since it solves the biggest problems of reactive signals: since there are no objects, there's no lifetime wrangling, and reactivity isn't a viral annotation that spreads through your codebase like wildfire - it's all plain inputs and outputs, no object wrappers. It's why the example code I showed earlier looks so clean.

There's a more subtle and under-discussed aspect, though.

### UI lifecycle and reconstruction

A common consequence of reactive frameworks relying on constructed, region-allocated signal objects, is that re-running UI code means re-constructing all the objects. For example, consider this _correct_ Fusion code:

```lua
-- Written in Fusion, my old reactive signals framework

-- Create a timer that smoothly increases from
-- the start of the program.
local timer = scope:Value(0)
everyFrame(function(dt)
	timer:set(Fusion.peek(timer) + dt)
end)

-- Drive a sine wave using the timer.
local wobble = scope:Computed(function(use)
	return math.sin(use(timer))
end)

-- Define a physically simulated follower that
-- will lag behind the sine wave with some inertia.
-- (i.e. this will depend on the object's own history)
local laggard = scope:Spring(wobble)
```

The `Value`, `Computed` and `Spring` objects are constructed once, which is good - the `Spring` in particular is initialised to `wobble` at first, but then over time can refer to its own internal history to keep track of its own position and velocity.

Now, consider why this _very similar_ Fusion code does _not_ work:

```lua
-- Written in Fusion, my old reactive signals framework

-- Same as above.
local timer = scope:Value(0)
everyFrame(function(dt)
	timer:set(Fusion.peek(timer) + dt)
end)

-- Same as above.
local wobble = scope:Computed(function(use)
	return math.sin(use(timer))
end)

-- All we did was wrap this in a Computed.
local laggard = scope:Computed(function(use)
	-- Every time `wobble` changes, this internal Spring
	-- object is re-created, so it has no memory of where
	-- it has been. So, it doesn't animate!
	return scope:Spring(use(wobble))
end)
```

This is a perfect illustration of how Fusion's `Computed` objects break [Tennent's Correspondence Principle](https://gafter.blogspot.com/2006/08/tennents-correspondence-principle-and.html). By wrapping an expression in a cache, we unintuitively cause it to wipe all knowledge of its own memory. This is a direct artifact of tying all history state to the lifetime of particular objects. At a deeper level, it's a conflation between the lifecycle of objects in the UI, and the lifecycle of the function that renders them - Fusion is only designed for UI functions to run once.

Of course, Perfection - being an immediate mode framework - is designed to run its UI functions more than once. This forces a decoupling of those two lifecycles, so instead of storing state in fragile objects, it _has_ to be stored at a stable named location. So this example fully works, with or without the `expensive` cache - it's totally transparent.

```rust
// Written in Perfection, my new immediate-mode UI library

// No need to manage this ourselves,
// we are called every frame.
let timer = current_frame.timer;

// Same wobble logic - just a sine wave.
let wobble = timer.sin();

// Option 1: without the cache
// Name a storage location "my_spring",
// and save a spring there.
let laggard = ctx.key("my_spring").spring(wobble);

// Option 2: with the cache
// It works exactly the same, because storage
// is decoupled completely from runs.
let laggard = ctx.expensive(|get| {
	let wobble = get(wobble);
	move |ctx| ctx.key("my_spring").spring(wobble)
});
```

(It's worth mentioning that reactive signal frameworks could choose to do this too. As mentioned before, they're two sides of the same coin. They just don't, because it's not the norm to name stable storage locations - they just use the object wrappers as storage.)

This property makes Perfection code a lot more fearless - instead of being precious about re-runs as a matter of correctness, it's just a matter of performance. That frees you up to think about bigger issues, like...

### The memory tragedy of the commons

Ever since the 90s, computation has outpaced the speed of memory accesses. To smooth over this pressure, modern CPUs tend to have caches that are physically closer to the chip. Any cold memory accesses that miss the cache take on the order of thousands of times longer than a "usual" ALU operation operating directly on available registers.

Of course, that's a slight simplification - there's actually multiple caches (L1, L2, L3...) at various distances from the chip. Closer caches like L1 are much faster, but also much smaller.

As a result of all this, you _really_ want to be economical with CPU cache lines. Anything that loads new things in from memory, by necessity, will evict older cached items - first down to lower caches like L1->L2 or L2->L3, but eventually it'll evict it entirely.

This is why caching things on modern computers isn't always a win. If the computation is sufficiently trivial, or the memory sufficiently cold, you can end up in situations where recomputing the thing from scratch is many times faster than waiting for the previously computed answer to be retrieved from memory.

That's the obvious takeaway, anyway. There's actually a more subtle angle you should consider too, which is that _even if something is locally faster to cache_, you might not want to, because _something else_ benefits from the cache more than you do. That is: the more things that do reads from memory, the more things that may be evicted from the caches. By the time you get to some _critical_ cached computation, you may end up with a fully wiped-out cache, meaning that super-important cache read takes orders of magnitude longer.

By caching more things to get *real, measurable local speedups*, the pressure imposed by the whole procedure worsens, and *raises the memory latency floor overall*. It's a tragedy of the commons. (I'll call it TOTC.)

The worst thing about this kind of TOTC is that it is basically un-attributable by normal profiling done on a per-function basis. Your function can measurably get faster, it can measurably take less frame time, whatever. But your profiler would be lying by omission to you; silently, in the background, the whole procedure quietly ticks up in overall execution time, most nefariously in functions you aren't even explicitly profiling, or which you assume take "constant time".

This TOTC is why it's so important to choose what parts of your UI you want to cache, and why it's so important for UI libraries to have thin, straightforward caching logic. It's also exactly the place where Fusion fails with its reactive signals; the common practice is to wrap all derived values in `Computed`, which carries a whole bunch of bulky caching logic and environment setup logic with it.

```lua
-- Written in Fusion, my old reactive signals framework

local itemName = scope:Value("Carrots")
local itemPrice = scope:Value(150)

-- There is no reason to cache such a simple inline
-- computation, but Fusion will cache it for us anyway.
local message = scope:Computed(function(use)
	return "Buy " .. use(itemName) .. " for " .. use(itemPrice) .. " coins"
end)
```

The only way to avoid it is to replace it with a plain closure, which still imposes some inherent overhead in Luau, which means you're still competing for resources (though fewer).

```lua
-- Written in Fusion, my old reactive signals framework

local itemName = scope:Value("Carrots")
local itemPrice = scope:Value(150)

-- You can't remove the closure, because you need
-- to keep the use() calls in case it ends up in
-- a higher-up `Computed` somewhere.
local message = function(use)
	return "Buy " .. use(itemName) .. " for " .. use(itemPrice) .. " coins"
end
```

And of course, Fusion's various UI lifecycle problems only compound this, forcing you to use `Computed` and its relatives in very particular ways to achieve correct behaviour.

By comparison, Perfection is a breath of fresh air. If you don't want to spend limited cache resources on something cheap to recompute... just recompute it. It's immediate mode. There are no viral annotations.

```rust
// Written in Perfection, my new immediate-mode UI library

let item_name = "Carrots";
let item_price = 150;
let message = format!("Buy {item_name} for {item_price} coins");
```

This gets towards one of the interesting things about the difference between reactive signals and immediate mode. The choice not to use objects in the latter isn't just me wanting to avoid OOP, nor is it a debate about syntax. It's really about a fundamental truth of code that adds these layers of indirection, and the way it behaves on the metal.

### Proof in the pudding

I would hope by this point, I've shown mechanically that the distinction between retained-mode signals and immediate-mode flip caches are negligible. Any computation that can be cached in one, can be cached in the other. Any efficiency or inefficiency follows from norms, not from an inherent limitation of the possibility space.

But if you remain unconvinced, here are some of the most performance-sensitive production projects which use immediate mode, and experience none of the conjectured issues.

#### [File Pilot](https://filepilot.tech/)

It'd be impossible not to lead with this. Using this file manager myself was a religious experience; I've never used a program so fast that hovering the buttons felt truly connected to my hand as I moved my mouse. It lets you scroll through _your whole C drive_ as if it's nothing.

It's so fast that [an internal Microsoft source claims it's the new benchmark the in-box File Explorer is being measured against.](https://xcancel.com/vkrajacic/status/2048853124211752987) That's a glowing review for a UI that's built with - supposedly - the "slower" option.

#### [Blick](https://blickeditor.com/)

This one's a new one, demoed at Better Software Conference, but it's staggeringly performant and well-put-together. Funnily enough for this post, it's even got endorsements from both [Casey Muratori](https://xcancel.com/cmuratori/status/2081778849885684020) (who coined imgui) and [Nic Barker](https://xcancel.com/nicbarkeragain/status/2081647191727493523) (who made Clay)!

[Just check out this clip of it absolutely eating up 30,000 clips for breakfast.](https://xcancel.com/Blick_Editor/status/2082456045943689595#m)

Video editors are some of the hardest pieces of software to make truly performant, so the dedication to fluidity is remarkable here. To say it's competitive with Premiere and friends, is to give Premiere way too much credit.

#### [Nomad Sculpt](https://nomadsculpt.com/)

This one's notable because it runs on mobile hardware, which gives us a reference point for a complex bit of kit that's not plugged into a wall. As it turns out; battery drain just isn't a problem if you build it well.

It's listed on [Dear Imgui's list of disclosed users](https://github.com/ocornut/imgui/wiki/Software-using-dear-imgui) and is generally a really cool app. [There's even a web demo if you're curious.](https://nomadsculpt.com/demo/)

#### [Blender](https://www.blender.org/)

Following the same train of thought, Blender has a huge presence in the 3D modelling world too, and it's shown to be perfectly capable even for people just getting into modelling without workstation-level hardware.

In fact, it has a unique blend of immediate mode with damage redrawing, showing that immediate mode can be a perfectly principled solution even if you want to retain final pixels instead of refreshing them 144 times a second. Immediate mode logic can just be about solving lifecycle problems and keeping code simple!

#### [Tracy](https://github.com/wolfpld/tracy)

Profiles are one of the most data-dense UIs in existence, so it means something that one of the most popular and most performant ones runs on Dear ImGui. It renders timelines with _millions_ of profiling zones at full framerate, completely smoothly. If ever there was proof that "running every frame" can scale incredibly well, well, here you go.

I use Tracy actively in the development of my voxel game [Cavey](https://caveygame.com/) and I have never, _ever_ looked back. It's brilliant and I wouldn't ever skip an opportunity to sing its praises.

#### [RemedyBG](https://remedybg.itch.io/remedybg)

Staying in the world of high-performance dev tools for a second; this commercial Windows debugger sells itself as a super-fast alternative to more traditional tools like the Visual Studio debugger. It refreshes nearly instantly despite being composed of dense data fields and tons of text to be truncated all the time.

----
![Clouds at midday.](/assets/posts/i-made-my-perfect-ui-library/day-header.jpg)
## Chapter 2: Presentation

### Hey, let's just draw stuff!

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

### Unmixing layout from rendering

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
		draw_text(item, text_pad, top_of_item + text_pad, Colour::WHITE);
		top_of_item += item_height;
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

### Introducing shapes

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

### Operating on shapes

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

### The golden unlock

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

### A closed algebra of shapes

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

### Now, about that asterisk...

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
	].bounded_by(bounds);
}
```

How's that for a semantically rich yet syntax-efficient system? :)

---
![Clouds at noon.](/assets/posts/i-made-my-perfect-ui-library/noon-header.jpg)
## Chapter 3: Interactions

### ...hey, just... query the mouse position?

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

### Breaking the dependency cycle

To try and move past this, we'll employ the same trick we did for those layout cycles last chapter; duplicate the logic, and figure out what wants to happen first, and what wants to happen later. That means two `render_ui()` calls for now.

First: tracking where our button ends up between the two calls. We'll add a new "hit region" to our composition to represent the interactive area, that'll flow along with all the rest of our shapes. Importantly, it'll have a stable key from our `UiContext` so we can refer to it across runs.

```rust
fn render_button(
	mut ctx: UiContext,
	label: String
) -> impl Paint {
	let text = render_text(label);
	let button_bounds = pad(text.bounds(), 4);
	
	let region = hit_region(ctx.key("region"), button_bounds);
	let (bg_colour, fg_colour) = match region.state {
		PointerState::Rest => (Colour::DARK_BLUE, Colour::WHITE),
		PointerState::Hover => (Colour::DARK_BLUE, Colour::YELLOW),
		PointerState::Press => (Colour::WHITE, Colour::BLACK)
	};
	
	let text = recolour(text, fg_colour);
	
	compose! [
		Quad::solid(button_bounds, bg_colour),
		region,
		text
	].bounded_by(button_bounds)
}
```

Now, all we need to do is a first-run of the UI which is inert and collects all the hit region positions/sizes, then we can do a second run that updates `region.state` for all of the hit regions based on where they landed in the first run:

```rust
let ctx = UiContext::new();

loop {
	ctx.next_frame();
	// Lay out all the hit regions, being careful not to cause side effects.
	let first_shapes = render_ui(ctx);
	ctx.reposition_hit_regions(first_shapes);
	// Update the states of the hit regions as-laid-out based on the mouse
	// position and pressed state.
	ctx.update_hit_region_states(mouse_position, mouse_pressed);
	// Do the final layout of the frame, ready for rendering.
	let second_shapes = render_ui(ctx);
	render_to_screen(second_shapes);
}
```

This would work! Assuming your render functions were deterministic and any side effects were kept under control, you'd be able to precisely evaluate which hit regions were hovered or pressed.

That said, rendering the same UI twice for some hover effects is pretty unfortunate. Clearly, we need the layout of the hit regions _before_ we can build the rest of this UI. 

What's _not_ clear is _how_ you would do that. Layout is a screen-wide process, so our component doesn't own it. Inputs could be blocked by other components we're not aware of that overlap us. Plus, layout and presentation are tightly woven together in a lot of cases, so it's not clear that you can even clearly make a separation between them.

So, instead of fighting against our UI system trying to rationalise a split inside of the procedure, let's instead perform a lateral move along a different dimension.

### Frames live between renders

In theory, we present the user with a rendered frame, then they move their mouse somewhere and click in response. That response is processed at the end of the frame, ready to feed forward into the next rendered frame, and into the wider game engine.

In other words, renders are transient reconciliations between frames, where the trailing edge of the past frame is closed, and the leading edge of the next frame is opened. 

So, we should be measuring how a user _responds_ to the hit regions we have shown last frame. However, what our two-pass system does in practice is set up a bunch of hit regions, then immediately measure where the cursor intersects them in a second pass. We're measuring the user's input against UI we haven't yet shown them - and in fact, haven't even finished constructing. That's where our dependency cycle is coming from.

This is one of the few places where depending on the UI's history is actually a good thing. A user being able to interact with a button they've never seen on the screen is a failure mode, but it's exactly what a history-less system like ours would offer. (Some platforms like Android go as far as to artificially sink inputs to buttons when they've _just_ appeared at a location, so this kind of observation isn't new.)

So, to break the cycle, let's set up the hit regions on frame N, then measure the user's interaction with those hit regions as we build frame N + 1. Comparing those regions with the mouse pointer's location tells us where the mouse ultimately landed on the last presented frame that a user _actually saw_.

This works well for almost all cases; for a button sitting on the screen idle, last frame's hit regions are an exact match for this frame's hit regions, making them behave identically. 

The only behaviour change comes if anything interactive is changing (e.g. dialogs appearing, buttons moving etc.) - they have to *first* render to the screen before responding to inputs on the next frame. That can be a source of visual discrepancy, but in those cases, you'd likely want to sink any interactions happening across those state changes anyway, as Android does.

Here's what the change looks like in the main loop; notice how `reposition_hit_regions` now sits next to `render_to_screen` - both are now per-frame "committed" outputs, presented to the user together for the next period of time.

```rust
let ctx = UiContext::new();

loop {
	ctx.next_frame();
	// Use the latest mouse position to reduce input latency.
	ctx.update_hit_region_states(mouse_position, mouse_pressed);
	// Render this frame, using the previous hit region positions to inform
	// what our hover and press states are.
	let shapes = render_ui(ctx);
	// Set up the hit region positions for next frame.
	ctx.reposition_hit_regions(shapes);
	// Render to screen.
	render_to_screen(shapes);
}
```

### Clicking things in place

Now that we've established whether the mouse is over our button, the natural next step is to detect clicks that occur over our button, and take action accordingly.

Our current system already gives us a lot of nice properties for free. For example, since we update hit region states globally, it's trivial to implement hit region "occlusion" where hit regions can absorb hovers that'd otherwise pass to hit regions behind them. Compared to pulling messages from an input queue _during_ rendering, a single pass over all hit regions makes it easy to reason about which hit regions sink which events, rather than having to reason about implicit nonlocal ordering of how events are pulled during rendering.

As a second example, having a stable key for every hit region _also_ makes it trivial to implement proper press/release handling, where a button only "clicks" if the press and release happens over the same region. Since regions can be tracked across frames with stable identity, it's easy to abandon a click that ends with the mouse located elsewhere.

That leaves us with one question: what do you do when someone clicks a region? Where does that get handled?

A classic imgui approach is to return the clicked state as a result of rendering a clickable object. For example, in classic imgui, a clickable button would have this shape:

```rust
// Returns true if the button has just been clicked.
if button("Hello, world") {
	do_click_action();
}
```

We actually already do this with our hover state; the only difference is that it's accessed _through_ a returned object, rather than being the returned value directly.

```rust
// ... snip ...
let region = hit_region(ctx.key("region"), button_bounds);
let (bg_colour, fg_colour) = match region.state {
// ... snip ...
```

So, let's do it again, but this time with a `was_clicked` state. To match the shape of things, let's also try to pass that state out of the button so our caller can do something in response.

```rust
fn render_button(
	mut ctx: UiContext,
	label: String
) -> (impl Paint, bool) {
	let text = render_text(label);
	let button_bounds = pad(text.bounds(), 4);
	
	let region = hit_region(ctx.key("region"), button_bounds);
	let (bg_colour, fg_colour) = match region.state {
		PointerState::Rest => (Colour::DARK_BLUE, Colour::WHITE),
		PointerState::Hover => (Colour::DARK_BLUE, Colour::YELLOW),
		PointerState::Press => (Colour::WHITE, Colour::BLACK)
	};
	
	let text = recolour(text, fg_colour);
	
	// New: return both the composition and the state.
	let shapes = compose! [
		Quad::solid(button_bounds, bg_colour),
		region,
		text
	].bounded_by(button_bounds);
	
	(shapes, region.was_clicked)
}
```

But, this is awkward. Anyone using the button now has to deconstruct or index into it, meaning it doesn't slot neatly into constructs like `compose!` any more. You can find yourself shuttling state around in higher-up functions, and the button's actions and presentation can get visibly distant on the page for no good reason.

```rust
fn render_pause_menu(
	mut ctx: UiContext,
	current_view: &mut Option<Views>,
	game_state: &mut GameState
) -> impl Paint {
	// Render the buttons.
	let resume = render_button(ctx.key("resume"), "Resume the game");
	let achievements = render_button(ctx.key("achieve"), "View achievements");
	let options = render_button(ctx.key("options"), "Game options");
	let quit = render_button(ctx.key("quit"), "Quit to desktop");
	
	// Enact any required actions.
	if resume.1 {
		*current_view = None; 
		game_state.paused = false;
	}
	if achievements.1 {
		*current_view = Some(Views::Achievements);
	}
	if options.1 {
		*current_view = Some(Views::Options);
	}
	if quit.1 {
		game_state.loop_state = GameLoopState::Exit;
	}
	
	// Compose them into the final shape stream.
	let buttons = &[resume.0, achievements.0, options.0, quit.0];
	let buttons = vstack(buttons, 4);
	let buttons = align(buttons, screen_bounds, Some(0.5), Some(0.5));
	
	compose! [
		Quad::solid(screen_bounds, Colour::BLACK.opacity(0.5)),
		buttons
	]
}
```

Ideally, adding interactivity to a component wouldn't alter its fundamental shape, so altering its return value like this isn't going to scale well as components are refactored or take on different responsibilities.

So, instead of returning anything different at all, let's change `render_button` to accept a closure, specifying what to do when the button is clicked.

```rust
fn render_button(
	mut ctx: UiContext,
	label: String,
	on_activate: impl FnOnce()
) -> impl Paint {
	let text = render_text(label);
	let button_bounds = pad(text.bounds(), 4);
	
	let region = hit_region(ctx.key("region"), button_bounds);
	// Pass through to on_activate() right away if we can.
	if region.was_clicked { on_activate(); }
	
	let (bg_colour, fg_colour) = match region.state {
		PointerState::Rest => (Colour::DARK_BLUE, Colour::WHITE),
		PointerState::Hover => (Colour::DARK_BLUE, Colour::YELLOW),
		PointerState::Press => (Colour::WHITE, Colour::BLACK)
	};
	
	let text = recolour(text, fg_colour);
	
	// The return value is now completely unaffected.
	compose! [
		Quad::solid(button_bounds, bg_colour),
		region,
		text
	].bounded_by(button_bounds)
}

fn render_pause_menu(
	mut ctx: UiContext,
	screen_bounds: PxRect,
	current_view: &mut Option<Views>,
	game_state: &mut GameState
) -> impl Paint {
	// Now, the buttons can be directly composed, and no state
	// needs to be shuttled around at all.
	let buttons = &[
		render_button(ctx.key("resume"), "Resume the game", || {
			*current_view = None;
			game_state.paused = false;
		}),
		render_button(ctx.key("achieve"), "View achievements", || {
			*current_view = Some(Views::Achievements);
		}),
		render_button(ctx.key("options"), "Game options", || {
			*current_view = Some(Views::Options);
		}),
		render_button(ctx.key("quit"), "Quit to desktop", || {
			game_state.loop_state = GameLoopState::Exit;
		})
	];
	let buttons = vstack(buttons, 4);
	let buttons = align(buttons, screen_bounds, Some(0.5), Some(0.5));
	
	compose! [
		Quad::solid(screen_bounds, Colour::BLACK.opacity(0.5)),
		buttons
	]
}
```

With this change, what a button does always lives with the rest of the button's definition, preserving local reasoning and making code more concise without losing meaning.
### It's (not) time to act

There's a more subtle timing concern to walk through with our code as-written.

Our intention with this code is to let the user switch views and unpause the game when they press the button, so that's exactly what we wrote in the closures. But, consider the _effect_ of those closures on higher-up UI code:

```rust
fn render_game_ui(
	mut ctx: UiContext,
	screen_bounds: PxRect,
	game_state: &mut GameState
) -> impl Paint {
	let mut current_view = ctx.key("current_view").once(|| None);
	
	compose! [
		// If a view is open, add cinematic black bars over the world.
		current_view.is_some().then(|| {
			compose! [
				Quad::solid(PxRect::new( ... ), Colour::BLACK),
				Quad::solid(PxRect::new( ... ), Colour::BLACK),
			]
		}),
	
		// If we're paused, render the pause menu.
		game_state.paused.then(|| render_pause_menu(ctx.key("pause_menu"),
			screen_bounds,
			&mut current_view, // could be mutated by this call!
			game_state         // could be mutated by this call!
		)),
		// Otherwise, if we're unpaused, render the in-game HUD.
		(!game_state.paused).then(|| render_hud( ... )),
		
		// Render the view that's currently being shown, if any.
		current_view.map(|view| match view {
			Views::Achievements => render_achievements( ... ),
			Views::Options => render_options( ... )
		}),
	]
}
```

Because we're mutating inline with our UI builder code, clicking the buttons leaves us in a 'torn' state. Consider these two scenarios:

1. Clicking the "resume" button, changing `game_state.paused` halfway through.
2. Clicking the "options" button, changing `current_view` halfway through.

In the case of "resume", the pause menu is rendered due to `paused.then` executing, then `paused` is changed to `false`, which allows `(!paused).then` to execute _in the same frame_. As a result, you see the in-game HUD rendered over the pause menu unintentionally.

In the case of "options", the black bars can be skipped if no view is open, but once `render_pause_menu` sets `current_view`, the options view is rendered next, without the black bars that are supposed to render behind all views.

This points to a general truth: generally, you _don't_ want to change anything while the UI is in the middle of being built. Once you enter `render_ui()`, anything you use to determine what's rendered needs to be frozen in place until you're done - never read your writes. That means any changes you make in `render_ui()` must be _deferred_ to a later point.

So, how do you defer those changes? 

### Deriving a correct defer abstraction

Your first thought might be to somehow defer the _closures_; if setting `paused = true` is the problem, running the code that sets `paused` later seems like the obvious approach, given it barely impacts the code footprint:

```rust
// Pseudocode.
render_button(ctx.key("resume"), "Resume the game", || run_after_frame(|| {
	*current_view = None;
	game_state.paused = false;
})),
```

However, there's a key problem with this approach: because control flow is allowed to return from the component between construction and deferral, there's no longer any guarantee that the values you locally saw are up to date, or even exist at all - especially if other UI touches them.

This manifests in two ways:

- Values you read from within the closure are being sampled at an unpredictable point in the future, possibly _after_ other closures have mutated the values you're reading; effectful closures are combinatorially harder to reason about.
- Rust can't guarantee that it's memory safe. Our closures here refer to `game_state` mutably, which means nobody else is allowed to refer to `game_state` in the meantime, but `game_state`  is a _widely-used struct_. That will instantly turn into a compilation failure unless you pay for heavier state sharing methods.

In general, deferring closures is a bad idea. Code that lives in the component should run inside the component, not just for ease of editing, but because it's required to enforce a consistent, unmoving view of the world that can be reasoned about thoroughly.

The real problem here is that, in these closures, we're mutating the state our UI depends on while we're halfway through the build. A better solution - that solves _both_ for tearing _and_ for ensuring closures sample unmutated, predictable state - is to run all the closures immediately, but only read from _this frame's_ value and only write to _next frame's_ value.

Let's start by tackling `current_view`, which we currently define as a basic (and _slightly_ fictional) "mutable storage place". It's initialised once using `.once()`, then you write to that variable directly from there on out. Since it's mutable, it can change halfway through the procedure as we pointed out.

For things like `current_view` which are stored directly by our UI code, we already built a solution back in Chapter 1: we store a flip cache in `UiContext` that lets us read the old state while writing the new one. 

So, let's make `UiContext` sprout a new `.state()` utility that lets us store our in-UI state directly in the flip cache, using the same stable keys we've used everywhere else. When we call it, it'll give the current value, as well as a function to set next frame's value. This'll replace our (fictionalised) usage of `.once()` and fix that tearing:

```rust
fn render_game_ui(
	mut ctx: UiContext,
	screen_bounds: PxRect,
	game_state: &mut GameState
) -> impl Paint {
	// Now, current_view isn't `mut`, so we literally can't change it.
	// set_next_view also doesn't need `&mut` because it can't mutate anything
	// that will be read here; the mutations are kept internal to UiContext.
	let (current_view, set_next_view) = ctx.key("current_view").state(|| None);
	
	compose! [
		// Just a plain value, so every operation on it works!
		current_view.is_some().then(|| {
			compose! [
				Quad::solid(PxRect::new( ... ), Colour::BLACK),
				Quad::solid(PxRect::new( ... ), Colour::BLACK),
			]
		}),
	
		// .paused will still tear - we'll get back to it soon...
		game_state.paused.then(|| render_pause_menu(ctx.key("pause_menu"),
			screen_bounds,
			current_view,
			set_next_view, // Grant permission to set next frame's view.
			game_state
		)),
		(!game_state.paused).then(|| render_hud( ... )),
		
		
		// Always reads the original value, so current_view can
		// never cause a tear here (even when the pause menu acts!)
		current_view.map(|view| match view {
			Views::Achievements => render_achievements( ... ),
			Views::Options => render_options( ... )
		}),
	]
}

fn render_pause_menu(
	mut ctx: UiContext,
	screen_bounds: PxRect,
	current_view: Option<Views>,
	set_next_view: impl Fn(Option<Views>), // New setter!
	game_state: &mut GameState
) -> impl Paint {
	let buttons = &[
		render_button(ctx.key("resume"), "Resume the game", || {
			set_next_view(None); // No longer takes effect immediately.
			game_state.paused = false;
		}),
		render_button(ctx.key("achieve"), "View achievements", || {
			set_next_view(Some(Views::Achievements)); // Same here.
		}),
		render_button(ctx.key("options"), "Game options", || {
			set_next_view(Some(Views::Options)); // Same here.
		}),
		render_button(ctx.key("quit"), "Quit to desktop", || {
			game_state.loop_state = GameLoopState::Exit;
		})
	];
	let buttons = vstack(buttons, 4);
	let buttons = align(buttons, screen_bounds, Some(0.5), Some(0.5));
	
	compose! [
		Quad::solid(screen_bounds, Colour::BLACK.opacity(0.5)),
		buttons
	]
}
```

(As a happy note for any Fusion users in the audience, we have exactly replayed the [top-down control principle](https://elttob.uk/Fusion/0.3/tutorials/best-practices/state/#top-down-control) where we pair read-only state with a callback that can change it! In general, the `let (value, setter_for_value) = state();` shape should feel familiar to anyone who has worked in modern reactive frameworks recently.)

So, now that `current_view` is behaving properly, that only leaves `game_state.paused` to deal with. Since that's _not_ owned by the UI, it can't use a lightweight flip cache as its source of truth, so we're going to have to dig further.

The tearing caused by `game_state.paused` fundamentally comes from the same place as it did with `current_view` - _anything_ `mut` is dangerous, because it's capable of changing between reads. That's why we got rid of `.once()`, and it's why we're _now_ going to get rid of `&mut GameState`, replacing it with a downgraded read-only `&GameState`.

Which leaves only one question: how do you enact an unpause or quit action, if you can't mutate the game state to make that so?

Well, hey - we're pragmatists here, so let's employ a "temporary" solution, and create a dedicated callback for sending actions like unpause or quit.  On the other side of the callback, we can collect those actions into a list as we go, and once we reach the end of the frame, we can just manually loop through the list and make the changes we intended to.

```rust
fn render_game_ui(
	mut ctx: UiContext,
	screen_bounds: PxRect,
	// No longer mutable, so we know for the duration of this function,
	// the state we read from here isn't going to change.
	game_state: &GameState,
	// Similar to set_next_view, this will change what the game state is
	// only after this frame has finished being built.
	queue_action: impl Fn(Action)
) -> impl Paint {
	let (current_view, set_next_view) = ctx.key("current_view").state(|| None);
	
	compose! [
		current_view.is_some().then(|| {
			compose! [
				Quad::solid(PxRect::new( ... ), Colour::BLACK),
				Quad::solid(PxRect::new( ... ), Colour::BLACK),
			]
		}),
	
		// These two usages of .paused can never disagree, because
		// render_pause_menu can't change .paused mid-function anymore.
		game_state.paused.then(|| render_pause_menu(ctx.key("pause_menu"),
			screen_bounds,
			current_view,
			set_next_view,
			game_state
		)),
		(!game_state.paused).then(|| render_hud( ... )),
		
		current_view.map(|view| match view {
			Views::Achievements => render_achievements( ... ),
			Views::Options => render_options( ... )
		}),
	]
}

fn render_pause_menu(
	mut ctx: UiContext,
	screen_bounds: PxRect,
	current_view: Option<Views>,
	set_next_view: impl Fn(Option<Views>),
	game_state: &GameState, // No longer mutable!
	queue_action: impl Fn(Action) // Newly tasked with external state changes.
) -> impl Paint {
	let buttons = &[
		render_button(ctx.key("resume"), "Resume the game", || {
			set_next_view(None);
			queue_action(Action::SetPaused(false)); // No longer immediate.
		}),
		render_button(ctx.key("achieve"), "View achievements", || {
			set_next_view(Some(Views::Achievements));
		}),
		render_button(ctx.key("options"), "Game options", || {
			set_next_view(Some(Views::Options));
		}),
		render_button(ctx.key("quit"), "Quit to desktop", || {
			queue_action(Action::Exit); // Also no longer immediate.
		})
	];
	let buttons = vstack(buttons, 4);
	let buttons = align(buttons, screen_bounds, Some(0.5), Some(0.5));
	
	compose! [
		Quad::solid(screen_bounds, Colour::BLACK.opacity(0.5)),
		buttons
	]
}
```

By definition, this solution works, but maybe we don't feel so good about the idea of everything having to become an action. Perhaps it recalls memories of giant global reducers from some older UI frameworks, which we already know don't work.

But, we arrived here from first principles, so maybe there's something we can learn. Let's continue with this for now, and see if it develops into something more nuanced later, once we've exposed it to the real world.

### Touching on something input-ant

Cavey ships on a Steam Deck, which puts alternative inputs front and centre. We can't assume keyboard and mouse as the "best" input method. For example, someone may want to play Cavey using a _touch screen_.

So, let's consider the case of touch controls, which allow a player without any physical input types to still control the game. We'll focus on a single "attack" button to keep the code examples short.

When the player pushes the button to attack, we can't set the player's inputs directly in-line - that'd be `&mut GameState` - so we'll turn this state mutation into a queued action.

```rust
fn render_hud(
	mut ctx: UiContext,
	screen_bounds: PxRect,
	queue_action: impl Fn(Action)
) -> impl Paint {
	let attack = render_button(ctx.key("attack"), "Attack", || {
		queue_action(Action::Attack); // using our new defer vocabulary
	});
	let attack = pad(attack, 4);
	let attack = align(attack, screen_bounds, Some(0.5), Some(1));
	attack
}
```

This may raise an eyebrow. This _kind of_ looks like we're dispatching an _input_, right?

Let's talk about that.

Typically, input systems for games map key codes to "semantic" events like Attack, Jump, Walk and Look. Let's take Cavey's input system as a quick example - I won't get _too_ detailed about it, but I'll share some snippets as visual aids instead.

Here are some of Cavey's existing definitions from its existing input system, which separately define the _semantic_ information about an input (what kind of input is it?) and the actual events from the OS that will drive them (what devices write this input?):

```rust
(
	// "Attack" is a one-dimensional button-style input ...
	OutputDescription1D {
		output_id: output_ids::ATTACK,
		title: "Attack",
		constraint: Constraint1D::Button
	},
	// ... which is controlled by the left mouse button.
	Writer1D {
		input: Input1D::Mouse(MouseButton::Left),
		mode: WriterMode::Overwrite,
	}
),

(
	// "Walk" is a two-dimensional joystick-style input ...
	OutputDescription2D {
		output_id: output_ids::WALK,
		title: "Walk",
		constraint: Constraint2D::Joystick
	},
	// ... which is controlled by four keys on the keyboard.
	Writer2D {
		input: Input2D::Keyboard {
			up: KeyCode::KeyW,
			down: KeyCode::KeyS,
			left: KeyCode::KeyA,
			right: KeyCode::KeyD
		},
		mode: WriterMode::Overwrite
	}
),
```

Cavey even has some more _complex_ input types which accumulate over time with their own internal state, for example, the look direction is stored in the input system so that incoming mouse moves can directly change the camera angle with zero latency.

You could almost think of "Look" like a physical gimbal, which the mouse "pushes around" statefully:

```rust
(
	// "Look" is a two-dimensional gimbal-style input ...
	OutputDescription2D {
		output_id: output_ids::LOOK,
		title: "Look",
		constraint: Constraint2D::Gimbal
	},
	// ... which is statefully added to by mouse movements.
	Writer2D {
		input: Input2D::Mouse {
			pixels_per_unit: DVec2::ONE * MOUSE_DPI / 5.0
		},
		mode: WriterMode::Delta // not overwriting!
	}
),
```

This solution itself was motivated from first principles: one of Cavey's goals is to support remappable inputs. If you don't like the keys and input styles that were chosen for you, then you should be able to configure them to something else that _does_ work for you. But input handling is complex, so the system grew to capture all that complexity and state management.

As a result, all the game engine has to worry about, is reading off some semantic values without worrying about provenance at all:

```rust
// Player physics doesn't care about where these come from.
// The code path continues without branching on device types.
let walk_input = input_snapshot.states_2d.get(output_ids::WALK);
let jump_input = input_snapshot.states_1d.get(output_ids::JUMP);
let crouch_input = input_snapshot.states_1d.get(output_ids::CROUCH);

let forward_vector = flat_forward_vec(yaw);
let right_vector = flat_right_vec(yaw);
let walk_vector = right_vector * walk_input.x + forward_vector * walk_input.y;

let perform_jump = *jump_input > 0.5;
let perform_crouch = *crouch_input > 0.5;

// ... rest of locomotion code ...
```

Tying this back to our touch controls example, you'd *almost* want to write a definition like this, where you declared that an on-screen button would drive the input:

```rust
(
	// "Attack" is a one-dimensional button-style input ...
	OutputDescription1D {
		output_id: output_ids::ATTACK,
		title: "Attack",
		constraint: Constraint1D::Button
	},
	// ... which is controlled by creating an on-screen touch control.
	Writer1D {
		input: Input1D::OnScreen("Attack"),
		mode: WriterMode::Overwrite,
	}
)
```

That's _two_ places in our code base that want to care about on-screen controls. The longer you look at them, the more you start to see the similarities emerge:

- *Both* systems, at some times, need to show controls to the user on the screen.
- *Both* systems listen to incoming OS events to figure out when interactions occur.
- *Both* systems have complex state interactions that need to be managed.
- *Both* systems emit a series of actions to be interpreted by the game engine.

What's more: these systems need to be aware of each other's state. Touch controls need to disappear when you're in a dialog or menu. Drags and joystick inputs need to be redirected or have their inputs sunk when the game world isn't the focus. Gamepads need on-screen hints to accurately reflect the face buttons on the controller model being used.

So we've established these two systems broadly _do the same thing_, and are _highly coupled_.

Let me plant a thought in your head...

### Input handling *is* UI

To justify this, let's dig into the nature of the actions being emitted.

Both systems attempt to be _semantic_ to some degree - we don't want the game engine to care about specific OS events, devices, and such. `Action::Mouse1Clicked` would be bad for obvious reasons; it's not portable to different input modalities like gamepad and makes input remapping much harder.

But what about the grey areas? `Action::OpenSettingsMenu` would be input-agnostic, and it's the sort of thing Redux-style apps encode. But - as we saw with closures before - it doesn't make a lot of architectural sense to export local UI concerns to a global queue just to mutate some local state again. It complicates the control flow of the UI's business logic by splitting it across a boundary. So, we shouldn't be sending these actions over to the game engine side at all.

Instead, **the queue's vocabulary should just be defined by what the game engine needs to know.** The engine has no need for settings menus or scrollbars, but it _does_ have a use for knowing when the player is AFK, or when they're walking and looking around, or when they drop an item on the ground.

This line is exactly the same one most _input handling_ systems draw today. The reason we define verbs like Jump, Attack and Walk is because that's what our character controllers and gameplay systems need to know. What's more, the Redux problem is avoided by ensuring UI concerns are managed _locally_ using our `state()` constructs instead of being carved up across the action boundary.

The case for unifying the systems is strengthened by considering the frame loop as a whole; both input and UI want a low-latency path through simulation/response logic directly into a newly rendered frame. Since they both consume OS events and both emit game engine vocabulary, they are forced to occupy the same region of the frame loop; the game can't simulate or respond without having the completed action queue, and can't render until that simulation or response has completed.

Under all of these constraints, and given the deep inter-relation between the two systems demonstrated by touch controls, gamepad hints and input sinking, it's ultimately far more reasonable to construct them as one unit instead of two.

