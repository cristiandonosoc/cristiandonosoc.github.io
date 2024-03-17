---
title: "Some Neovim Setup Notes"
date: 2024-03-17T01:22:12-05:00
draft: false
toc: true
images: ["bg/cohen.jpg"]
tags:
  - neovim
---

I have been using Vim for over a decade now. Using Vim bindings is second-nature to me now and I
refuse to program on an editor/tool that does not have Vim bindings. I also get grumpy when the Vim
emulation on those tools doesn't work quite right. For the most part they do OK with the bindings,
but most fail miserably with more advanced stuff such as Macros. Rider, I'm looking at you.

But throughout those times, I have used a relative vanilla Vim config. Like everyone else, I had a
.vimrc that I have accumulated over time, building some idiosyncrasies along the way. Some things
about my setup that are relative uncommon:

- I use "," as my leader.
- In normal mode, I mapped `<space>` to search (basically like pressing '/').
- Since as long as I remember, in normal mode I use `<tab>` to format the file (eg. invoke gofmt).
- This is not _strictly_ a Vim thing, but **of course** I map caps-lock to be `<escape>`.

... and so and so forth.

That is all fine, but in terms of plugins, I have always been fairly austere. Mostly because I
always felt the environment to be very complicated and I didn't felt the need. The one big exception
was [YouCompleteMe](https://github.com/ycm-core/YouCompleteMe), which I used for the longest time to
get C++ (and later Go) completions.

## Neovim Comes In

I have been using Neovim for a while, but really I was using the same as Vim. I was using my same
.vimrc. I really changed because of the integrated terminal, as I am a heavy terminal user.

But this year that changed as I took the plunge and I went full Neovim by creating a native setup,
to get all the goodies that the native Neovim ecosystem marketed so much. So over the course of
several days, in my spare time, I went through pretty much all of this great
[Neovim from Scratch](https://www.youtube.com/playlist?list=PLhoH5vyxr6Qq41NFL4GvhFp-WLd5xzIzZ)
series and got myself a pretty decent setup. For the most part it does more stuff that I had in my
previous setup, though some things are still a bit quirky. Overall, I'm happy... but it is quite a
bit more complex.

## Complexity Galore

{{<admonition note>}}
This is a snapshot of my setup as the time of the writing of this post. It is very likely that a
year or more from now, some of the stuff has changed. But I do think that for the most part, the
overall architecture of things should remain relatively stable.
{{</admonition>}}

{{<figure src="images/posts/neovim/neovim-plugins.png" >}}

Neovim and the whole plugin ecosystem that was created is... impressive, for lack of a better word.
There are some quite cool things that have been done, and has permitted for a lot of people to
provide functionality without being too blocked with each other.

The downside is that there is *a lot* of things involved, even in the _base_ pro setup. Packer,
nvim-cmp, nvim-lspconfig, mason, mason-lspconfig, treesitter are some of the concepts that I've had
to wrangle together to get my setup working. It is not simple. I mean, just look at the diagram
above. It's huge, and this is for an editor **config**.

So I'm writing this post partly for myself, so I can refer back to it the next time I need to change
something not trivial about my setup and have to re-learn what all these things are. So this is my
completely subjective, mostly wrong glossary of the important things of my Neovim setup.

{{<admonition important>}}
These description of the plugins is as _I understand them_. Considering that I'm a lowly user that
does not work on these tools (and it's not too interested in doing so), I think I did OK. While I do
find all of this stuff cool, at the end of the day I want to get the ball rolling and code on my own
heap of complexity.

That said, if there are some errors, feel free to ping me and I'll update it and thank you for
improving my knowledge of all of this :)
{{</admonition>}}

### 1. Packer

[Packer](https://github.com/wbthomason/packer.nvim) is a package manager for Neovim. Basically it
just clones git repos and runs setup functions over it. It is pretty reliable and can be setup to
run automatically. With this I can just clone my setup, open Neovim once and Packer will make sure all
my plugins are installed and running. Works great.

Main file: https://github.com/cristiandonosoc/environment/blob/main/nvim/lua/user/plugins.lua

### 2. Common dependencies

These are plugins that a lot of other plugins depend on, so I might as well load them of the bat:

- [plenary.nvim](https://github.com/nvim-lua/plenary.nvim): Is a set of common functions. I think of
  it as a pseudo standard library.
- [popup.nvim](https://github.com/nvim-lua/popup.nvim): Provides the popup API so other plugins can
  use it. Eventually this would be upstreamed into main Neovim I guess.
- [nvim-web-devicons](https://github.com/nvim-tree/nvim-web-devicons): Provides nerd-font like icons
  with multiple colors and an lua API.

### 3. nvim-cmp

[nvim-cmp](https://github.com/hrsh7th/nvim-cmp) is a completion engine for Neovim. It perform the
completions per se, but rather sources are hooked and nvim-cmp will use them accordingly. This is a
very central plugin from which a whole tree of other plugins "hook to", most notably being the whole
LSP world.

Main file: https://github.com/cristiandonosoc/environment/blob/main/nvim/lua/user/cmp.lua

### 4. LuaSnip

In a very modular fashion, `nvim-cmp` seems to not paste the changes into the code, but rather
delegates that to a snippet engine. I use `LuaSnip`, mostly because that was what they used in the
guide I followed. I'm not too familiar with this part of the plugin ecosystem to be honest, but it
seems to be working fine...

### 5. Cmp Sources

As states before, `nvim-cmp` doesn't directly know what to auto-complete, but rather uses sources
that will provide options given a particular state of the text being written. Each plugin refers to
a different source, which can be ranked within `nvim-cmp`.

Some of them are simple, such as [cmp-buffer](https://github.com/hrsh7th/cmp-buffer), which brings
suggestions from the opened buffers. [cmp-path](https://github.com/hrsh7th/cmp-path) does something
similar with paths (though doesn't work very well on Windows sadly).

This is where we hook the LSP world as a auto-complete source.

### 6. cmp-nvim-lsp & configs

[cmp-nvim-lsp](https://github.com/hrsh7th/cmp-nvim-lsp) is a cmp source that wraps over the builtin
[Native Neovim LSP client](https://neovim.io/doc/user/lsp.html). This permits us to get completion
suggestions from different language servers.

Now while the native client wrapper acts as a generic client for the different LSP servers, they do
require specific configurations. Like they literally need different flags values. For that there is
a handy plugin called [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig) which holds config
entries for [many](https://github.com/neovim/nvim-lspconfig/blob/master/doc/server_configurations.md)
known servers and we can use it to configure ours.

How this plugin injects the config over to the native client I'm not sure. As in, if I wanted to
write my own config without this plugin, I would need to investigate how to do it. Luckily, most
major LSP servers are included so unless you're writing your own LSP server, you should be covered.

The settings are a bit of a mess, since there is a whole concept of capabilities (what the LSP
server can do. Think things like "can it report errors?", "can it go to definition?", etc.). Also
there is per buffer hook configs, and per language configs, so it is not that easy to follow. But if
you're curious, the entry point is here: https://github.com/cristiandonosoc/environment/blob/main/nvim/lua/user/lsp/init.lua

### 7. Mason

So now that we have our LSP servers configured, we need to actually get them. We could download the
different servers ourselves, but people decided that **another** package manager dedicated to these
kinds of tooling was a good idea, so that's how we have [Mason](https://github.com/williamboman/mason.nvim)

I have to say, it works pretty well. It installs the tools in separate directories that are easy to
track, deals with PATH for you without contaminating your environment and has a nice dashboard.

There are some complementary plugins that make life easier:
- [mason-tool-installer](https://github.com/WhoIsSethDaniel/mason-tool-installer.nvim): This is a
  plugin that lets you define a list of plugins that you want Mason to install manually at startup.
  You can _kinda_ do something like this with the next plugin, but that one is geared only for LSP
  servers. Honestly, I'm a bit disappointed that this is not ingrained in the main Mason plugin.

- [mason-lspconfig](https://github.com/williamboman/mason-lspconfig.nvim): Works as a bridge between
  the tooling installed by Mason and the LSP configs that nvim-cmp-lsp expects. The way I understand
  it is that this plugin translates all the paths and other custom elements that Mason does into the
  other lspconfig.

### Extras

Finally there are a lot of pseudo-standalone plugins that are important-ish in my environment. I
have other plugins but those are more custom to me (like the colorscheme). If you're curious about
those, you can look at my plugin file in the Packer section.

#### Telescope

[telescope](https://github.com/nvim-telescope/telescope.nvim) is a fuzzy finder dashboard. It
permits to filter through a list of files, preview them, open them, etc. It is pretty neat.

In a classic Neovim modular fashion, it has the concept of pickers, sorters and previewers which
people can plugin to extend functionality. Pickers are the most important IMO, which provide a
source of results, like from rip-grep.

Config file: https://github.com/cristiandonosoc/environment/blob/main/nvim/lua/user/telescope.lua

#### formatter.nvim

[formatter.nvim](https://github.com/mhartington/formatter.nvim) is kinda similar to what
`nvim-lspconfig` is, in the sense that holds a bunch of configurations for known formatters.
With this plugin you can hook formatters config to file types and then the tool will be called
correctly. I can even install the tool with Mason and this config will work correctly because it
can find the tool within PATH.

This one is a bit more manual, but since the language I work on don't change that often, I can
afford to update the config on demand.

Config file: https://github.com/cristiandonosoc/environment/blob/main/nvim/lua/user/formatter.lua

#### toggleterm.nvim

[toggleterm.nvim](https://github.com/akinsho/toggleterm.nvim) is a plugin that permits to open a
terminal and then being able to toggle it on/off without loosing the session. I had something
similar with a simple vim autocmd, so this plugin hasn't completely convinced me yet. In the future
I want to play with this a bit more to see if it justifies itself.

Config file: https://github.com/cristiandonosoc/environment/blob/main/nvim/lua/user/toggleterm.lua

#### nvim-treesitter

This one is a world on to itself. [nvim-treesitter](https://github.com/nvim-treesitter/nvim-treesitter)
is the Neovim interface to [tree-sitter](https://tree-sitter.github.io/tree-sitter/), which is a
generic AST library/generator. The idea is that people write their `tree-sitter` parsers for their
languages and then other programs can use the library to reason about the AST in a generic way.

This plugin permits to download, compile and manage known tree-sitter parsers and then plug them in
the program. The main use-case is better syntax highlighting, which is based on the **actual** AST
rather than using the default regex highlighting.

Apparently people can do all kind of stuff with this, but this is the kind of things (like function
renaming, refactor, etc.) that turn your editor insanely slow when programs become complex, so I
keep it simple for now.

The exception being [nvim-treesitter-context](https://github.com/nvim-treesitter/nvim-treesitter-context)
which is a nice little plugin that permits to show the name of the block in the upper bound if it
cannot be seen. So if your editing a 400 lines long function, you will always see the signature on
top. It's nice.

Config file: https://github.com/cristiandonosoc/environment/blob/main/nvim/lua/user/treesitter.lua

#### nvim-tree

[nvim-tree](https://github.com/nvim-tree/nvim-tree.lua) is a replacement for the trusty NERDTree
that I used to use. It is a quite good file explorer that can be configured a lot. I'm quite happy
with it, as it works better than NERDTree, but there are some things that I could do with the other
that I still haven't being able to do, so I need to spend some time with it.

Config file: https://github.com/cristiandonosoc/environment/blob/main/nvim/lua/user/nvimtree.lua

## Conclusion

Neovim is cool and the modular architecture allows it do advance quite fast. The problem is that it
is a **bomb** of complexity. I keep harping about how Neovim feeling is much better than using other
tools, like VSCode or Rider. But you have to invest into it. Vim is already hard to get into, with
the bindings, the modes and the overall philosophy. On top of that, there all this huge amount of
stuff.

I do believe Vim it is worth it, 1000%. This whole Neovim setup I do believe it worth it, though not
as sure. It is hard to justify to someone that knows nothing on Vim to get into it. Thankfully I got
into Vim fairly early in my programming life, in college. That seems like a good time to get into,
when one has way more time and energy to explore than in the middle portion of your career.

Anyway, hope this was helpful, though as I said, this will be for me for when I do my annual Vim
config cleanup and I have to remember what all this web of nonsense means.


