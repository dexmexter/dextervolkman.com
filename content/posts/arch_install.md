---
title: "Arch Install"
date: 2020-07-06T22:57:52-07:00
draft: false
toc: true
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
    from Ubuntu offering it as a installation option, however the reading I've
    done suggests that BTRFS offers more flexibility when it comes to changing
    the size of the data pools.

## Internet for installation environment

The first thing to do once booted into the Arch iso is to connect to the
internet. I've covered [how to make a bootable USB](./bootable_usb) in a
previous post. Also note that I'm basically just documenting all the steps I
needed to install a basic arch system in this post. A much more better way
to learn how to install arch would be to go through the official tutorial[^1].

Use `ip link` to locate the name of the network device and then enable it with:

```bash
ip link set [device] up
```

Then pick a website and try to reach it with `ping`. Once you've confirmed that networking is up the
next task is to partition the drives.

## Update the system clock

```bash
timedatectl set-ntp true
```

## Partition drives[^2]

Look at the available drives with `lsblk`. In my case I have four drives and
they look like this:

```bash
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINT
sda      8:0    0 931.5G  0 disk
├─sda1   8:1    0   511M  0 part /boot
└─sda2   8:2    0   931G  0 part /
sdb      8:16   0 931.5G  0 disk
└─sdb1   8:17   0 931.5G  0 part
sdc      8:32   0 931.5G  0 disk
└─sdc1   8:33   0 931.5G  0 part
sdd      8:48   0 931.5G  0 disk
└─sdd1   8:49   0 931.5G  0 part
```

If you see numbers it means the drives have already have partitions, that's
fine because we'll be wiping them and starting over. Just take note of the
device names like `sda` or `sdb`. If you want more information about the
available block devices use `fdisk -l`.

To partition the drives, open the device in `parted` like so:

```bash
parted /dev/sda
```

The first drive requires an additional partition for the bootloader. The
commands needed to partition this drive are:

```bash
mktable gpt
mkpart "EFI system partition" fat32 1MiB 512MiB
set 1 esp on
mkpart "Storage0" btrfs 512MiB 100%
quit
```

You can use `print` to review the partitions and ensure everything looks the
way that it should before exiting `parted`. For the other three drives, I did
something like this:

```bash
mktable gpt
mkpart "Storage1" btrfs 0% 100%
quit
```

## Filesystem creation

Now that the partitions are ready, the next step is to make the necessary
filesystems on each of the partitions. The boot partition needs a fat32
filesystem and everything else will be getting a single btrfs filesystem that
spans multiple disks in a RAID1 configuration for both the data and the
metadata.

For the `boot` partition:

```bash
mkfs.fat -F 32 /dev/sda1
```

For the `btrfs` partitions (everything else)[^3]:

```bash
mkfs.btrfs -d raid1 -m raid1 /dev/sda2 /dev/sdb1 /dev/sdc1
```

Mount the filesystem, create the boot directory and mount the `EFI` partition:

```bash
mount /dev/sda2 /mnt
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
lsblk
```

## Select mirrors

Edit `/etc/pacman.d/mirrorlist` so that physically closer mirrors are at the
top.

## Install packages

```bash
pacstrap /mnt base linux linux-firmware tmux vim btrfs-progs ranger man-db
man-pages
```

## Configure system

### Fstab

```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

### Chroot

```bash
arch-chroot /mnt
```

### Time zone

```bash
ln -sf /usr/share/zoneinfo/[region]/[city] /etc/localtime
hwclock --systohc
```

### Localization

Edit `/etc/locale.gen`, uncomment `en_US.UTF-8 UTF-8`. Generate with:

```
locale-gen
```

Create `/etc/locale.conf` and set `LANG`:

```
LANG=en_US.UTF-8
```

### Network configuration

Create `/etc/hostname`:

```
[myhostname]
```

Add matching info to `/etc/hosts`:

```
127.0.0.1       localhost
::1             localhost
127.0.1.1       [myhostname].localdomain    [myhostname]
```

Replace `127.0.1.1` with a static IP if appropriate.

### Root password

```
passwd
```

## Boot Loader[^4]

```bash
bootctl install
pacman -S intel-ucode
```

Install the processor microcode[^5]:

```bash
pacman -S intel-ucode
```

Edit `/boot/loader/loader.conf`:

```
default arch.conf
timeout 4
console-mode max
editor  no
```

Create `/boot/loader/entries/arch.conf`:

```
title   Arch Linux
linux   /vmlinuz-linux
initrd  /intel-ucode.img
initrd  /initramfs-linux.img
options root="PARTUUID=[PARTUUID]" rw
```

The `PARTUUID` can be located with:

```bash
lsblk -dno PARTUUID /dev/sda2
```
[^6]

### Reboot

The system should now be ready to restart the system. Type `exit` to exit the
chroot environment, then type `reboot` command. Remove the installation media
and watch your system boot! The next step is to make sure that networking is
working correctly. A static IP, default route, and DNS servers are all needed.

## Time synchronization

```
timedatectl set-ntp true
```

## Set Static IP[^7]

Start/Enable both two systemd services for networking. `networkd` is for
setting the IP address and `resolved` is for DNS settings:

```bash
systemctl enable systemd-networkd
systemctl start systemd-networkd
systemctl enable systemd-resolved
systemctl start systemd-resolved
```

Create the file `/etc/systemd/network/20-wired.network`:

```
[Match]
Name=eno1

[Network]
Address=192.168.1.6/24
Gateway=192.168.1.1
DNS=1.1.1.1
DNS=8.8.8.8
```

You may need to restart `systemd-networkd` with:

```bash
systemctl restart systemd-networkd
```

Use `ip a` and `ip route` to confirm that all the addresses look right. If you
do not have an IP address yet double check that the network device is set to
    UP with `ip link set [device] up`.

## SSH Access

This is something that should have it's own post so I'm just going to brush
over setting this up for now. Just install `openssh` and configure it by
editing `/etc/ssh/sshd_config`. After editing that file, use `systemctl
enable/start sshd` to get the daemon running. We still have not setup the sudo
user so this is something that really should come later but I set it up as
soon as I can so that I am able to finish everything else from my laptop, with
my mechanical keyboard and comfy chair.

I temporarily allowed root access so that I could do this but that is really
not recommended and when I'm finished I will be setting `PermitRootLogin no`
according to best practice.

## Sudo and user account

Next up is getting a user account setup and allowing that user to issue
commands with elevated priveleges.

Install `sudo`:

```bash
pacman -S sudo
```

Create new user, set password, and open sudoers file for editing:

```bash
adduser -m dexmexter
passwd dexmexter
EDITOR=vim visudo
```
With the sudoers file open, add the following lines:

```
dexmexter ALL=(ALL) ALL
Defaults passwd_timeout=0
Defaults timestamp_timeout=10
```

Now you can switch to the new user and make sure it is working with `su
dexmexter`

## XDG-User directories

```bash
sudo pacman -S xdg-user-dirs
xdg-user-dirs update
```

## SSH part 2

Okay now that we have a user account that is not `root` we can get `ssh`
working correctly. As long as the daemon `sshd.service` is running we should
be able to login with the user account. After confirming that this works,
close the ssh connection and copy your ssh key with:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519 dexmexter@dv-storage
```

Don't forget to undo the root access that was granted earlier and restart the
daemon. SSH is now good to go and the a basic server is ready. This

## Conclusion

That it for now, at this point you can throw the server in a closet somewhere
and start installing whatever special software you are planning to use with
it. This particular server, `dv-storage` will be configured as a network work
storage drive that will be used to hold Proxmox VHD's that all cluster nodes
have access to.

[^1]: Most of the material in this post is ripped from the fantastic [arch
linux installation
guide](https://wiki.archlinux.org/index.php/Installation_guide) and other
archwiki articles.

[^2]: I used the examples from [this
page](https://wiki.archlinux.org/index.php/Parted) to help with the
partitioning syntax.

[^3]: The command for creating the Btrfs partitions was
from [this
page](https://wiki.archlinux.org/index.php/Btrfs#Multi-device_file_system)

[^4]: I use systemd-boot because it comes with systemd and I do not have need
for the complexity of grub. The archwiki has [helpful
instructions](https://wiki.archlinux.org/index.php/Systemd-boot) as always

[^5]: <https://wiki.archlinux.org/index.php/Microcode>

[^6]:
<https://wiki.archlinux.org/index.php/Persistent_block_device_naming#by-partuuid>

[^7]: <https://wiki.archlinux.org/index.php/Systemd-networkd>
