---
title: "Setting up scheduled Borg Backups with systemd-timers"
date: 2020-09-19T11:21:20-07:00
draft: false
toc: true
images:
tags:
  - howto
  - linux
  - borg backups
  - systemd-timers
---

## Intro

Backups are crucial for making sure that important data does not get lost but
when you have to remember to connect an external drive everytime you need a
backup it becomes a chore and is liable to be forgotten. In order to force
myself to take good backups of my systems, I setup a backup server that I could
store `Borg Backup` repos, then I used `systemd-timers` to automatically take
backups every night. Here are the instructions for how I set everything up for
my systems. The instructions assume that you have a system to store backups but
most of the information could be adjusted to work in other contexts as well. At
the end I'll include links to the articles where I got most of this setup from.

## Prerequisites

Make sure that `/etc/hosts` includes a line for the IP address of your backup
server.

```
192.168.5.5     dv-backups.localdomain          dv-backups
```

You'll also need to ensure that the client system you are backing up has `ssh`
access to the computer where the backups are being stored. I like [this
explainer](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-20-04)
from Digital Ocean on how to set up ssh keys.

Install borg on both the client system and the system storing the backups:

```bash
sudo pacman -S borg
```

If you are not using `archlinux` you will need a different command to install
borg. Check the `Borg Backups` documentation for [installation
instructions](https://borgbackup.readthedocs.io/en/stable/installation.html).

Next is initializing the backup repository on `dv-backups` from the client
system, `dv-storage`. The following command will ask for an encryption
passphrase used to access the repo, select something secure and store it
somewhere safe, losing this password means losing access to the backups. The
second command is for backing up the repo key, store this somewhere safe as
well.

```bash
borg init --encryption=repokey-blake2 dv-backups:/home/dexmexter/borg/dv-storage
borg key export dv-backups:~/borg/dv-storage ./dv-storage_borg.key
```

## Backup Script

Now that the backup repository is ready, the next step is to create a script
that contains all the backup settings. Backups can always be triggered manually
but making a script will allow for automating the entire process as well as
provide consistency for the backup archive names.

`~/.local/bin/autoborg`

```
#!/bin/sh

# ssh key setup
eval $(ssh-agent)
ssh-add /home/dexmexter/.ssh/backups_ed25519

# repo and passphrase
export BORG_REPO="dexmexter@dv-backups:/home/dexmexter/borg/dv-storage"
export BORG_PASSPHRASE='***********'

# some helpers and error handling:
info() { printf "\n%s %s\n\n" "$( date )" "$*" >&2; }
trap 'echo $( date ) Backup interrupted >&2; exit 2' INT TERM

info "Starting backup"

# backup the directories
borg create 			    \
    --verbose 			    \
    --progress			    \
    --filter AME 		    \
    --list 			        \
    --stats 			    \
    --show-rc 			    \
    --compression zlib,5	\
    --exclude-caches 		\
    --exclude '*.iso'		\
  				            \
    ::'{hostname}-{now}' 	\
    /etc 			        \
    /home			        \

    # Route normal process logging to journalctl
    2>&1

backup_exit=$?

info "Pruning repository"

# prune the repo
borg prune 			\
    --list 			\
    --prefix '{hostname}-'	\
    --show-rc 			\
    --keep-daily   4 		\
    --keep-weekly  2 		\
    --keep-monthly 5		\

    # Route normal process logging to journalctl
    2>&1

prune_exit=$?

# use highest exit code as exit code
global_exit=$(( backup_exit > prune_exit ? backup_exit : prune_exit ))
info "Global exit is $global_exit"

if [ ${global_exit} -eq 0 ]; then
    	info "Backup and Prune finished successfully"
elif [ ${global_exit} -eq 1 ]; then
    	info "Backup and/or Prune finished with warnings"
else
	info "Backup and/or Prune finished with errors"
fi

exit ${global_exit}
```

For details on how to make changes to this file and what the different options
mean check the [Borg Backups
documentation](https://borgbackup.readthedocs.io/en/stable/). Note that this
file will need to be turned into a script with `chmod u+x
~/.local/bin/autoborg` and may need to be executed as the root user depending
on what you are backing up.

## Automating with systemd-timers

For automating the backups I chose to use `systemd-timers`. To make a timer
work there are two parts needed, a file with a `.timer` which defines how often
the job is run, and a file with a `.service` which defines what commands are
going to be executed. In my case the `.service` file is executing the
`autoborg` script from above. It is important to note that both the `.timer`
and `.service` files should have the same prefix name and should be stored in
the same location. This is what mine look like:

`/etc/systemd/system/autoborg.service`

```
[Unit]
Description=Borg User Backup

[Service]
Type=simple
Nice=19
IOSchedulingClass=2
IOSchedulingPriority=7
#ExecStartPre=/usr/bin/borg break-lock $BORG_REPO
ExecStart=/home/dexmexter/.local/bin/autoborg
```

`/etc/systemd/system/autoborg.timer`

```
[Unit]
Description=Borg User Backup Timer

[Timer]
WakeSystem=false
OnCalendar=\*-\*-\* 03:00
RandomizedDelaySec=10min

[Install]
WantedBy=timers.target
```

This timer is set to run everyday at 3am. If you need something different,
you'll need to look up instructions. The `archlinux wiki` has some great
examples on [this page](https://wiki.archlinux.org/index.php/Systemd/Timers).

Enable/Start the timer, then verify that it is loaded and active with:

```bash
sudo systemctl enable autoborg.timer
sudo systemctl start autoborg.timer
systemctl status autoborg.timer
```

Verify that it has been started by checking if it appears in the list of
timers:

```bash
systemctl list-timers
```

When you want to see if the backups have been running correctly you can check
the most recent log with:

```bash
systemctl status autoborg
```

Or view all the logs with:

```bash
sudo journalctl -u autoborg
```

## Conclusion

That's it! Now you've got proper backups set up. The next important thing to
know is how to restore files when needed. I'm not going to explain that here,
maybe in another post. The `Borg Backups` documentation has instructions so if
you are not sure of what something means or if I didn't explain something very
well, it would be best to `RTFM` and see if you can figure it out yourself.

In the future I want to use a proper network monitoring system like `Zabbix` or
`Nagios` or `Icinga` that would monitor the backups and notify me when
something has gone wrong.  In the meantime I've just been periodically checking
the logs.

## Resources

<https://borgbackup.readthedocs.io/>
<https://practical-admin.com/blog/backups-using-borg/>
<https://opensource.com/article/17/10/backing-your-machines-borg>
<https://blog.andrewkeech.com/posts/170718_borg.html>
