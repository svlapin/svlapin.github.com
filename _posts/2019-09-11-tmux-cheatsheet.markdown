---
layout: post
title: "tmux cheat sheet"
date: 2019-09-11 22:10:50 +0200
categories: engineering
tags: [notes, devops]
---

# tmux cheat sheet

Notes taken while following another `tutoriaLinux`'s [video](https://www.youtube.com/watch?v=BHhA_ZKjyxo).

## Windows

`Ctrl+B C` - create a new tmux window. Changes window stats at the bottom panel. Current window is highlighted with `*`

`Ctrl+B ,` - rename current window

`Ctrl+B P` - switch current window to Previous one

`Ctrl+B N` - switch current window to Next one

`Ctrl+B W` - list windows with an option to switch between.

## Pane

Pane is another abstraction in `tmux`.

`Ctrl+B %` - splits current window or pane into 2 panes horizontally

`Ctrl+B :` - shows prompt for named command. `split-window` to split vertically

`Ctrl+B <arrow keys>` - navigate between panes

## Sessions

`tmux new -s <session name>` - create a new session

`Ctrl+B D` - detach from session

`tmux list-sessions` - list sessions on current machine

`tmux attach -t <session name>` - attach to current session


