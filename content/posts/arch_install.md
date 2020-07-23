---
title: "Arch Install"
date: 2020-07-06T22:57:52-07:00
draft: true
---

## Intro

I recently finished creating a home backup server so that I can start taking
backups of my laptop automatically. It is a project that lays some groundwork
for future systems I would like to implement. The server is working now and
    want to take the time to document the pieces that I needed to get it
    working. This will take the form of several posts, each focusing on a
    different piece of the implementation. This is the first post and will
    focus on the overview of the project up to the installation of the
    operating system.

## Hardware and Planning

The end goal was to have to a server that I can send backups to automatically.
I built it out of used hardware that I got from work so the hard-drives are
quite old, this means that I felt it was really important to make sure I used
some sort of RAID so that I am able to recover from a hard-drive failure when
it happens.

Using Free and Open Source Software (FOSS) was also a primary goal because I
think open source is great and it is free. GNU/Linux, specifically Arch Linux,
is the operating I decided to go with. Linux has several good ways to implement
drive redundancy, mdadm probably being the most known. I chose the BTRFS file
system because I had heard good things about it from a collegue who has used it
for years in his server. I considered ZFS given the recent hype that resulted
    from Ubuntu offering it as a in installation option, however I read that
    BTRFS offers more flexibility when it comes to changing the size of the
    data pools.
