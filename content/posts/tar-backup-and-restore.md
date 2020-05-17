---
title: "Tar Backup and Restore"
date: 2020-05-16T15:06:00-07:00
draft: false
toc: true
images:
tags:
  - linux
  - tar
  - howto
  - backup
---

## Preamble

In my quest to learn and become more proficient with computers, I will often
have some kind of ongoing personal project that I will work on in my spare
time. Over the past few weeks the project that has taken up my evenings has
been building a [Linux From
Scratch](http://www.linuxfromscratch.org/index.html) system. The goal is to
build a working linux system by downloading and compiling everything manually
from the source code.

I chose the systemd version of [the
book](http://www.linuxfromscratch.org/lfs/view/stable-systemd/) to walk
through.  I learned a ton about how programs are built and what pieces are
needed for a working system. I ended up with an extremely basic but functioning
linux system. It was a fun and rewarding project but was also a fairly tedious
process to compile every package manually. If I were to ever do it again I will
likely go for the [Automated Linux From
Scratch](http://www.linuxfromscratch.org/alfs/) version.

If you are wanting to look at the system you can download the backup right
[here](https://drive.google.com/file/d/1JXqcG9EnXOpNSHf7I5TTrtikatTHqPnF/view?usp=sharing).

I have no intentions of using this system on a regular basis but I did want to
find a way to back it up and make it available somewhere that I can look back
at later. Figuring out how to actually backup and restore the system was not
nearly as straight forward as I initially expected, mostly because I did not
have much prior experience with backing up and restoring a linux system. This
was a big part of the learning process and I wanted to document that portion
because I am sure it will be useful to reference later.

This is one way to create a full system backup and restore that backup to a new
disk. These instructions are based on [this
page](https://help.ubuntu.com/community/BackupYourSystem/TAR) from the Ubuntu
Community Wiki. I also needed some help getting the system to boot which I
eventually found
[here](https://howtoubuntu.org/how-to-repair-restore-reinstall-grub-2-with-a-ubuntu-live-cd).

## Backup system

The backup process can be done from the system that is being backed up. This
process uses the `tar` program to create on archive of all the system files,
this archive is then compressed so it takes up less space. The following
commands should either be prefaced with `sudo`. If you do not want to type
`sudo` before every command, you can switch to the root user with `sudo su`.

Switch to root directory.

```bash
cd /
```

Run tar to backup the entire system.

```bash
tar -cvpzf backup.tar.gz --exclude=/backup.tar.gz --one-file-system /
```

Let's break that down.

- `tar` A program used for working with archives.
    - `c` - create a new archive
    - `v` - verbose mode
    - `p` - preserve file permissions
    - `z` - compress the archive
    - `f` <filename> - the name of the archive file to being created. It will be
      made in the working directory.
- `--exclude=/example/path` - This is a list of directories that the program
  should not backup. It is important that tar doesn't try to backup the archive
  it is using creating, this would create a recursive mess and is sure to break
  things.
- `--one-file-system` - This is used to prevent tar from trying to backup any
  other filesystems, including virtual filesystems like `/proc`, `/sys`,
  `/mnt`, `/media`, `/run`, and `/dev`. These are temporary filesystems
  specific to a running  system and will be recreated when the system is
  booted. Backing them up would cause issues.
- `/` - Last comes the directory being backed up, in this case we are backing
  up the root directory which will recursively include the entire system.

Once this process has completed, the archive can be copyied or moved to a USB.
Plug in the USB and locate it with `lsblk` or `fdisk -l`. Next mount the drive.
Replace `XY` with the letter and partition number for the USB drive.

```bash
mount /dev/sdXY /mnt
```

Move the archive file to the USB drive.

```bash
mv /backup.tar.gz /mnt/
```

Unmount the USB.

```bash
umount /mnt
```

The USB can now be removed. The system has now been succesfully backed up. Next
we will move onto how to restore the archive to a different hard drive.

## Prepare new drive

This next section should be done from a Live installation environment. For
instructions on how to create a bootable USB, download an ISO and go
through my [previous post](/posts/bootable-usb) (Ubuntu is a good
place to start but any distribution with a Live installer will work).

Once you are have the hard drive you wish to restore to hooked up and are
booted into the recovery environment, the next step is to format and prepare
the new drive. These instructions have only been tested with Legacy Boot and
will not work with a UEFI system. Booting with UEFI is beyond the scope of this
post.

The easiest way to prepare the new drive is using `gparted` or a similar
program. You'll need to format the drive with a new partition table as well as
a new partition that is formated with a filesystem, typically `Ext4` is used.

## Transfer backup to the new drive

Make sure that the new drive is mounted. Then copy all the contents of the
archive to the new drive.

```bash
tar -xcpzf /path/to/backup.tar.gz -C /path/to/mounted/drive --numeric-owner
```

Options explained:

- `x` - Tells tar to extract the archive file.
- `-C <directory>` - Change to a specific directory before extracting. In this
  case it would be the path where the filesystem for the new drive is mounted.
- `--numeric-owner` - Tells tar to restore the numeric owners of the files in
  the archive, rather than any user names from the restoration environment that
  might match accidentally making some files inaccessible because of
  missmatched user id's.

## Restore grub

Lastly we need to restore the bootloader in order to make the system bootable
again. These commands assume that the partiton with the filesystem is
mounted to `/mnt`.

Bind the directories that `grub` will need.

```bash
sudo mount --bind /dev /mnt/dev &&
sudo mount --bind /dev/pts /mnt/dev/pts &&
sudo mount --bind /proc /mnt/proc &&
sudo mount --bind /sys /mnt/sys
```
Jump into the restored system with `chroot`

```bash
sudo chroot /mnt
```

Install `grub` to the device holding the restored system. Then check that it
installed correctly and update.

```bash
grub-install /dev/sdX
grub-install --recheck /dev/sdX
update-grub
```

It is a good idea to check that `/etc/fstab` and `boot/grub/grub.cfg` are
referencing the correct drives and partition numbers.

That's it! You can now `exit`, `unmount` everything and then `reboot` the system.

```bash
exit &&
sudo umount /mnt/sys &&
sudo umount /mnt/proc &&
sudo umount /mnt/dev/pts &&
sudo umount /mnt/dev &&
sudo umount /mnt

sudo reboot
```

Hopefully it boots. If you can get to the `grub` command line but cannot boot
into the file system, `grub.cfg` is probably misconfigured and looking for the
incorrect drive to boot from. You'll need to find other instructions for how to
find and boot with the correct settings from within `grub`.
