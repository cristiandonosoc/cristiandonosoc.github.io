---
title: "Shared Library Hot Reloading (Part 2)"
date: 2025-10-14T14:40:57-04:00
draft: true
toc: true
images:
tags:
  - cpp
  - SDL3
---

In the previous post I did an overview of the process of the hot code reloading, but in this post
I'm going to go into some important details about how to architect and make the technique work.

## Architecting the code

In the previous post I went over how the "main app calls the DLL", but didn't really specify how
this is done. There are several ways about this, but my approach is to create a "manual function
table" that I keep around.

As explained, DLL export a set of function available on it to be called. So the easiest way is to
define on a common header that both the main app and the DLLs will share a function table:

```cpp
struct LoadedGameLibrary {
    SDL_SharedObject* SO = nullptr;
    SDL_Time SOModifiedTime = {};

    bool (*__EntryPoint_OnSharedObjectLoaded)(PlatformState* ps) = nullptr;
    bool (*__EntryPoint_OnSharedObjectUnloaded)(PlatformState* ps) = nullptr;
    bool (*__EntryPoint_GameInit)(PlatformState* ps) = nullptr;
    bool (*__EntryPoint_GameUpdate)(PlatformState* ps) = nullptr;
    bool (*__EntryPoint_GameRender)(PlatformState* ps) = nullptr;
};
```

This struct will be owned by the main program, and will call this functions at the correct moment.
In the context of a program, it is a game-like program, so most of the functions are around the game
loop (update/render).

{{<admonition note>}}
There is nothing special at all about the naming of these functions, they are exported functions as
any other. I like to mark them as `__EntryPoint_*` so that they are easy to find by searching.
{{</admonition>}}

On the DLL side, we need to make sure we compile these functions and export them in with the C
naming convention, since we're going to search them by name:

```cpp

// This make sure that the exported function name is just the function name, with no nonsense.
extern "C" {

    // Each of this function now can call any code within the DLL.

    bool __EntryPoint_OnSharedObjectLoaded(PlatformState* ps) { ... }

    bool __EntryPoint_OnSharedObjectUnloaded(PlatformState* ps) { ... }

    bool __EntryPoint_GameInit(PlatformState* ps) { ... }

    bool __EntryPoint_GameUpdate(PlatformState* ps) { ... }

    bool __EntryPoint_GameRender(PlatformState* ps) { ... }
}
```

The `extern "C"` is quite important. Try not to put them and use a tool like using
`dumpbin /exports <DLL_PATH>` and see the nonsense the compiler outputs.

### Loading the library

The process of getting it is fairly simple: we can get them by name and get the function pointer:

```cpp
if (SDL_SharedObject* so = SDL_LoadObject(so_path)) {
    SDL_FunctionPointer pointer = SDL_LoadFunction(so, function_name);
}
```

{{<admonition note>}}
I'm using `SDL_LoadObject` and `SDL_LoadFunction`, which wraps over OS primitives details, so it
should be relatively quick to make it work on Linux as well. But using the Windows primitives would
be the same flow.
{{</admonition>}}

To make it easy, I created a macro that helps make it easy to load them:

```cpp
void LoadGameLibrary(LoadedGameLibrary* lgl, String so_path) {
    ...

    lgl->SO = SDL_LoadObject(so_path);
    if (!lgl->SO) {
        SDL_Log("ERROR: Could not find SO at \"%s\"", so_path);
        return false;
    }

#define LOAD_FUNCTION(lgl, function_name)                                           \
    {                                                                               \
        SDL_FunctionPointer pointer = SDL_LoadFunction(lgl->SO, #function_name);    \
        if (pointer == NULL) {                                                      \
            SDL_Log("ERROR: Didn't find function " #function_name);                 \
            return false;                                                           \
        }                                                                           \
        lgl->function_name = (bool (*)(PlatformState*))pointer;                     \
    }

    LOAD_FUNCTION(lgl, __EntryPoint_OnSharedObjectLoaded);
    LOAD_FUNCTION(lgl, __EntryPoint_OnSharedObjectUnloaded);
    LOAD_FUNCTION(lgl, __EntryPoint_GameInit);
    LOAD_FUNCTION(lgl, __EntryPoint_GameUpdate);
    LOAD_FUNCTION(lgl, __EntryPoint_GameRender);

#undef LOAD_FUNCTION

}
```

And with that we have a nice struct with all those entry points ready.

### Unloading the library

Unloading the library is very simple:

```cpp
void Unload(LoadedGameLibrary* lgl) {
    SDL_UnloadObject(lgl->SO);

    // Remember to clear the function pointers!
    *lgl = {};
}
```

## DLL object lifetimes

Shared objects have a particular property in that they own their own memory. This has a lot of
details but you can think that each loaded shared object has mapped to it certain sections of
memory. This is all fine and well, but it also meanstwhen an SO is unloaded, that
memory goes away with it.

** That is a HUGE consequence**

This means that if you intent to unload something and expect it to continue to use it, **you cannot
allocate it in the memory space of that shared object**. C++ actually makes it very easy to make
this mistake, since by default the allocator is implicit:

```cpp

// In main program.

void InMainProgram_GameUpdate(PlatformState* ps, LoadedGameLibrary* lgl) {
    ...
    std::string string_in_main("hello");  // <--- This is allocated in the main.exe memory space
    ...


    lgl->__EntryPoint_GameUpdate(ps);
}

// In the DLL.

void __EntryPoint_GameUpdate(PlatformState* ps) {
    ...
    std::string string_in_dll("hello");  // <--- This is allocated in the dll space.
}
```

The moment we call `SDL_UnloadObject`, the memory contained by `string_in_dll` is invalidated. If
you have a reference to it, now you are having a pointer to undefined memory, which is bad news.
In the case of strings it's easy, but this will apply to any game element: entities, physics, etc.

### Everything allocated in the main program

An option is to "serialize" all the information associated with the game before unloading and when
loading the new DLL, deserialize it and restore the world. While this might work in theory, it can
be a *huge* amount of work and might be very hard to correctly "recreate" the world.

An easier solution is to make sure that the DLL **never allocates any memory... ever**. That way
unloading and loading a DLL is a no-op: we're just reloading the machine code associated with the
functions, but all the data is owned by the main application.

For that we do need to change the way we deal with memory, as the default C++ allocator will allocate
in the heap of the current DLL it is running from.

The approach I use is to use [arena allocators](https://www.rfleury.com/p/untangling-lifetimes-the-arena-allocator),
which basically mean that at the start of the program I allocate a huge amount of memory and then I
have an API to chunk that memory as needed. But the ownership of that memory is never allocated by
the DLL.

For me, this has the consequence that I cannot (easily) use the STL containers, which is a trade off
I chose. But it is also possible to create your custom C++ allocator that is aware of this concept.
Likely the easiest is to re-map the `new/delete (or malloc/free)` functions to map to the ones
associated with the main binary.

TODO PROGRAM

### Globals

Another aspect that is quite important is that C++ global variables are created *per shared object*.
This has huge implications, to the point that you might have duplicated global variables: one for
the main program and another for the DLL. And which one is called depends on where your code is
running. This can create a lot of very weird behaviour.

The main reason this happens is when you compile your libraries statically, meaning that you bundle
the code of your library in the program. While this sounds right, it has this consequence when
loading DLLs.

As an example, take the case of the excellent [Dear Imgui](https://github.com/ocornut/imgui), which
uses some global variables to manage the current state of the UI. Say we might want to have some
part of the main executable visualize some stuff and the DLL as well. If you
link Imgui statically to both targets, your program will end with 2 copies of Imgui, which will
100% break if you try to use naively.

There are two ways of solving this:
1. Always link **every** dependency as a shared object. That means that they will be loaded by the
   runtime linker and you will have one copy of the globals of that dependency. The con of this is
   that now you will have to deploy the dll if you want to run the game.

2. Make sure that you "patch" the globals. That is where the `__EntryPoint_OnSharedObjectLoaded`
   callback is made.

    This function is called _right_ away the DLL being loaded, before any other function is called.
    This is important since other code might rely on correctly set globals. With that you can
    patch the global pointer to point to the correct thing. Well designed libraries also provide
    facilities to do that (like Imgui). As an example, this is what I do in my program.

    ```cpp
    bool __Internal_OnSharedObjectLoaded(PlatformState* ps) {
        platform::SetPlatformContext(ps);

        SDL_GL_MakeCurrent(ps->Window.SDLWindow, ps->Window.GLContext);

        // Initialize GLEW.
        if (!InitGlew(ps)) {
            return false;
        }

        ImGui::SetCurrentContext(ps->ImGuiState.Context);
        ImGui::SetAllocatorFunctions(ps->ImGuiState.AllocFunc, ps->ImGuiState.FreeFunc);

        ImGuizmo::SetImGuiContext(ps->ImGuiState.Context);

        ...
    }
    ```

    In this case, the following things needed to be patched:
    - We use a global PlatformContext for our own game.
    - We need to correctly patch the SDL/OpenGL globals.
    - We need to re-initialize GLEW (some OpenGL stuff) to correclty understand the new globals.
    - We need to patch the Imgui context (global)
        - Also we need to patch the malloc/free functions for the memory ownership
    - ImGuizmo (a dependent guizmo imgui lib) needs to patch the globals as well.

    That is a lot of stuff. If your build process makes it easy to allocate dynamically, it is
    better. But in some cases, like OpenGL, you will need to do it anyway.

{{<admonition note>}}
For me, it seems that during dev/editor time, when you want hot reloading, you should try to link
stuff dynamically as much as you can, to avoid having to track globals. But once your program is
packed to ship, you should just link all statically, so you don't need to ship the dlls.
{{</admonition>}}
