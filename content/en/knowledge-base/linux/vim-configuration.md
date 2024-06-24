---
author: "Hugo Authors"
title: "VIM Basics and Configuration File"
date: 2024-06-01T12:00:00+00:00
description: "Love it or hate it. Basics of using the legendary text editor, and the configuration I like to use."
draft: false
hideToc: false
enableToc: true
enableTocContent: false
author: Jacob Kollasch
authorEmoji: Written By
pinned: false
tags: 
- vim
- linux
- productivity
---
When I first learned about linux at Hennepin Tech back in linux admin 1 class I learned about `nano` for editing text files from the command line. `nano` is pretty straightforward. You can use your mouse right to click around the file your editing, and all the keyboard shortcuts you need are listed at the bottom of the window. `nano` is great for begginers to the linux os becuase it doesn't get in the way of learning linux, but eventually it's a right of passage to learn vim. For example, the jump hosts we use at my current job don't even have `nano` installed. Just `vim`, so it's a common linux application that you at least should know the basics of to get around a linux system.

`vim` is a different beast than `nano`, and a beast it is. Powerful, but it comes with a sharper learning curve. It's made to sound scary, but really you just need to learn the keyboard shortcuts to navigate a file, start editing a file, and how to exit `vim` while saving or not saving changes to your file. 

After that the sky is the limit. Some Developers, who may also describe themselves as a masochist when questioned on the matter, use VIM as their full time IDE. It supports endless community plugins, and is of course open-source. 

# Basic Commands


## CONFIGURATION I USE
```
# JACOB'S VIM CONFIG FILE - LAST UPDATE - 6/15/23
set mouse=a
syntax enable
nnoremap <BS> X
set backspace=indent,eol,start

```