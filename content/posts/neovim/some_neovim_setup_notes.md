---
title: "Some Neovim Setup Notes"
date: 2024-03-05T20:22:12-05:00
draft: true
toc: false
images:
tags:
  - neovim
---

I have been using Vim for over a decade now. Using Vim bindings is second-nature to me now and I
refuse to program on an editor/tool that does not have Vim bindings. I also get grumpy when the Vim
emulation on those tools doesn't work quite right. Rider is the latest thing to annoy me.

But throughout those times, I have used a relative vanilla Vim config. Like everyone else, I had a
.vimrc that I have accumulated over time, building some idiosyncrasies along the way. Some things
about my setup that are relative uncommon:

- I use "," as my leader.
- In normal mode, I mapped <space> to search (basically like pressing '/').
- Since as long as I remember, in normal mode I use `<tab>` to format the file (eg. invoke gofmt).

... and so and so forth.

That is all fine, but in terms of plugins, I have always been fairly austere. Mostly because I
always felt the environment to be very complicated and I didn't felt the need. The one big exception
was [YouCompleteMe](https://github.com/ycm-core/YouCompleteMe), which I used for the longest time to
get C++ (and later Go) completions.

## Neovim Comes In

I have been using Neovim for a while, but really I was using the same as Vim. I was using my same
.vimrc. I really changed because of the integrated terminal, as I am a heavy terminal user.

But this year that changed as I tool the plunge and I went full Neovim by creating a native setup,
to get all the goodies that the native Neovim ecosystem marketed so much. So over the course of
several days, in my spare time I went through pretty much all of this great
[Neovim from Scratch](https://www.youtube.com/playlist?list=PLhoH5vyxr6Qq41NFL4GvhFp-WLd5xzIzZ)
series and got myself a pretty decent setup. For the most part it does more stuff that I had in my
previous setup, though some things are still a bit quirky. For the most part I'm happy... but it is
quite a bit more complex.

## Complexity Galore

Neovim and the whole plugin ecosystem that was created is truly great. There are some quite
interesting things that can be done. The downside is that there is *a lot* of things involved, even
in the _base_ pro setup. Packer, nvim-cmp, nvim-lspconfig, mason, mason-lspconfig, treesitter are
some of the concepts that I've had to wrangle together to get my setup working. It is not simple.

So I'm writing this post partly for myself, so I can refer back to it the next time I need to change
something not trivial about my setup and have to re-learn what all these things are. So this is my
completely subjective, mostly wrong glossary of the important things of my Neovim setup.

### Packer

[Packer](https://github.com/wbthomason/packer.nvim) is a package manager. Basically it just clones
git repos and runs setup functions over it. It is pretty reliable and can be setup to run
automatically. With this I can just clone my setup, open Neovim once and Packer will make sure all
my plugins are installed and running. Works great.

Main file: https://github.com/cristiandonosoc/environment/blob/main/nvim/lua/user/plugins.lua

### nvim-cmp

[nvim-cmp](https://github.com/hrsh7th/nvim-cmp) is a completion engine for Neovim. It doesn't know
about any completions per se, but rather sources are hooked and nvim-cmp will show them accordingly.

Main file: https://github.com/cristiandonosoc/environment/blob/main/nvim/lua/user/cmp.lua


Native Neovim LSP client: https://neovim.io/doc/user/lsp.html


