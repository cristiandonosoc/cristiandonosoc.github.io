---
title: "X Macros"
date: 2025-09-17T20:12:58-04:00
draft: false
toc: false
images:
tags:
  - cpp
---

When watching the interview to [Billy Basso (Animal Well) on the Wookash Podcast](https://www.youtube.com/watch?v=YngwUu4bXR4),
he mentioned that he relied a lot of [X macros](https://en.wikipedia.org/wiki/X_macro) to generate code for
his project. That got me interested and led to discover a pattern to which I've come to rely a lot
in my latest personal coding. I'm happy with the results for now, but with the caveat that I've used
it on a small (but non trivial) personal project in which I'm the sole programmer for now.

Basically, X macros are a preprocessor macro that defines a table-like format of data. This macro
receives *another* macro as an input, which is then called which each row of data as an input.
Sounds convoluted (and it somewhat is) but it is actually a really powerful idiom for code
generation.

{{<admonition note>}}
In my code I went all in on them, but a lot of the things that can be done with X macros can be done
with other tools in the language.

Some things can only be done with X macros though.
{{</admonition>}}

But really, X macros become clearer with examples.

## Generate an enum.

Say you're working on an entity system for a game, and you want to define some types of entities.
You can define the enum by hand, or you can use the X macros to generate it. If you're just going to
use X macros to generate an enum, that is *clearly* overkill, but we will build on the example.

```cpp
// We define a macro that receives `X` as an input.
// `X` is another macro that is meant to consume each row.
#define ENTITY_TYPES(X) \
    X(Player)           \
    X(Enemy)            \
    X(Projectile)       \
    X(Building)
```

Then to use the macro, we need to define the input macro. You can create an adhoc one, but the idiom
normally generates one named X that is immediately undefined:

```cpp
// entity.h

// Define the input macro.
// In this case it just generates an entry in an enum.
#define X(ENUM_NAME, ...) ENUM_NAME,

// Call the X macro table after the enum definition.
// I added an Invalid and COUNT types, but that is not necessary for the example.
enum class EEntityTypes : u8 {
    Invalid = 0,
    ENTITY_TYPES(X)
    COUNT,
};

// We undefine the macro so that we don't have problems with re-definition.
#undef X
```

If we run the compiler, the resultant enum will be:
```cpp
enum class EEntityTypes : u8 {
    Invalid = 0,
    Player,
    Enemy,
    Projectile,
    Building,
    COUNT,
};
```

{{<admonition note>}}
Note that the macro has a `, ...` in the end, even though we only have one value. This will become
evident in the future, but this is to make each X macro future proof when we add more arguments.
{{</admonition>}}

That is a lot of overhead and ugly things to define/undefine for a lousy macro! At this point, you
should be thinking that this pattern and I are stupid. Otherwise you're the stupid one.

## Generate enum to string.

An annoying thing that C++ doesn't have is automatic enum to string. That is such a simple and
*very common* case that there is every reason to special case it in the language. But in their
"wisdom", the C++ committee follows their mantra of `Make the (im)perfect enemy of the practical`,
aka. "lets make sure our API covers every case under the sun, so that everything is super complex to
do, regardless of relevance". It has taken decades and there is still not this little useful feature,
and it will put into what is sure to be a monster feature/api that is reflection in C++26.

In the meantime, people have gone to stupid lengths for such a simple feature. Many projects have
code generator that detects enum and generates code (this is what Unreal does with UENUM). Me,
before this, I would just bite the bullet and write a function with a switch case, relying on the
warning that not all case are handled.

But with our trusty X macro, we can just generate an enum that will do it for us.

```cpp
// Entity.h

// After our defining of EEntityType

// Normally I would use std::string_view, but let's keep the example simple.
const char* ToString(EEntityType entity_type);

// Entity.cpp

const char* ToString(EEntityType entity_type) {
    // Define the X macro that will generate the case of the enu,
#define X(ENUM_NAME, ...) case EEntityType::ENUM_NAME: return #ENUM_NAME;

    // Call it as part of a switch case.
    switch (entity_type) {
        ENTITY_TYPES(X)
        // Special case the other.
        case EEntityType::Invalid: return "<invalid>";
        case EEntityType::COUNT: return "<count>";
    }

#undef X

    ASSERT(false);
    return "<unknown>";
}
```

and voila! That generates a switch case that generates the correct name for us

```cpp
    switch (entity_type) {
        case EEntityType::Player: return "Player";
        case EEntityType::Enemy: return "Enemy";
        case EEntityType::Projectile: return "Projectile";
        case EEntityType::Building: return "Building";
        // Special case the other.
        case EEntityType::Invalid: return "<invalid>";
        case EEntityType::COUNT: return "<count>";
    }

```

And this it will be always up to date: the moment we add another row, this enum will be updated.

{{<admonition important>}}
Note that we can inject the `ENTITY_TYPES` macro anywhere. We can have it inside an enum declaration,
inside a switch statement, anywhere really. This is the powerful concept, since we can use it to
generate arbitrary tokens.
{{</admonition>}}

## Generate entity arrays

We can take this pattern way longer, by defining multiple "columns" to our data table.
Let's say the next problem is to generate a struct that will hold an array of pre-allocated
entities. We also want to define how many of these should be pre-allocated.

We can expand our macro to do the following:

```cpp
// Entity.h

// Row format: (ENUM_NAME, STRUCT_NAME, MAX_COUNT)
// Note that we could generate the struct name from the enum name, but there are always exceptions
// and with this is easy to define those.
#define ENTITY_TYPES(X)                     \
    X(Player, PlayerEntity, 2)              \
    X(Enemy, EnemyEntity, 256)              \
    X(Projectile, ProjectileEntity, 1024)   \
    X(Building, BuildingEntity, 32)

// Generate the holder.
// You can define the X macro inline the struct or out, whatever is clearer to you.
#define X(ENUM_NAME, STRUCT_NAME, MAX_COUNT, ...) \
    std::array<STRUCT_NAME, MAX_COUNT> ENUM_NAME##Entities;

struct EntityManager {
    ENTITY_TYPES(X)
};

#undef X
```

And with this we generated the following code, that will be in sync with the enum and the numbers.

```cpp
struct EntityManager {
    std::array<Player, 2> PlayerEntities;
    std::array<Enemy, 256> EnemyEntities;
    std::array<Projectile, 1024> ProjectileEntities;
    std::array<Building, 32> BuildingEntities;
};
```

Now we're getting somewhere! If you like ECS-like systems, you might be liking what this is starting
to look like.

{{<admonition important>}}
Something that is easy to miss here is that those types _have_ to be defined here in order to be
used, since we are defining arrays and therefore the compiler needs to know the size. This means
that using this technique as is will require you to import the declarations of all the types of
entities into this header, which might make it quite big.

There are ways around this, if you're willing to jump through some indirections or other hoops.
{{</admonition>}}

## Typed getters

The above code generation is likely the coolest kind of thing you can get, since it is not really
easy to get something like that with templates without jumping through a lot of horrible hoops,
making the code quite difficult to understand and compile. This is not that easy to read, but it's
not that hard either.

A great advantage of this pattern vs templates is that you can hide a lot of the complexity in the
.cpp file, keeping your headers cleaner.

Say we had an `EntityID` type that encodes a reference to an entity. It would be cool to just be
able to get the entity by type, similar to what `Cast` would do in Unreal. We totally can! And
keeping the complexity in the header. We just need a helper function:

```cpp
// Entity.h

// For getters you can either go template or generate an entry per entity type!
class EntityManager {
 public:
    // Here we assume that our T are entities that have a `StaticType()` function defined in them, but
    // that is optional. We could use concepts to enforce that as well.
    template <typename T>
    T* GetEntity(EntityID id) {
        ASSERT(T::StaticType() == id.GetType());
        return (T*)(GetEntityOpaque(id));
    }

    // Or we can generate a specific function for all of them!
    // This will generate GetPlayerEntity, GetEnemyEntity, etc.
#define X(ENUM_NAME, STRUCT_NAME, ...)              \
    STRUCT_NAME* Get##ENUM_NAME##Entity(EntityID id)    \
        return (STRUCT_NAME*)GetEntityOpaque(id);      \
    }
#undef X

 private:
    // Helper function that is not meant to be called directly. We sadly need it defined in the header.
    // You can use internal namespaces or other mechanisms, or make it part of the public API.
    void* GetEntityOpaque(EntityID id);


    // The X macro generation of the arrays would be here.
    ...
};

```

{{<admonition important>}}
Yeah, I use C-style casts. Bite me.
{{</admonition>}}

With this we can either call `GetEntity<Player>(id)` or `GetPlayerEntity(id)` that will return a
player if the id is valid (and of that type) or `nullptr` otherwise. The cool thing is that all
the complexity is now hidden in the .cpp, which we could not do with templates:

```cpp
// Entity.cpp

// Yolo implementation without bounds checking, lifetime, etc..
// This is for example purposes.
void* EntityManager::GetEntityOpaque(EntityID id) {

#define X(ENUM_NAME, STRUCT_NAME, ...) \
    case EEntityType::ENUM_NAME: return &ENUM_NAME##Entities[id.GetIndex()];

    switch (entity_type) {
        ENTITY_TYPES(X)
        case EEntityType::Invalid:
            ASSERT(false);
            return nullptr;
        case EEntityType::COUNT:
            ASSERT(false);
            return nullptr;
    }

    ASSERT(false);
    return nullptr;

#undef X
}
```

And that's it! With this we have a nice types interface that will get automatically the correct
instance from the correct id. Of course there is a lot of error checking to be done to transform
this to a usable system, but the core of the technique is there and can be expanded. And again, the
complexity will be hidden in the .cpp and the header can be held quite lean.

## Drawbacks

Of course not everything is giggles and rainbows. There are important considerations when using this
technique that you must evaluate if you want to use this technique.

My mitigation to these drawbacks is pretty the same always: I strive to keep my usage to X macros
roughly for the use cases stated above and as simple as possible, not really going into much more
complex. I feel I got a lot of the flexibility and keeps the generation macros manageable.

### constant define/undef of macros / Unorthodox / Annoying to use

This is annoying boilerplate that looks very weird and foreign when not used to the pattern. It is
ugly, to be honest. But if you use this sparingly it's not too bad.

I do believe that we get a lot of mileage out of the little macros we wrote, and I think the ROI is
positive. But **you are writing macros**, which is not the nicest thing. They are hated for a reason.
Editors don't do a great job of expanding them, you normally lose all tools of syntax highlighting,
indenting, etc.

I always choose to define the same macro name `X` so that if I forget to undefine it, I will get a
redefinition error the next time I use the pattern, so I don't really have a problem forgetting to
undefine. But it is annoying.

### X macros cannot pick the columns they want

Because of a limitation of macros, you can only ignore all "next" arguments after the one you want,
by using `, ...`. But before that, you will need to correctly define all the arguments until you
get the one you want.

In this example, say you were only interested in the `MAX_COUNT`, you would still need to get the
other ones `#define X(ENUM_NAME, STRUCT_NAME, MAX_COUNT, ...)`.

This also has the consequence that once you define a column, it is stuck in place forever, since
all usages rely on that order. You can add columns, but never remove. This likely can become a
maintenability concern, so something to keep in mind before you use them in your project that is
meant to be developed by dozens or hundreds of engineers.

### Debuggability

This is not really exclusive to X macro, but really a property of any code generation solution: the
code that you're writing is not _technically_ the one that the compiler is seeing, so you need a way
to debug any problems. Basically you need a way to see what code is generated when you have
problems.

This is straightforward, just compile your file with the "only run preprocessor" flag (`-E` for
gcc/clang, `/E` for MSVC). It might be a bit harder to hook this into your build system, but overall
it should not be that hard to get this output on demand. If it is very difficult, reconsider your
build system.

{{<admonition note>}}
Be prepared that the output will be several hundreds of thousands of lines because just including
one C++ header brings heaps and heaps of code. A very sad reality. But normally your code is easy to
find so it's not that much of a problem.
{{</admonition>}}

## Conclusion

I find this pattern awesome. In particular the fact by two factors:
1. It is kept always in sync by definition. No more "remember to edit this file when editing this"
   crap... To a limit of course, but the limit is surprisingly far.
2. No need for external tooling. There are clear limits to what you can generate, but you get a lot
   of value without the need to have an extra build step. Of course you won't get full blown
   reflection with this, but the value is there for sure.

