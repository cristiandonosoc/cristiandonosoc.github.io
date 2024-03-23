---
title: "Unexpected Copy Contructors on MSVC"
date: 2024-03-23T11:41:38-04:00
draft: false
toc: true
images: ["bg/cohen.jpg"]
tags:
  - cpp
  - unreal
  - msvc
---

This is not _technically_ an Unreal thing, but working on Unreal is the most common way to hit this
issue. It is how I found it. And I managed to find the solution to a hair-removing error because I
*remembered* that a couple of weeks ago one of our engine programmers helped with an unrelated issue
that happened to have the same root cause. But enough preamble...

## The Setup

Say you wanted an array of things that are stable in memory. This means that as long as those
elements are alive, they will keep the same address in memory, so you can use a keep a non-owning
pointer to them (as long as you ensure that you're well aware of the lifetime, of course). There are
many ways to achieve this, but likely the simplest of this would be a vector of unique pointers:
`std::vector<std::unique_ptr<Foo>>`.

Let's make the example a bit more expressive and create some C++ structs for this. I'm going to be
using the excellent https://godbolt.org to show my work throughout.

```cpp
#include <memory>
#include <vector>

struct Element {
    int Value = 0;
};

struct ElementManager {
    std::vector<std::unique_ptr<Element>> Elements;
};
```
godbolt: https://godbolt.org/z/8oGe4q65v

So far so good, and we can see that the code compiles, as it should.

## No Copying

Now, unique pointers are, well as they name suggests... unique. This means that they cannot be
copied, thus they have their copy constructors deleted. In the code above we didn't try to copy
the anything, like the vector or the `ElementManager` itself, so there is no compiler error. Since
nothing has tried to copy those things, those functions are not generated and thus the compiler is
happy. But if we tried copying stuff we would get that error:

```cpp
#include <memory>
#include <vector>

struct Element {
    int Value = 0;
};

struct ElementManager {
    std::vector<std::unique_ptr<Element>> Elements;
};

ElementManager CopyManager(const ElementManager& other) {
    ElementManager new_manager;
    new_manager = other;    // Copy happens here.
    return new_manager;
}
```
godbolt: https://godbolt.org/z/vnooo4xjK


The error logs would be:
```bash
example.cpp
C:/data/msvc/14.39.33321-Pre/include\vector(1420): error C2280: 'std::unique_ptr<Element,std::default_delete<Element>> &std::unique_ptr<Element,std::default_delete<Element>>::operator =(const std::unique_ptr<Element,std::default_delete<Element>> &)': attempting to reference a deleted function
C:/data/msvc/14.39.33321-Pre/include\memory(3289): note: see declaration of 'std::unique_ptr<Element,std::default_delete<Element>>::operator ='
C:/data/msvc/14.39.33321-Pre/include\memory(3289): note: 'std::unique_ptr<Element,std::default_delete<Element>> &std::unique_ptr<Element,std::default_delete<Element>>::operator =(const std::unique_ptr<Element,std::default_delete<Element>> &)': function was explicitly deleted
C:/data/msvc/14.39.33321-Pre/include\vector(1420): note: the template instantiation context (the oldest one first) is
<source>(9): note: see reference to class template instantiation 'std::vector<std::unique_ptr<Element,std::default_delete<Element>>,std::allocator<std::unique_ptr<Element,std::default_delete<Element>>>>' being compiled
C:/data/msvc/14.39.33321-Pre/include\vector(1480): note: while compiling class template member function 'std::vector<std::unique_ptr<Element,std::default_delete<Element>>,std::allocator<std::unique_ptr<Element,std::default_delete<Element>>>> &std::vector<std::unique_ptr<Element,std::default_delete<Element>>,std::allocator<std::unique_ptr<Element,std::default_delete<Element>>>>::operator =(const std::vector<std::unique_ptr<Element,std::default_delete<Element>>,std::allocator<std::unique_ptr<Element,std::default_delete<Element>>>> &)'
<source>(10): note: see the first reference to 'std::vector<std::unique_ptr<Element,std::default_delete<Element>>,std::allocator<std::unique_ptr<Element,std::default_delete<Element>>>>::operator =' in 'ElementManager::operator ='
C:/data/msvc/14.39.33321-Pre/include\vector(1496): note: see reference to function template instantiation 'void std::vector<std::unique_ptr<Element,std::default_delete<Element>>,std::allocator<std::unique_ptr<Element,std::default_delete<Element>>>>::_Assign_counted_range<std::unique_ptr<Element,std::default_delete<Element>>*>(_Iter,const unsigned __int64)' being compiled
        with
        [
            _Iter=std::unique_ptr<Element,std::default_delete<Element>> *
        ]
Compiler returned: 2
```

What a great help this compiler is! Might as well print the compiler stack trace while it's at it.
But it does, in a very convoluted way, say that it's trying to:
1. use the `operator=` of the `std::unique_ptr`...
2. by trying to use `operator=` of `std::vector`...
3. by trying to use `operator=` of `ElementManager`.

This is an error, since the copy assignment `operator=` is deleted for unique pointers.

{{<admonition note >}}
See how the reference to the copy line (would be `<source>(14)`) is not referenced nowhere? You
kinda have _know_ what code prompted the compiler to do stuff. This is a theme with C++, and
specially with MSVC and will come back on this post.
{{</admonition>}}

The error is correct, since unique pointers cannot be copied. So there should not be any surprises
here.

## Unreal Enters the Battlefield

If you have programmed in Unreal, you have seen their `<MODULE>_API` markers. You would normally put
them at the start of your class (say your module is called `MyModule`):

```cpp
UCLASS()
class MYMODULE_API AMyActor : public AActor {
    ...
};
```

Well these makers actually [Module API Specifiers](https://dev.epicgames.com/documentation/en-us/unreal-engine/module-api-specifiers-in-unreal-engine)
which Unreal uses to specify what part of the code is part of the "Module API". Basically, what part
of the code of that Module is available to be used by other modules.


When you're programming in Unreal, you use their concepts of [Unreal Modules](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-engine-modules)
which are ways to compartmentalize your code into different independent sources that can link of
each other. You can _roughly_ think of them of each being a separate `.dll` (or `.so` for the Linux
folks, or `cc_library` for the Bazel folks). They have all kinds of specs of what modules can
depend to the modules, when they are loaded, etc. This is controlled by their subpar (as build
systems goes at least) [Unreal Build Tool: (UBT)](https://dev.epicgames.com/documentation/en-us/unreal-engine/unreal-build-tool-in-unreal-engine)
and the Engine itself (for things like runtime module loading).

Note that "tagging" code is a very Windows/Microsoft thing, since on the Linux side, while these
kind of mechanisms exist, they are not normally sprinkled through the code. I remembered being
confused and annoyed that I had to do the compiler's work for it, by stating what I want to have
available for linking, since I had worked on big C++ code bases (eg. Chrome) and we didn't need to
"tag" our classes so that compiler could "know" what should be available.

But anyway, let's say we wanted to use our `ElementManager` in Unreal, and say we wanted to expose
it to other modules:

{{<admonition note>}}
I added a "class" interface on it so that people don't say "but what functions are you exposing?"...
`structs` have implicit functions too, but for explicitness sake, I added a little "interface". It
doesn't change a thing, as I will show later anyway.
{{</admonition>}}

```cpp
struct Element {
    int Value = 0;
};

class MYMODULE_API ElementManager {
public:
    // Some crappy api.
    Element* AddElement(int value);
    void RemoveElement(int value);

private:
    // We can use TArray and TUniquePtr... It will fail also.
    // This problem is not a vector/unique_ptr thing.
    std::vector<std::unique_ptr<Element>> Elements;
};
```

By writing that, I would expect other modules to be able to use `ElementManager`. But turns out I
get the compilation error! This one I tried on an Unreal project of mine with Rider:

{{<admonition important>}}
As the comment says, this would've happened as well if you used Unreal containers, so `TArray`
instead `std::vector` and `TUniquePtr` instead of `std::unique_ptr`. The problem is not that, and I
decided to stick to the same containers so that I don't have to show a different error log.
{{</admonition>}}

```bash

10>vector(1420): Error C2280 : 'std::unique_ptr<ElementManager,std::default_delete<ElementManager>> &std::unique_ptr<ElementManager,std::default_delete<ElementManager>>::operator =(const std::unique_ptr<ElementManager,std::default_delete<ElementManager>> &)': attempting to reference a deleted function
10>memory(3320): Reference C2280 : see declaration of 'std::unique_ptr<ElementManager,std::default_delete<ElementManager>>::operator ='
10>memory(3320): Reference C2280 : 'std::unique_ptr<ElementManager,std::default_delete<ElementManager>> &std::unique_ptr<ElementManager,std::default_delete<ElementManager>>::operator =(const std::unique_ptr<ElementManager,std::default_delete<ElementManager>> &)': function was explicitly deleted
10>vector(1420): Reference C2280 : the template instantiation context (the oldest one first) is
10>ARCharacter.h(37): Reference C2280 : see reference to class template instantiation 'std::vector<std::unique_ptr<ElementManager,std::default_delete<ElementManager>>,std::allocator<std::unique_ptr<ElementManager,std::default_delete<ElementManager>>>>' being compiled
10>vector(1480): Reference C2280 : while compiling class template member function 'std::vector<std::unique_ptr<ElementManager,std::default_delete<ElementManager>>,std::allocator<std::unique_ptr<ElementManager,std::default_delete<ElementManager>>>> &std::vector<std::unique_ptr<ElementManager,std::default_delete<ElementManager>>,std::allocator<std::unique_ptr<ElementManager,std::default_delete<ElementManager>>>>::operator =(const std::vector<std::unique_ptr<ElementManager,std::default_delete<ElementManager>>,std::allocator<std::unique_ptr<ElementManager,std::default_delete<ElementManager>>>> &)'
10>ARCharacter.h(38): Reference C2280 : see the first reference to 'std::vector<std::unique_ptr<ElementManager,std::default_delete<ElementManager>>,std::allocator<std::unique_ptr<ElementManager,std::default_delete<ElementManager>>>>::operator =' in 'ElementManager::operator ='
10>vector(1496): Reference C2280 : see reference to function template instantiation 'void std::vector<std::unique_ptr<ElementManager,std::default_delete<ElementManager>>,std::allocator<std::unique_ptr<ElementManager,std::default_delete<ElementManager>>>>::_Assign_counted_range<std::unique_ptr<ElementManager,std::default_delete<ElementManager>>*>(_Iter,const unsigned __int64)' being compiled
        with
        [
            _Iter=std::unique_ptr<ElementManager,std::default_delete<ElementManager>> *
        ]
```

No one is trying to copy this class. In fact, **no one is using this class** and it still fails to
compile. What is going on?

## Microsoft and dllspec

If you looked at [Module API Specifiers](https://dev.epicgames.com/documentation/en-us/unreal-engine/module-api-specifiers-in-unreal-engine)
they say that it translates to `dllimport/dllexport` statements. As I stated before, this is a very
Microsoft thing, to the point that this is literally using a MSVC extension: [dllexport](https://learn.microsoft.com/en-us/cpp/build/exporting-from-a-dll-using-declspec-dllexport?view=msvc-170)

Roughly the `<MODULE>_API` macro does is put the `__declspec(dllexport)` which instruct the
compiler to "export" all the non-deleted function of that class (or function if you put in on the
function level) out to the DLL (aka. module).  The key part here is the **non-deleted** functions.

So the compiler happily goes over `ElementManager` and generate entry points for those functions as
it might be that some other external DLL might want to call it. That includes the copy constructors.
This in turn instructs the compiler to instantiate that code, which leads to the compiler error of
trying to down the line the deleted `operator=` of the unique pointer.

We don't need to use Unreal to corroborate this, we can just use the __declspec directly:

```cpp
#include <memory>
#include <vector>

struct Element {
    int Value = 0;
};

struct __declspec(dllexport) ElementManager {
    std::vector<std::unique_ptr<Element>> Elements;
};
```
godbolt: https://godbolt.org/z/obv7sh3vs

This gives the same error. This is a rare issue because normally (at least in my life experience)
you don't go around exporting DLL tags... expect that with Unreal you do... **all the time!** Only
that you don't explicitly do so, but rather by using their `<MODULE>_API` macros.

## The Fix

The fix in this case is as sad as it is straightforward: make the problematic functions deleted so
that the compiler won't generate them:

```cpp
#include <memory>
#include <vector>

struct Element {
    int Value = 0;
};

struct __declspec(dllexport) ElementManager {
    // Delete copy constructor/assignment.
    ElementManager(const ElementManager&) = delete;
    ElementManager& operator=(const ElementManager&) = delete;

    std::vector<std::unique_ptr<Element>> Elements;
};
```
godbolt: https://godbolt.org/z/E8fWP3bPc

And now it compiles!

## Conclusion

By far the worst part of all this is that the error log is **not useful** to find the
error. While it is _technically_ correct, as the failure is that some code is trying to use a copy
operator for an unique pointer, there is nothing on the code that tells you why this code is being
generated, or from where.

You have to **LITERALLY** know this to snuff that this is happening. Try googling this error,
try telling chatGPT about this... You're fucked, as it all points to the more common error of you
trying to copy an uncopyable thing. How do I know this? Because I tried for a while, even asking
for help to people who know **WAY** more C++ than me and we couldn't figure out the error until I
happened to remember that it _might_ be this `<MODULE>_API` nonsense.

I looked around to see what the MSVC folk said and of course the answer given is what you would
expect from Compiler people: https://developercommunity.visualstudio.com/t/compilation-fails-for-non-movable-type-passed-to-c/309335

{{<figure src="images/posts/technically_correct.png" >}}

The important thing is not only what the error is, but **WHY** it is happening. In this case it is
actually more important, since the cause of the error came from an obscure side effect. Personally I
consider this a compiler bug, in the documentation category, since the compiler is failing to aid
the programmer in finding the cause of the error. Error messages are part of the "API" of any
compiler, since error messages are how programmers know how to get it to do the work. And sadly,
MSVC, of all the "major" compilers (the others being GCC and Clang), is by far the one that has the
worst... pretty much everything I would say. But specially error messages.

I don't think this error message will ever change. So as a takeaway, tuck what you read here
somewhere in your mind and remember that your compiler error might be coming from "the ether".













