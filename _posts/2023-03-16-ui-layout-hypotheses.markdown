---
layout: post
title:  "UI layout hypotheses"
image:
  path: "/assets/posts/ui-layout-hypotheses/thumb.jpg"
  width: 256
  height: 256
comments_url: "https://elk.zone/tech.lgbt/@Elttob/110030464471607938"
---

Presented with wholly inadequate logical explanation - that is, none.

- Hierarchy is the wrong abstraction. Layout and placement is a data flow
incorporating more variables than simple positions and sizes, and as such are
better modelled by pipelines such as reactive graphs.
- Placement of UI elements can be informed by layouts, but cannot be prescribed
by them. That is to say, all placement is absolute positioning which calculates
and factors in layout information at its own discretion.
- Layout is not prescribed to UI elements. There is, by definition, no single
way to lay out a UI element, because the placement process may choose to
generate whichever layouts it pleases in accordance with the current condition
of the reactive graph.

Further hypotheses derived from these;

- Today's layout systems are backwards. We have gotten very good at building
prescriptive single-layout systems that wholly dictate placement.
- As a result, it's obscenely difficult to blend between multiple layouts, or to
ignore layout information. Anyone who has implemented parallax, drag &
drop, or connected animations, has already encountered this friction.
- The first person to get the DX right, in accordance with the first three
hypotheses, will enjoy massive success in unifying all placement processing into
a universal, understandable, intuitive and debuggable pipeline.
