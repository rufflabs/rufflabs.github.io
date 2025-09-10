---
layout: post
title:  'Fixing keybindings with GhosTTY and remote Linux hosts'
categories: 
image: 
permalink: 
---

I've recently switched to GhosTTY for my primary terminal on macOS and have ran into the issue of missing xterminfo definitions when SSH'ing to remote Linux systems. The following commands copies over the working xterminfo to the remote system. In particular, I often run into this with Kali Linux.

First install the xterminfo to the Kali user:
```
infocmp -x | ssh kali@10.2.10.99 -- tic -x -
```

Then remote in as Kali user and run the following to install it systemwide:
```
infocmp -x | sudo tic -x -o /usr/share/terminfo -
```

Now the terminal should work as expected!