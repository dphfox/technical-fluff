---
layout: post
title: Early ruminations on monorepo life
image:
  path: /assets/posts/early-ruminations-on-monorepo-life/thumb.jpg
  width: 256
  height: 256
---

A while back, as part of some broader refactoring, I moved all of the Elttob Suite products into a single repository. Here's my setup and how it's working for me so far.

## The setup

I have three top-level directories:

- **Libraries**
	- All of the dependencies of Elttob Suite products are consolidated here. Each library gets its own folder under here.
- **Licenses**
	- To make sure I retain licensing and copyright information, I include some readme files inside the product packages. This directory contains all the notices I need to include so they're easy to reference and re-use.
- **Products**
	- These contain the unique contents of each product, not including dependencies - those will be pulled in from the other folders. In the final package, these will show up as a hierarchy alongside all the other libraries and dependencies.

I assemble each package directly in Rojo, explicitly pulling in only the required dependencies.

```JSON
"Elttob Reclass": {
  "$className": "Folder",
  "CODE IS COPYRIGHTED": {
    "$path": "Licenses/PluginCopyright.lua"
  },
  "Reclass": {
    "$path": "Products/Reclass"
  },
  "Synchro": {
    "$path": "Libraries/Synchro"
  },
  "SharedToolbar": {
    "$path": "Libraries/SharedToolbar"
  }
}
```

## The benefits

I've been working this way for a while now. I've noticed a few great benefits of this:

- Because every product draws from the same pool of dependencies, I'm heavily incentivised to not use multiple versions of the same library at the same time. When I do a version upgrade, I am doing a unified upgrade across the entire suite, avoiding version conflict issues.
- It's incredibly easy to reuse code - just a directory change and a project.json update to pull it in. No nonsense with external package management, which has a tendency to suck badly in Luau. I almost don't notice the modularity of my code until I need to take advantage of that modularity, at which point it actively makes my life easier.
- The package structure is nice, consistent and well-defined as a flat folder structure. This makes anything with dependencies super easy to manage, because they can safely assume their dependencies are located under the same parent with no trouble. This is a no-tech, low-cost solution.
- My products are often pretty related and intertwined, so being able to jump between product codebases with zero delay or lost context is awesome.

I think it was absolutely the right choice for the sort of codebase I'm building. It doesn't introduce more technology than it needs to, and it has low mental overhead. It's boring and it works.