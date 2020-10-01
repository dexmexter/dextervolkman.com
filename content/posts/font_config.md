---
title: "Change Default Terminal Font"
date: 2020-09-30T22:15:10-07:00
draft: false
toc: false
images:
tags:
  - linux
  - archlinux
  - fonts
---

I was having trouble changing the font in my terminal so I did some searching
and found a solution. The initial problem that I was trying to solve is that
the default terminal font I was using did not have the glyphs I needed for a
prettier version of `ls` called `ls Deluxe` (`lsd` for short).

I'm using `XResources` to manage terminal colours so at first I thought I
should look there. The only thing related to the font was a line about the
default font size though. Eventually I discovered the file I was looking for.

The font that I decided to switch to is
[Hack](https://sourcefoundry.org/hack/).  I installed it with:

```bash
sudo pacman -S ttf-hack
```

Next I needed to create and edit the font config file.

`~/.config/fontconfig/fonts.conf`:

```
<alias>
        <family>monospace</family>
        <prefer><family>Hack</family></prefer>
</alias>
```

After saving the file, I reloaded the terminal and all was right with the world.  Double-check that the correct font is set with:

```bash
fc-match monospace
```

Here is the beautiful end result:
![Working lsd](/img/working_lsd.png)
