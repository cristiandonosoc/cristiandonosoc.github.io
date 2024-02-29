---
title: "Gazelle Unexpected Module Loading"
date: 2024-02-28T20:44:38-05:00
draft: false
toc: false
images:
tags:
  - bazel
  - go
---

So I was going on my merry way, Go-ing around when I wanted to some of the fancy (not so new by now)
`maps` packages functions. In particularly I wanted something that is not hard to write, but might
as well use the fanciness: https://pkg.go.dev/golang.org/x/exp/maps#Keys.

Some of the maps stuff is already in standard library but as the time of this post, some of the
functions are in the experimental `x` package, which is like a kitchen sync of stuff the go devs
play around with before they actually add it to the standard library. Use at your own risk and
whatnot.

So I go ahead and add the `golang.org/x/exp/maps` import so I can use the code. Works perfectly when
I do `go test`. Gazelle updates with no problems but when I go to use `bazel test`, I get this weird
error:

```bash
> bazel test //...
INFO: Invocation ID: 310b3dc7-4bed-44d2-9ddf-67bd929fa002
ERROR: D:/projects/go/gochart/pkg/backend/unreal/BUILD.bazel:3:11: no such package '@org_golang_x_exp//maps': BUILD file not found in directory 'maps' of external repository @org_golang_x_exp. Add a BUILD file to a directory to mark it as a package. and referenced by '//pkg/backend/unreal:unreal'
ERROR: Analysis of target '//cmd/gochart:gochart' failed; build aborted:
INFO: Elapsed time: 0.419s
INFO: 0 processes.
ERROR: Couldn't start the build. Unable to run tests
FAILED: Build did NOT complete successfully (24 packages loaded, 237 targets configured)
```

Huh? It is complaining about the `maps` directory within the `golang.org/x/exp` module?
But we used it from go! And if we go check the repo, [it is right there](https://cs.opensource.google/go/x/exp/+/814bf88c:maps/).
So what is going on?

Bazel is not complaining that it doesn't know what the module is, but rather that it cannot find the
correct BUILD file. Why a BUILD file? Because the way [rules_go](https://github.com/bazelbuild/rules_go)
is that, **very roughly**, they get whatever module at the version you specified in your
`go_repository` statements, run Gazelle on it so that it has BUILD files Bazel understands and then
it's good to go.

## Debugging

When debugging errors with Go and Bazel, I follow the following golden rule: `"Blame Gazelle First"`.
So I assume Gazelle imported the wrong version of the `exp` module.

Since this worked in my `go.mod`, lets check what version we had there:

```go.mod
go 1.21.5
...
	golang.org/x/exp v0.0.0-20240222234643-814bf88cf225
...
```

Alright, so `v0.0.0-20240222234643-814bf88cf225` is the version. A great commit, if I can say so.
So, by following the golden rule, we go check to see if Gazelle generated the wrong `go_repository`
statement:

```bzl
    go_repository(
        name = "org_golang_x_exp",
        importpath = "golang.org/x/exp",
        sum = "h1:LfspQV/FYTatPTr/3HzIcmiUFH7PGP+OQ6mgDYo3yuQ=",
        version = "v0.0.0-20240222234643-814bf88cf225",
    )
```

Same version! Gazelle actually came through. This means that Bazel is being instructed to load the
correct version. So what is going on?

## Believe in the Golden Rule

At this moment I'm stumped. People tell me to `ls $(bazel info output_base)/external/org_golang_x_exp`
to see what module is actually there. And indeed, the `maps` directory is not there. It's almost as
if Bazel is pulling down the wrong version of the module. But this is my own WORKSPACE that I
control completely and I haven't used this module before. So _where_ could this another version be
coming from?

And this is where I remember there is a reason we have old traditions and sayings. In particular the
battle tested `"Blame Gazelle First"`. I should have stayed true.

So in my `WORKSPACE` file, this is the way I load Gazelle and the dependencies:

```bzl
# Gazelle ------------------------------------------------------------------------------------------

http_archive(
    name = "bazel_gazelle",
    sha256 = "29218f8e0cebe583643cbf93cae6f971be8a2484cdcfa1e45057658df8d54002",
    urls = [
        "https://mirror.bazel.build/github.com/bazelbuild/bazel-gazelle/releases/download/v0.32.0/bazel-gazelle-v0.32.0.tar.gz",
        "https://github.com/bazelbuild/bazel-gazelle/releases/download/v0.32.0/bazel-gazelle-v0.32.0.tar.gz",
    ],
)

load("@bazel_gazelle//:deps.bzl", "gazelle_dependencies")
gazelle_dependencies()

load("//:deps.bzl", "go_dependencies")
# gazelle:repository_macro deps.bzl%go_dependencies
go_dependencies()

load("//:local_deps.bzl", "go_local_dependencies")
go_local_dependencies()
```

If you see the error, you're a better human being than me. It is technically documented in a
[comment within a code snippet on how to setup Gazelle](https://github.com/bazelbuild/bazel-gazelle?tab=readme-ov-file#running-gazelle-with-bazel).
The error is that when loading `go_repostory`, the first time an external module is stated, that is
the one that wins.

So by loading the `gazelle_dependencies()` first, I'm loading whatever they need first, before my
own packages. And if we go and look at Gazelle deps, we see our [culprit](https://github.com/bazelbuild/bazel-gazelle/blob/master/deps.bzl#L279):

```bzl
    _maybe(
        go_repository,
        name = "org_golang_x_exp",
        importpath = "golang.org/x/exp",
        sum = "h1:c2HOrn5iMezYjSlGPncknSEr/8x5LELb/ilJbXi9DEA=",
        version = "v0.0.0-20190121172915-509febef88a4",
    )
```

Gazelle loads the `exp` package itself. And since the first one wins, this is the one that Bazel is
going to use, compatibility be damned. And why bother _at least_ stating a warning about it. It is
much better to err silently and use a dependency from another version **that the user explicitly
stated**.

## The Fix

The fix is to put your dependencies first so its Gazelle the one who can get screwed. This fixed my
problem but is of course a time bomb, since it can happen that the `exp` module (or whichever other
we happen to share) that I use can be so different version-wise than the one Gazelle needs, that it
might fail to compile or might work erratically.

Personally I find this **very** disappointing, particularly because Go would _never_ do this.
They have a [whole documentation section](https://go.dev/ref/mod#minimal-version-selection)
on this! That the go rules do something as yolo as this is, as I said, a tad disappointing.

I was told that [Bzlmod](https://bazel.build/external/overview#bzlmod) was supposed to do the right
thing, but that seems like a huge can of worms that I do not want to open right now. Gazelle is
enough problem as it is.

So as a conclusion, remember the golden rule. While _technically_ it was a bug I introduced, the
tool whose one of its _main tasks is to bridge Go dependencies and Bazel_ sure made it easy to make. And it
was not fast nor pleasant to find the cause. So yeah... Golden rule.
