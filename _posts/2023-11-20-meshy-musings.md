---
layout: post
title: Meshy musings
image:
  path: /assets/posts/meshy-musings/thumb.jpg
  width: 256
  height: 256
---
[Roblox is getting editable meshes - and that's going to change the platform forever.|https://x.com/dub_dino/status/1670265140795809794] Yet, after playing around with `DynamicMesh`/`EditableMesh` for a while, I have thoughts.

## Do it yourself

In the last iteration I could play with before they took away our beloved `zIntegration`, there was one persistently annoying flaw. The moment you turn a `DynamicMesh` into a `MeshPart`, that's it. Your mesh data is locked and no longer editable. This seems to make enough sense to me (I imagine it is undesirable to clone potentially huge mesh data willy-nilly) but it's an unfortunate limitation. I'd be pleasantly surprised if it were waived by release, but I completely expect this to be the behaviour on release day.

My general advice for anyone looking to build for `DynamicMesh` is therefore quite simple; don't rely on it, do it yourself. I would consider `DynamicMesh` to be a sort of 'compilation API', in that you should be using it as the final step in your process to turn your existing data into some immutable, rendered form. Specifically, this means you won't be interfacing with `DynamicMesh` as part of any 'editor' workflow. If you wanted to make a triangle terrain plugin using this, for example, you'd still be storing the vertices in Luau and operating on that data, rather than simply shunting it off to a `DynamicMesh`.

In some ways, I actually consider this to be a good thing. Implementing mesh editing operations across the C boundary sounds like a performance nightmare, even if Roblox were to implement them super efficiently on their end. It'll likely always be faster to do pure computation in Luau, especially now that native JIT is rolling out. Moreover, I expect that controlling your own data structure means you'll get more semantically useful data structures to work with, rather than being always forced to work with vertices and triangles. It also means you don't have to wait for Roblox to finish figuring out `DynamicMesh` to work on your ideas today; since you're just using `DynamicMesh` as an output, you could just as easily reroute to a `WireframeHandleAdornment` or wedge part rendering.

## Manifold mandate

An interesting aspect of the `EditableMesh` API is that it almost incentivises you to construct non-manifold geometry (for example, one-sided meshes). In general I get the impression that Roblox does not like this sort of thing.

For collision, it already processes meshes to make sure they're all nicely closed off and decomposed into smaller chunks the physics engine can deal with. Furthermore, judging by what they've said on their 'vision', it seems like Roblox really wants you to be creating physically plausible objects, and does not want you to cut corners like everyone in computer graphics historically has. No baking lightmaps, no fudged constants.

This makes me wonder whether the eventual API will impose restrictions on *what* geometry you can generate. In the most extreme case, it could mandate that everything has to be properly closed off, that everything must have some minimum inner volume, that you can't generate paper-thin objects, and so on. Alternatively, maybe they'll just keep on quietly fixing our janky meshes with their (actually pretty awesome) convex decomposition algorithms.

You should probably err on the side of caution here. Be prepared to generate your meshes properly, if Roblox decides that's a thing they want to do.

## This is going to be huge

If you think you know how huge this is going to be, you're off by an order of magnitude. This is going to be *huge huge*. This is like, "we just gave everyone shaders" level huge. You're about to see a Cambrian explosion of perturbed polygons.

I reckon that, in combination with some other secret knowledge I've gleaned, we're about to see the birth of a new genre of game. Or perhaps a rebirth? I think people are going to make experiences specifically around content creation. Perhaps it won't be that powerful or even desirable at first, but people will try to do it, and I have no reason to believe someone won't crack some kind of formula. Maybe someone from the Royale High side of Roblox will make some pretty sweet dress maker or something that people could export into Studio and upload as their own UGC. I'm not super imaginative beyond that, unfortunately, but this is one of the lynchpins that was holding back a whole lot of creative possibilities. Even in existing games that would find no use for content creation mechanics, you'll likely see technical artists going wild using this for all sorts of dynamic VFX. You might even see it for mundane things like automatic optimisation.

I'm really hoping this stuff works in Studio, since my domain is building professional plugins. In particular, I would love to bring a ton of concepts over from full-fat modelling packages to make building high-fidelity experiences easier. The dream is - one day - that you could do a whole archviz in Roblox if you wanted to, and you wouldn't want to pull your teeth out at the end. That's my bar to clear in my head. 

I even have a few gorgeous graphics hacks in my back pocket that would get totally blown wide open by dynamic mesh access. For example, [I recently figured out how to do PBR material blending with detailed transitions.]  Since this depends on mesh trickery and specially authored PBR materials, it's currently something you have to manually (and slowly) author externally, then carefully recreate in Studio. In the future, it'd be totally reasonable to depend on `EditableMesh` access to let you paint it directly in Studio. If that's then combined with the ability to *generate* PBR materials from scripts using `EditableImage`, well, you've got yourself a fully automated process. Imagine being able to paint this shit in real time, *on any surface*:

![Cobblestones blending naturally into forest ground.](pretty-transitions.jpg)

We're in for a good time.