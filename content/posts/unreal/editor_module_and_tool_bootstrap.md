---
title: "Editor Modules and Tools bootstrap"
date: 2024-04-07T10:02:18-04:00
draft: true
toc: true
images:
tags:
  - unreal
  - editor_tools
---
Once you start developing with Unreal, inevitably you will start wanting to run some custom tools to
aid you, whether that's working with some of the actors on the map, running some kind of generation
script, an external program, etc. In my case, it was that I wanted to be able to launch the server
executable for [NetImgui](https://github.com/sammyfreg/netImgui) from within Unreal itself.
Something like having a section at the top menu bar where I could select "Launch NetImgui" from.

{{<admonition info>}}
Most of the information of this post is based from this treasure trove of information in shape of an
article: https://lxjk.github.io/2019/10/01/How-to-Make-Tools-in-U-E.html.

The article is a bit terse and hard to follow, so I'm writing these posts by having gone and
implemented some of those patters, but with my understanding and twists. Maybe my way of explaining
it makes it easier... Maybe.
{{</admonition>}}

This is what we want to achieve, but I will take some roundabout time to create some bootstrapping
work so that next tools are easy to integrate:

{{<figure src="images/posts/editor_module_and_tool_bootstrap/menu_bar.png" >}}

## Editor Modules

In order to modify the editor, we need to hook ourselves to the "Editor" side of Unreal, rather than
the runtime one, which is what is normally done. The typical way to do this is to create an Editor
module. Editor modules have two important properties: Unreal understand them as editor code, which
permits us to do all kinds of interesting things and also, this code will not be compiled in client
builds.

### Unreal Software Architecture (Quick) Primer

{{<admonition tip>}}
A much better and in depth explanation of these files and structure can be found in this
[excellent video](https://www.youtube.com/watch?v=94FvzO1HVzY) by Alex Forsythe.
{{</admonition>}}

As a quick reminder for [Unreal Modules](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-modules?application_version=5.3),
they are a grouping of code that is meant to be managed together. Modules can depend on other
modules and there is a whole set of options about how to manage a Module. They are configured by a
`<MODULE_NAME>.Build.cs` file, which it's used by UBT (Unreal Build Tool) to know how to build the
module.

After the module has been defined, they have to be referred to by an [Unreal Target](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-build-tool-target-reference),
which are defined by a `<TARGET_NAME>.Target.cs`. Targets are what we actually end up building, like
Game builds, or initially and editor build (which you can think as client + editor), but they can
also be standalone programs. Normally these files are found within the `Source` directory.

A bit confusing bit of the puzzle is that Modules are actually "defined" into your project not only
by the `*.Build.cs` file, but also they have to be defined into your project's `UProject` file,
which describes the big picture configurations of the project.

