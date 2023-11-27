---
layout: post
title: PluginAction sucks (and how to improve it)
image:
  path: /assets/posts/pluginaction-sucks-and-how-to-improve-it/thumb.jpg
  width: 256
  height: 256
---
You'd assume that, as a plugin developer, I've used `PluginAction` a ton of times. However, they are missing a critical piece of functionality, rendering them nearly useless.

## The key issue

It's 2023 and you still can't declare a default key bind for `PluginAction`.

The use cases for an action with no keyboard shortcut by default are extremely limited. That kind of action is saved for more niche operations which aren't important enough to surface to the user directly, but might still be useful in, say, the command palette. 

Key-bind-less actions are useless for any primary actions, though. Unfortunately, most actions in Roblox plugins tend to be primary, because Roblox plugins aren't exactly massive software suites. No self-respecting developer will write a plugin where you have to spend five minutes manually assigning keyboard shortcuts for basic operations, and no self-respecting user will spend five minutes every time they install a new plugin. This is why plugin developers still use `UserInputService` for they keyboard bindings; it requires no setup, it ensures everyone has a consistently assigned key map to make communication easier ("press E to extrude" etc.), and it just works.

That is the bar that `PluginAction` *must* meet if it wants to be useful for more than two people. If it requires any setup, if it leads to inconsistencies between users, or if it fails sometimes, it's not good enough and nobody will care.

## Fixing a flawed suggestion

I've seen many suggestions along the lines of adding a `KeyCode` and/or `ModifierKey` to the constructor, as a 'suggested default' that's applied so long as no other action is bound there. 

Let's check this API design against our criteria:

- This does not require the user to do any setup.
- This is mostly consistent between users. It's not perfectly consistent because -
- Depending on what plugins and binds the user has set up already, certain binds will silently fail to register.

So this is no good either. It trades the setup stage for not knowing what's bound on your keyboard, because all your plugin actions get spliced together silently into some abomination of random key binds that you aren't told about.

Here's my first fix: unless the user explicitly changes the key binding, the default is always adopted for the `PluginAction`, **even if another action is bound there already.** In addition, users should be allowed to specify overlapping key bindings in the 'Customise Shortcuts' dialogue without being prevented or having other key bindings reset. TL;DR - don't consider whether key bindings collide ahead of time.

Of course, this immediately introduces ambiguity when a key binding is used more than once. If my plugin registers G for 'grow grass' and another registers G for 'gradient tool', which one should be activated when I press G?

Firstly, we must recognise that actions don't always apply. For example, that 'gradient tool' might be part of a UI editing tool, and we only care about activating the gradient tool when it's on screen and in focus. `PluginAction` currently doesn't give you any way of communicating that.

That leads me to my second fix: plugins should be able to enable and disable `PluginActions` at any time, whenever they feel like their actions are relevant or not. This reduces the chance of ambiguity by reducing the number of active key bindings at any one time.

However, we can't expect all plugins to behave, and moreover, we can't expect only one plugin to be active at any one time. For example, there can be multiple widgets on screen, there can be plugins running in the background without visible UI, etc. The next step, then, is figuring out if one action should be prioritised over others competing for the same key binding.

I suggest sorting the competing actions into tiers. Here's a suggestion for what these tiers could look like:

1) Actions belonging to Roblox Studio
2) Actions belonging to the plugin with the currently focused dock widget
3) Actions belonging to the activated plugin with the exclusive mouse
4) Actions belonging to any activated plugins regardless of exclusive mouse
5) Actions belonging to any other plugins

If actions exist in a tier, all lower tiers are ignored. The actions in the highest (non-empty) tier are left to compete among themselves.

Some examples:

- A plugin tries to bind `Ctrl+S`. 
	- When the user presses `Ctrl+S`, the plugin competes with Roblox Studio's save command, and loses because plugins are always in a lower tier compared to Studio.
- A building plugin with exclusive mouse binds `Space`. A plugin with a dock widget also binds `Space`.
	- If the user presses `Space` just after interacting with the dock widget, the dock widget plugin wins because its focused dock widget places it in a higher tier. However, if the user presses `Space` just after interacting with the viewport, the building plugin wins because it's activated in the viewport with exclusive mouse, and no other dock widget is stealing the focus from the viewport right now.
- Three deactivated plugins bind `P`. An activated non-exclusive-mouse plugin also binds `P`.
	- When the user presses `P`, the activated plugin wins because the deactivated plugins are placed in the lowest tier. The higher tier of the activated plugin immediately makes all lower tiers lose.
- Two activated non-exclusive-mouse plugins bind `R`.
	- When the user presses `R`, there is a conflict and neither plugin wins. This is because they are in the same tier, and so theoretically have the same priority as each other. This shows that the priority system does not resolve all conflicts.

This system should realistically resolve most conflicts and help plugins play nicely together the way the user would expect. It also prevents plugins from overriding Roblox Studio actions, which might stop users from accessing important functions easily such as saving and publishing.

However, as the last example demonstrates, this priority system won't resolve all conflicts. Rather than arbitrarily trying to assign granular priorities per-action (which would be hard to learn), I instead propose that Studio gives up at this point. Instead of trying to resolve the error, a toast should be shown along the bottom of the screen, notifying the user of the conflict:

![Toast saying 'G key binding failed; it is used by 5 actions (Foo, Bar, Baz, Frob, Garb)' with a link to 'Change keyboard shortcuts'.](/assets/posts/pluginaction-sucks-and-how-to-improve-it/toast.png)

This is better than silently failing and it's better than choosing something arbitrary for the user. It makes sure they're aware of the conflict and gives them control over how it is resolved.

## Is it good enough?

Let's check this new design against our criteria:

- This does not require the user to do any setup.
- Key binds are always consistent between users; nothing is changed unless the user explicitly re-binds.
- Failure is much rarer thanks to the tiered priority system and plugins being able to disable `PluginActions` should they no longer wish to reserve the key binding. However, when it does fail, it is failing where the user's intent is not clear, and it is clear and actionable, rather than silent and confusing.

I think this would be good enough to ship.