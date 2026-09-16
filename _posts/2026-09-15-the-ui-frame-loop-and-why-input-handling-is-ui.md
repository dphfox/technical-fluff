---
layout: post
title: The UI frame loop, and why input handling is UI
image:
  path: /assets/posts/the-ui-frame-loop-and-why-input-handling-is-ui/thumb.jpg
  width: 256
  height: 256
  excerpt: In this final blog post, we'll round out with a broader understanding of where our UI procedure sits within the wider frame loop, and how it can be extended to handle interactions from the user in order to forward semantic events to the game engine.
---

*This is Chapter 3 of [a 3-part blog post series](https://fluff.blog/2026/09/13/i-made-my-perfect-ui-library.html) discussing "Perfection", a greenfield UI library I built for personal use.*

---

Over the last two blog posts, we've built up a whole UI system from first principles, and uncovered a fundamentally simpler way of organising the shapes in our UI that allows us to use function composition instead of a strict UI hierarchy, as well as tackle difficult layout problems with ease.

In this final blog post, we'll round it out with a broader understanding of where our UI procedure sits within the wider frame loop, and how it can be extended to handle interactions from the user in order to forward semantic events to the game engine.

---

## ...hey, just... query the mouse position?

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

## Breaking the dependency cycle

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

## Frames live between renders

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

## Clicking things in place

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
	screen_bounds: PxRect,
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

## It's (not) time to act

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

## Deriving a correct defer abstraction

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
			queue_action(Action::TerminateLoop); // Also no longer immediate.
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

## Touching on something input-ant

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
	let attack = align(attack, screen_bounds, Some(0.5), Some(1.0));
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

## Input handling *is* UI

To justify this, let's dig into the nature of the actions being emitted.

Both systems attempt to be _semantic_ to some degree - we don't want the game engine to care about specific OS events, devices, and such. `Action::Mouse1Clicked` would be bad for obvious reasons; it's not portable to different input modalities like gamepad and makes input remapping much harder.

But what about the grey areas? `Action::OpenSettingsMenu` would be input-agnostic, and it's the sort of thing Redux-style apps encode. But - as we saw with closures before - it doesn't make a lot of architectural sense to export local UI concerns to a global queue just to mutate some local state again. It complicates the control flow of the UI's business logic by splitting it across a boundary. So, we shouldn't be sending these actions over to the game engine side at all.

Instead, **the queue's vocabulary should just be defined by what the game engine needs to know.** The engine has no need for settings menus or scrollbars, but it _does_ have a use for knowing when the player is AFK, or when they're walking and looking around, or when they drop an item on the ground.

This line is exactly the same one most _input handling_ systems draw today. The reason we define verbs like Jump, Attack and Walk is because that's what our character controllers and gameplay systems need to know. What's more, the Redux problem is avoided by ensuring UI concerns are managed _locally_ using our `state()` constructs instead of being carved up across the action boundary.

The case for unifying the systems is strengthened by considering the frame loop as a whole; both input and UI want a low-latency path through simulation/response logic directly into a newly rendered frame. Since they both consume OS events and both emit game engine vocabulary, they are forced to occupy the same region of the frame loop; the game can't simulate or respond without having the completed action queue, and can't render until that simulation or response has completed.

Under all of these constraints, and given the deep inter-relation between the two systems demonstrated by touch controls, gamepad hints and input sinking, it's ultimately far more reasonable to construct them as one unit instead of two.

## Completing the interaction model

Let's use this insight to finish up our input handling. As an easy first win, we can finally codify our understanding of what our pause menu is _actually_ doing in our mental model, and rename our actions:

```rust
fn render_pause_menu(
	mut ctx: UiContext,
	screen_bounds: PxRect,
	current_view: Option<Views>,
	set_next_view: impl Fn(Option<Views>),
	game_state: &GameState,
	queue_action: impl Fn(Action)
) -> impl Paint {
	let buttons = &[
		render_button(ctx.key("resume"), "Resume the game", || {
			set_next_view(None);
			 // Notify the game engine we want to resume gameplay.
			 // We're now using the game engine's _own_ vocabulary for this.
			queue_action(Action::Idle(false));
		}),
		render_button(ctx.key("achieve"), "View achievements", || {
			set_next_view(Some(Views::Achievements));
		}),
		render_button(ctx.key("options"), "Game options", || {
			set_next_view(Some(Views::Options));
		}),
		render_button(ctx.key("quit"), "Quit to desktop", || {
			// Same here - now refers to the game, not the loop it controls.
			queue_action(Action::ExitGame);
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

Now for our first-person input handling. Since we don't need to query a specific spot on the screen for keys, mouse or gamepad inputs, we can use a hit region that has unlimited bounds to capture the inputs we care about.

Other controls can then be composed on top of this hit region to occlude certain inputs, for example, touch controls can sink presses before they reach our pitch/yaw or attack/interact inputs. 

```rust
fn render_game_ui(
	mut ctx: UiContext,
	screen_bounds: PxRect,
	game_state: &GameState,
	queue_action: impl Fn(Action)
) -> impl Paint {
	let (current_view, set_next_view) = ctx.key("current_view").state(|| None);
	
	compose! [
		// ... rest of render_game_ui function ...
		
		// Process player inputs / show on-screen controls
		// only when no other UI is active right now.
		current_view.is_none().then(|| render_inputs(
			ctx.key("inputs"), 
			screen_bounds, 
			game_state, 
			queue_action
		))
	]
	
}

// Because it implements Paint, it reserves the right to
// return any on-screen visual elements that we may want,
// for example touch controls.
fn render_inputs(
	mut ctx: UiContext,
	screen_bounds: PxRect,
	game_state: &GameState,
	queue_action: impl Fn(Action)
) -> impl Paint {
	let mut inputs = InputSnapshot {
		walk_vector: DVec2::ZERO,
		pitch_yaw: game_state.local_player.pitch_yaw,
		jump: 0.0, crouch: 0.0, attack: 0.0, interact: 0.0
	};
	
	// Define an input listener which takes up infinite 2D space,
	// but which still has a "z-index" in our occlusion stack.
	let hit = hit_region(ctx.key("hit"), PxRect::INF).mouse_locked();
	
	// Direct inputs for keyboard, taken from the hit region.
	// Sampled at the frame end as continuous input.
	if hit.key_is_held(KeyCode::W) { inputs.walk_vector += DVec2::Y; }
	if hit.key_is_held(KeyCode::S) { inputs.walk_vector -= DVec2::Y; }
	if hit.key_is_held(KeyCode::A) { inputs.walk_vector -= DVec2::X; }
	if hit.key_is_held(KeyCode::D) { inputs.walk_vector += DVec2::X; }
	if hit.key_is_held(KeyCode::LeftShift) { inputs.crouch += 1.0; }
	
	// Direct inputs sampled across the whole frame interval,
	// *not* point sampled at the end.
	// This ensures presses aren't missed.
	if hit.key_has_been_held(KeyCode::Space) { inputs.jump += 1.0; }
	if hit.mouse_has_been_held(MouseButton::Left) { inputs.attack += 1.0; }
	if hit.mouse_has_been_held(MouseButton::Right) { inputs.interact += 1.0; }
	
	// Mouse delta between last render and this render,
	// which we integrate into pitch_yaw.
	const PIXELS_PER_RADIAN: f64 = 6000.0;
	inputs.pitch_yaw += hit.mouse_delta() / PIXELS_PER_RADIAN;
	
	// A few touch controls to show on the screen.
	// These are overlaid on top of `hit` so they take over inputs
	// on their own part of the screen only.
	let on_screen_controls = {
		// Same as render_button, but fires the callback continuously.
		let attack = render_button_has_been_held(ctx.key("touch_attack"), "Attack", 
			|| inputs.attack += 1.0
		);
		let interact = render_button_has_been_held(ctx.key("touch_interact"), "Interact",
			|| inputs.interact += 1.0
		);
		let buttons = hstack(&[attack, interact], 4); // New operation; vstack, but horizontal
		let buttons = align(pad(buttons, 4), screen_bounds, Some(0.5), Some(1.0));
		buttons
	};
	
	// ... gamepad, trackpads, VR 6DOF, normalisation, etc ...
	
	// Send it to the engine.
	queue_action(Action::PlayerInputs(inputs));
	
	// `hit` - our input receiving layer - comes before our
	// on screen controls, so touch controls sink inputs before
	// they get to our other controls.
	compose! [
		hit,
		on_screen_controls
	]
}
```

As you can see, the code can look just as simple as querying inputs every frame, but now with the convenience of being trivially gated by the UI's `current_view` state, and being able to render on-screen buttons in-line with the rest of the input logic: we get exactly our desired "on-screen" declaration in line with our input handling that we previously fantasised about under the old divorced system.  

What's more, these controls are rendered using a variant of our existing `render_button` that fires continuously when held, showing how non-special touch controls are under this system - almost all of the code is shared without ceremony, and we could easily reuse this for a non-touch-control UI that needs similar behaviour.

With that, we've built a fully-motivated minimal UI interaction stack that covers all the bases needed to build various kinds of game UI for a wide range of input modalities, while avoiding all the common pitfalls, and all without introducing any complex book-keeping, asynchronous primitives, or unwieldy cross-system coupling. 

From here, it's theoretically easy to extend to remappable inputs, or even to unique input devices like analogue keyboard switches, 6DOF tracked VR, or custom driving wheels, joysticks and even MIDI instruments - all with one mental model.

All input styles *are* some kind of UI - it's just more visible with some than others.

---

## Conclusion

With that, I hope I've now demonstrated to you the three core principles of my new UI library, and why I selected them:

1. Immediate mode & flip caches, and why I chose them for their lighter weight and leaner cache interactions.
2. Operating on shapes as data, and why it's more powerful, expressive & animatable than a box model.
3. UI as the single procedure that consumes OS events, and how I think about event timing between frames.

I'm currently actively implementing and playing with these three ideas in the context of [Cavey](https://caveygame.com/), my casual cave spelunking game. When I have the time, I'd love to show off some cool example projects or develop these ideas further to show just how powerful a fully-composable shape-centric UI stack can be.

I haven't touched on ergonomics too much in this post, instead choosing to focus on fundamentals. As an example, the function parameters in these render methods could get harder to manage over time! However, most programming languages have decent tools for managing these things, such as builder patterns or ways of doing dependency injection, but I wanted to keep this blog post series clean and focused on the core. These posts cumulatively are hitting north of 15,000 words as-is!

I figured I would get these out in the world instead of waiting forever. I hope some of these ideas inspire you to think about UI more deeply in the future :)