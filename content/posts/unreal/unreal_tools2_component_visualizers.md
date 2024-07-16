---
title: "Unreal Editor Tools Series 2: Component Visualizers"
date: 2024-07-16T19:12:41-04:00
draft: true
toc: false
images:
tags:
  - unreal
  - unreal_editor
  - unreal_editor_tools
---

In the last few months I happened to end up working quite a bit with the Unreal Editor per se,
particularly in how to extend. Turns out that, if you're willing to scrounge the internet and read
quite a bit of source, there is quite a bit you can do to extend Unreal, from better visualizations,
editing tools to complete overhaul of the editor.

In order of complexity the tools I want to write about:

* Component Visualizers: Run some visualizing and editing code when an actor with a particular
  component is selected (Eg. The Spline Component).
* Render Scene Proxies: Extend the way a particular actor/component are rendered. A lot of the
  "normal" editor visualizations are done this way (Eg. NavMesh).
* Editor Modes: This is the big Kahuna, in which you can pretty much override all interactions with
  the viewport. The normal mode in Unreal is actually an editor mode called "Selection Mode", but
  others like Foliage and Fracture are other instance of this. It is also the more complex.

# Component Visualizers

Component visualizers are the most documented of the group of tools described above, so I will not
go into _too_ much detail about particular techniques, as there are very detailed blog posts around.
I will link to them at the end of this post.

Component Visualizers are, as their name describes, a way adding additional visualization for
actor(s) that have a particular component. The catch is that this visualization is only active when
these actors are selected, otherwise they don't activate at all. This works great when editing a
particular ingredient in isolation, like modifying the vision range of a tower, but can be a bit
limiting when working in multi-actor flows. But there are other tools for tackling those.

