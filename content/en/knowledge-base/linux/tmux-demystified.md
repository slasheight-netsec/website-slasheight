---
author: "Hugo Authors"
title: "TMUX Basics and Configuration"
date: 2024-06-21T12:00:00+00:00
description: "TMUX commands that I find useful, and my configuration file. Let's get multiplexing."
draft: false
hideToc: false
enableToc: true
enableTocContent: false
author: Jacob Kollasch
authorEmoji: Written By
pinned: false
tags: 
- tmux
- linux
- productivity
---
I wrote this page so everytime I forget something about TMUX I can look here. Including my current tmux config explained. TMUX So buckle up dawg because I heard you like terminals so we put some terminals inside your terminal so you have terminals inside your terminal.

[put crazy tmux terminal screenshot here, maybe in xzibit meme...]

## What is so great about TMUX?
TMUX stands for `terminal multiplexer`. Wow that sounds sweet what else do you need to know? "Multiplexer" is a term borrowed from electronics where it's defined as "a device that selects between several input signals and forwards the selected input to a single output line". Well TMUX allows us to select between multiple "pseudo terminals" using a single "real" terminal.

Imagine you are `ssh`'ing to a remote server to complete some administrative tasks, and you want to run multiple programs at the same time. Well you could create multiple seperate ssh connections to the remote server, or you can just split your existing ssh session using tmux.

One of the most truly brilliant features of TMUX is it seperates your programs from the actual terminal which will stop them from disconnecting. You can detach TMUX from the terminal and they will still run in the background. You could have an entire workflow using multiple apps assigned to a tmux session, and whenever you need to start that work you just fire up your tmux session and start right where you left off again. 

## TMUX Install
Let's go to the tmux github to make sure we install the latest version of tmux from source code instead of whatever version is hanging out in apt. It's much cooler.

This is somehting I like to check before installing any new tool. Unless I have a good reason to use an older version I only want the most bleeding edge version of my 16 year old open-source software. At the time of writing apt has version `3.2a`, but tmux offical repo has `3.3a` with ~30 documented changes from `3.2a` that I will probably never use. However, without reading them let's just assume *some* changes are securtiy and stability related, which is always a win!

https://github.com/tmux/tmux/releases/

### Install Script Breakdown

{{< codes python test>}}
  {{< code >}}
  ```bash
    tar -zxf tmux-*.tar.gz
    cd tmux-*/
    ./configure
    make && sudo make install
  ```
  {{< /code >}}
  {{< code >}}
  ```javascript
  console.log('Hello World!');
  ```
  {{< /code >}}
{{< /codes >}}

```
# Run this script from the same dir you installed the tmux download if that's not obvious. You can't extract what's not there.
tar -zxf tmux-*.tar.gz
cd tmux-*/
./configure
make && sudo make install
```
Let's break this install script down real quick. The first line is using the `tar (archiving utility)` to extract our tmux installer download. Wait my download file is called `tmux-3.3a.tar.gz` why is tar extracting `tmux-*.tar.gz`? Well that `*` is a wildcard meaning anything can match here. Anything including `3.3a` our tumx version. What is this a bash tutorial now?

Next line is `cd`'ing or `changing directory` using the `cd` command to navigate to our newly extracted installer. Check it out another usage of the `*` wildcard character.

Third line is using `./configure` command. What is the meaning of this magical conmmand that I am alwyas running to install my rad linux tools? You may be asking yourself that question right about now becuase you like to turn over rocks and see what's under there sometimes! Well under this rock we find that `./configure` is an execuable script that will help us make sure our system has the neccesary dependencies for our program to run. It will also check our system has a C complier we can use because TMUX is written in the ye old language of `C`.

The `.` portion is saying in this directory right here, and combined with the `/configure` portion is essentially telling our computer to run this script located in this directory.

![Jacob Kollasch](/tmux-demystified-images/configure-file-1.jpg)

The script has the executable bit set which we can see from the output of our `ls` command above.

One last thing, if you look at the top of the ./configure file you will see a shebang, or more specfically a bash shebang. A shebang `#!` tells our kernel which interpreter should be used to run the commands in the file. A bash shebang `#!/usr/bin/env` syas use the bash interpreter.

If we didn't include a shebang everytime we wanted to run this script we would have to run `/bin/bash ./configure`. I guess this just a bash tutorial.

## Basics of TMUX

## CONFIGURATION I USE
```
# JACOB'S TMUX CONFIG FILE - LAST UPDATE - 3/23/23
# This line changes unbinds the default tmux config from Ctrl+b
unbind C-b
# These 2 line will change our tmux shortcut to Ctrl+j
set-option -g prefix C-j
bind-key C-j send-prefix
#
#  These lines will change split pane shourtcuts to use | and -
bind | split-window -h
bind - split-window -v
unbind '"'
unbind %
#
# This command lets you update the tmux config from inside tmux without restarting your tmux session, and is really useful when playing with tmux configs
bind r source-file ~/.tmux.conf
# 
# TMUX is designed to be all keyboard input in the same vain as VIM, but it's too hard to not cheat sometimes.
# Enables mouse mode
set -g mouse on
```