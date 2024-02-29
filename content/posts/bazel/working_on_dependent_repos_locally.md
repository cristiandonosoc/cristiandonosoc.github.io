---
title: "Working on multiple Go repos locally with Bazel"
date: 2024-02-17T14:59:44-05:00
draft: false
toc: true
images:
tags:
  - bazel
  - go
---

Some time ago I finished a relatively big (for a personal project) Go project.
An interesting thing about the project is that, while it builds with the normal Go toolchain, I use
Bazel since the project has a strong inter-language component: the tool is actually a code generator
for state machines, currently targeting C++. So I use Bazel to create end-to-end tests that build
the tool, generate C++ code, links it against gtest code and verify that everything is working
dandy.

If you have ever worked with Bazel and Go, you likely have to deal with [Gazelle](https://github.com/bazelbuild/bazel-gazelle),
which is a tool that translates Go native package dependencies (mostly the go.mod file) into Bazel's
BUILD.bazel files and go rules. I am **not** a fan of Gazelle and all the complexity that comes with
it, but if you want Bazel and Go, it's likely the way to go.

For the most part this worked fairly well until I wanted to start a new project that would benefit
from some packages that reside in the `internal` packages of the first project. Mostly helper
packages to deal with files, common utilities, that sort of stuff.

So the use case would look a little like the following. From this module:
```bash
github.com/cristiandonosoc/initial_project
```

I would like to be able to arrive at something like the following:
```bash
github.com/cristiandonosoc/common_lib
  -> github.com/cristiandonosoc/initial_project
  -> github.com/cristiandonosoc/second_project
```

## Native Go

Turns out that separating a go module into two is not that straightforward because there is a
chicken and egg thing. Most likely you will require initial set of changes in **both** the original
`initial_project` and in the newly created `common_lib` when branching out.

You could create `common_lib`, publish into github (or somewhere else) with tags and then bring
those changes into `initial_project`, but that is very cumbersome, since there is likely a big
initial back and forth. So having them locally in the same machine and being able to edit them both
at the same time feels like a better developer experience. Once the `common_lib` is more mature we
could adopt a more "downstream" versioning setup.

Luckily Go already has something for this, the [replace](https://go.dev/ref/mod#go-mod-file-replace)
directive for go.mod files. This permits to redirect a particular package that has been required to
another location/version. This even permits to redirect it to local path! So we can do something
like this and being able to have our main project depend on a local module:

```go.mod
// In initial_project go.mod.
module github.com/cristiandonosoc/initial_project

go 1.21

require(
    ...
    github.com/cristiandonosoc/common_lib v0.1.0  // Or whatever version.
    ...
)

// Override the common_lib module with a local path (a Windows one in this case).
replace github.com/cristiandonosoc/common_lib => D:\projects\go\common_lib
```

You can also replace with another url, check the replace doc for more data. But with this Go will
do the correct thing and being able to find the code. Now you can iterate locally on both modules
and when you're done, you can submit `common_lib` and then remove the replace statement to make Go
find the actual published code.

## Bazel & Gazelle

I've found that everything I want to do with normal Go takes 10x the time with Gazelle. The whole
experience is riddled with weird errors and frustration. I do not enjoy it.

In any case, after quite a bit of googling, asking around and fiddling, I managed to find that you
can define `go_repository` rules that point to a local repo holding the module. This is less
flexible than Go since you have to define a commit hash, meaning that if you want to "publish"
changes from one local repo to the other, you have to commit in the first and update the hash in the
other. In Go it would just work with the local files, as you would expect from a tool designed for
actual humans.

But given how awful I find the Gazelle experience, I gladly take this L.


My process looks like the following:

1. Use Gazelle to generate all the known go_repositories into a file called deps.bzl with a
   `go_dependencies` function:

```
gazelle update-repos -from_file=go.mod -to_macro=deps.bzl%go_dependencies
```

This will generate a function that will add all the things in your go.mod as `go_repository` calls
within the `deps.bzl` file. Now, your local repo will not be there, or will be there in a screwed
up fashion. But don't worry, we will create a local entry.

{{< admonition important >}}
Gazelle is not very clever updating these entries, so when you change versions or replace statements
in your go.mod (or really when you change anything in your go mod), I normally truncate this file
and let Gazelle regenerate everything. Basically give the tool a fighting change of actually doing
its job.
{{< /admonition >}}


2. I create a `local_deps.bzl` file with my local repos:

```bzl
load("@bazel_gazelle//:deps.bzl", "go_repository")

# These are local dependencies that we can activate on and off.
def go_local_dependencies():
    go_repository(
        name = "com_github_cristiandonosoc_common_lib",
        importpath = "github.com/cristiandonosoc/common_lib",
        vcs = "git",
        remote = "D:\\projects\\go\\common_lib",
        commit = "80a15b903f6b3c7f4e5bf834cdcb757e95f1c19f",
    )

```

{{< admonition note >}}
As stated before, the commit hash will need to be updated if you want to "publish" local changes
into this repo.
{{< /admonition >}}

We can comment/uncomment this file depending on whether we want to work with the local repo or not,
similar to activating the `replace` entry in the go.mod. You can use `pass` to create an empty
function in Bazel, which makes it easy to just comment the go_repository entries:

```bzl
def go_local_dependencies():
    pass
    #go_repository(
    #    name = "com_github_cristiandonosoc_common_lib",
    #    importpath = "github.com/cristiandonosoc/common_lib",
    #    vcs = "git",
    #    remote = "D:\\projects\\go\\common_lib",
    #    commit = "80a15b903f6b3c7f4e5bf834cdcb757e95f1c19f",
    #)
```

3. In my WORKSPACE file, after all the expected Gazelle nonsense, I load a new file that describes
my local repositories:

And with that, and some fiddling and running Gazelle a couple of times until the deps are up to
date, you should be able to have Bazel find your local repo. Once you're done and the common lib has
been submitted, you should be able to return to a normal flow and hate Gazelle just a bit less.
