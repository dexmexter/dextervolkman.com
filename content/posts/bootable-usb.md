---
title: "Create a Bootable USB with dd"
date: 2020-05-04T17:56:49-07:00
draft: false
toc: false
images:
tags:
  - Linux
  - dd
  - howto
---

This is something I have had to lookup on more than one occasion and would like
my own reference so that I don't have to look it up anymore.

First locate the name of the device using:

`$ lsblk`

and/or

`$ sudo fdisk -l`

In my case it is `sdb`. Make sure the device is not mounted.

`$ umount /dev/sdb*`

Next we need to format the drive so that it is empty. This can be done by
initializing a new file system on the disk. If it is a Linux based operating
system you are flashing then just use ext4.

`$ sudo mkfs.ext4 /dev/sdb`

Use dd (data duplicator) to copy the ISO to the newly formatted disk.

`$ sudo dd if=[ input file ] of=[ output file ] status="progress"

The USB should now be ready. Pretty simple, maybe I'll remember how to do it
one day.

