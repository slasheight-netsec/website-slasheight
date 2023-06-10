---
author: "Hugo Authors"
title: "TMUX demystified"
date: 2023-6-10T12:00:06+09:00
description: "Why TMUX is amazing, and how I configure my settings."
draft: false
hideToc: false
enableToc: true
enableTocContent: true
author: Jacob Kollasch
authorEmoji: By
pinned: true
tags: 
- hugo
- github
- netlify
---
```
# JACOB'S TMUX CONFIG FILE - LAST UPDATE - 3/23/23
unbind C-b
set-option -g prefix C-j
bind-key C-j send-prefix
#
# split panes using | and -
bind | split-window -h
bind - split-window -v
unbind '"'
unbind %
#  
bind r source-file ~/.tmux.conf
# 
# mouse mode
set -g mouse on
```