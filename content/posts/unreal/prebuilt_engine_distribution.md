---
title: "Prebuilt Engine Distribution"
date: 2024-02-06T20:07:13-05:00
draft: true
toc: false
images:
tags:
  - unreal
---

A common problem that studios using Unreal will face, if they perform engine source modifications
(which they will inevitably do), is to how to distribute the editor to content creators. I am using
the label `content creator` to differentiate from the other role, which I will label `programmer`
role.

The main difference on this role is whether they build the editor from source or they use a prebuilt
version. This normally can be thought of whether they need/use the set of toolchains needed to build
this code (MSVC Build Tools, .NET, etc.). This difference is further separated from whether
programmers are building the engine editor source code, or only the project one. This gives us
another role which is the programmer who builds the engine editor, rather than using a prebuilt
version.

So to clarify, we have three "roles":
1. Content Creator: No compilation at all.
2. Programmer: Only builds code within the game project.
3. Engine Builder: Builds the code of the engine.

{{< admonition note >}}
In this context, Content Creator is not limited to creating assets, but rather a whole game can be
built by this role by using Blueprints. Since blueprints don't require source code compilation (are
rather treated as data), they fall into the category of content creation for the purposes of this
discussion.
{{< /admonition >}}

In turn, we have 3 levels of Projects:
1. **Content-Only Project**: The one distributed by Epic via the Epic Launcher. This project does
not require any programming tooling at all. This realm is sometimes called (sometimes pejoratively)
the realm of "Scripting".
2. **C++ Project**: The moment we write C++, either to create actors, but specially when we create
editor classes (such as tools and widgets), this becomes a programmer facing project. This would be
the real of the "Gameplay Programming".
3. **Modified Engine**: This is when the project decides that it requires modifications to the base
engine that cannot be done on top of the normal engine. This is normally done to swap/modify/extend
base behavior of the engine. Think changing the ticking logic, rendering, physics determinism, etc.
This would be literally entering the realm of "Engine Programmer".

Let's take a closer look at each of this projects and see how they relate

# Content-Only Projects

This is the simplest case, as there is no need to distribute custom editors. Each member of the team
can get the same version of the editor from the Epic Launcher and they should be good to go. This
comes with the benefit that no C++ programming experience is required, which can be daunting for
people who have not programmer before. Blueprints on the other hand, are much more accessible.

This simplicity comes with the requirement that only blueprint code can be used, which can be
sometimes limiting in terms of complexity management, performance, etc. But if the project scope is
such that can be tackled by this scope, then this is by far the simplest way forward.

Roles needed:
- Content Creator

Tools needed:
- Epic Launcher
- A way to distribute changes between teammates, normally VCSs (Version Control System): Perforce,
Git, Dropbox, etc.

# C++ Projects

This project, at some point, will require a programmer to build C++ from source. In this context, we
are still using the Epic Launcher to get the editor, which we leverage and link against for making
our C++ functionalities.

This also has the added problem of how would content creators, which by definition in this post do
not have toolchains (aka. do not compile) get the changes done by the programmers. These changes are
not distributed by the Epic Launcher.

At this scope, there are two rough way of solving the problem:
- Have everyone on the team have a programmer role, even when they are content creators. This is
sometimes referred as "making artists compile the project". Roughly this means that content creators
will install Visual Studio and other needed tools, open the solution and start the editor from there.
    - This is simple and can be the way to go for small projects, when compiling times are small and
      the build pipeline is simple. It is not the nicest workflow for non-programmers though.
- Distribute the project compiled artifacts (eg. DLLs) into the VCS, which content creators can then
  sync. Normally this is done by programmers via a script that finds the changed .dll files and
  adds them to the changelist.
    - This will permit a workflow for content creators which is similar to the Content-Only project.
    - The downside is that any programmer that forgets to put the .dlls will leave the project in an
      inconsistent state which can be hard to debug. Also programmers would compete on merging these
      files, which can be annoying.
        - For these reason, a better approach is to have automation that will "publish" the .dlls
          after each submission by programmers. Some care have to be done to coordinate assets with
          the code. Normally submitting the code first and assets later, since the .dlls are
          normally distributed after the code submission (not together).

Roles needed:
- Content Creator
- Game Programmer

Tools needed:
- Epic Launcher
- C++ Toolchain (normally Visual Studio on Windows)
- VCS: Perforce, Git, etc.

# Modified Engine

This is the most complex situation, normally taken by bigger studios, since it does require some
work to get it working correctly. This is mostly because we cannot rely on Unreal Launcher anymore
to distribute the prebuilt engine for us.



