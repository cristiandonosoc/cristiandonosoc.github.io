---
title: "Bazel Target Specific Options"
date: 2025-02-23T19:42:28-05:00
draft: true
toc: false
images:
tags:
  - bazel
  - cpp
---

I'm currently working on a little C++ project and I'm using Bazel as my build system. This project
has a couple of libraries, some that I bring as pre-compiled libraries, others I compile from
source.

As this project is mostly for fun, I haven't bothered with doing a "real" Bazel toolchain setup and
instead have decided to do the most brain dead thing I could do to get going with Bazel. In
particular:
1. Have Bazel discover the toolchain (installed MSVC in my case).
2. Have a global repository of flags I pass to the compiler.

Number 2 I get it done using the `.bazelrc` file at the root of my project:

```
### These will apply to any `bazel build` call.

# Enable a lot of warnings.
build --cxxopt="/W4",
# Warnings are errors.
build --cxxopt="/WX",
# No implicit switch fallthrough.
build --cxxopt="/we5262",
build --cxxopt="/std:c++20"
build --cxxopt="/FC"
build --linkopt="/debug"
build --compilation_mode=dbg
build --strip=never

### These will apply to any `bazel test call.

# For some reason Bazel doesn't output stdout/stderr from failing tests by default...
test --test_output=errors
# Make Bazel not complain about sharding.
test --test_env=GTEST_SHARD_STATUS_FILE=
```

And it works great and I can go by in my merry way.

## Third Party Libraries

The problem with global project flags is that well... they're global. Specifically, they apply as
well to third party code. And my case, the first 3 options were causing third party code to fail to
compile, since it would seem I have a much higher standard of hygiene than other code bases (for me
number 3 is non negotiable).

So I need to have a way to have some flags for my project and other for my third party libraries.
Now, the most brain dead thing is to add them to each target. There is an [option for that](https://bazel.build/reference/be/c-cpp#cc_library.cxxopts).

But that is TOO brain dead, since I don't want to be copying options around and remembering to have
these options arrays.

The compromise I found is to wrap the targets of my project with another specific target that will
add the specific flags I need. That way third party code compiles with vanilla targets and I get to
be as custom as I need without messing with individual targets.

And turns out that it is very easy:

```bzl
load("@rules_cc//cc:cc_binary.bzl", "cc_binary")
load("@rules_cc//cc:cc_library.bzl", "cc_library")

_ADDITIONAL_COPTS = [
    "/W4", # Enable a lot of warnings.
    "/WX", # Warnings are errors.
    "/we5262", # No implicit switch fallthrough.
]

def kdk_cc_library(**kwargs):
    kwargs["cxxopts"] = kwargs.get("cxxopts", []) + _ADDITIONAL_COPTS
    cc_library(**kwargs)

def kdk_cc_binary(**kwargs):
    kwargs["cxxopts"] = kwargs.get("cxxopts", []) + _ADDITIONAL_COPTS
    cc_binary(**kwargs)
```

In this case, I simply wrap over `cc_library` and `cc_binary` to add my specific options to my
targets only. That way I can use them in my targets in a straight-forward manner:

```bzl
load("//kandinsky:cc_rules.bzl", "kdk_cc_library")

kdk_cc_library(
    name = "core",
    hdrs = [
        "color.h",
        "defines.h",
        "math.h",
        "memory.h",
        "print.h",
    ],
    srcs = [
        "color.cpp",
        "math.cpp",
        "memory.cpp",
        "print.cpp",
    ],
    deps = [
        "//third_party:glm",
        "//third_party:stb",
    ],
    visibility = ["//kandinsky:__subpackages__"],
)
```

Note the `kdl_cc_library` instead of `cc_library`.

Now, to be able to do this, you need to bring [rules_cc](https://github.com/bazelbuild/rules_cc) as
a dependency. Luckily, in the new brave bzlmod days, this is very easy. Simply add the following
line to your `MODULE.bazel` file:

```bzl
bazel_dep(name = "rules_cc", version = "0.1.1")
```

And you're done.

I know that the "correct" way to do this is *most* likely not this one, but this has worked fairly
well for me, so I thought someone might benefit from this neat little trick.


