---
title: "Unreal Editor Tools Series 1: Editor Modules"
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

# Editor Modules

Before we can start creating editor tooling, we need to create a module where this code is going to
run. In order to modify the editor, we need to hook ourselves to the "Editor" side of Unreal, rather
than the runtime one, which is what is normally done.

The typical way to do this is to create an Editor module. Editor modules have two important
properties: Unreal understand them as editor code, which permits us to do all kinds of interesting
things and also, this code will not be compiled in client/server builds.

## Unreal Software Architecture (Quick) Primer

{{<admonition tip>}}
A much better and in depth explanation of these files and structure can be found in this
[excellent video](https://www.youtube.com/watch?v=94FvzO1HVzY) by Alex Forsythe.
{{</admonition>}}

As a quick reminder for [Unreal Modules](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-modules?application_version=5.3),
they are a grouping of code that is meant to be managed together. Modules can depend on other
modules and there is a whole set of options about how to manage Modules. They are configured by a
`<MODULE_NAME>.Build.cs` file, which it's used by UBT (Unreal Build Tool) to know how to build the
module.

After the module has been defined, they have to be referred to by an [Unreal Target](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-build-tool-target-reference),
which are defined by a `<TARGET_NAME>.Target.cs`. Targets are what we actually end up building, like
Game builds, or initially and editor build (which you can think as client + editor), but they can
also be standalone programs. Normally these files are found within the `Source` directory.

A confusing bit of the puzzle is that Modules are actually "defined" into your project not only
by the `*.Build.cs` file, but also they have to be defined into your project's `UProject` file,
which describes the big picture configurations of the project. A way to think about it is the
following:

* `Build.cs` describe how a module is built. The most important option is the module dependencies,
  which determines which headers and code will be available to link against. Most linking errors
  come from a missing dependency here.
* `Target.cs` describe how a target is built. You can think of a Target of a collection of modules
  with a "main" module in it.
* `Uproject` describe your project as a whole, which can (and normally does) have many targets.
  This file also determine at what moment modules are loaded, since they can be enabled in different
  orders, even on demand at runtime.

The main takeaway is that for creating editor code, we will need an editor module that depends on
your gameplay module.

## Creating an editor module

There are many ways of creating modules. You can even create them via the editor, which is a normal
way pointed out in videos. While they work just fine, I prefer to understand the layout and build
the modules myself, so I can understand what is going on, rather than just pressing buttons.


