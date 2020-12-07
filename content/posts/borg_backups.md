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
myself to take regular backups, I setup a backup server where I could store
`Borg Backup` repos, then I used `systemd-timers` to automatically take backups
everyday. Here are the instructions for how I set everything up for my systems.
The instructions assume that you have a system to store backups but most of the
information could be adjusted to work in other contexts as well, for example if
you wanted to use an external drive for storage. At the end I'll include links
to the articles where I got most of this setup from.

## Prerequisites

Make sure that `/etc/hosts` includes a line for the IP address of your backup
server.

```
192.168.5.3     dv-data.localdomain          dv-data
```

You'll also need to ensure that the client system you are backing up has `ssh`
access to the computer where the backups are being stored. I like [this
explainer](https://www.digitalocean.com/community/tutorials/how-to-set-up-ssh-keys-on-ubuntu-20-04)
from Digital Ocean on how to set up ssh keys.

Install `borgbackups` on both the client and server  and `borgmatic` on the client:

```bash
sudo pacman -S borg borgmatic
```

If you are not using `archlinux` you will need a different command to install
borg. Check the respective
[borgbackups](https://borgbackup.readthedocs.io/en/stable/installation.html)
and [borgmatic](https://torsion.org/borgmatic/) documentation for installation instructions.

Next is initializing the backup repository on `dv-data` from the client system,
`dv-arch`. First make sure that the folder you want to store backups in is
setup, in my case I created `~/borg` on the server. The following command will
ask for an encryption passphrase used to access the repo, select something
secure and store it somewhere safe, losing this password means losing access to
the backups. The second command is for backing up the repo key, store this
somewhere safe as well.

```bash
borg init --encryption=repokey-blake2 dv-data:/home/dexmexter/borg/dv-arch
borg key export dv-data:~/borg/dv-arch ./dv-arch_borg.key
```

## Backup Script

Now that the backup repository is ready, the next step is to create the
`borgmatic` yaml file that contains the configuration for the backups. Backups can always be triggered manually but
using `borgmatic` makes everything a lot easier to manage.

`/etc/borgmatic/config.yaml`

```
location:
    source_directories:
        - /home
        - /etc

    repositories:
        - dexmexter@dv-data:~/borg/dv-arch

    one_file_system: true

    exclude_patterns:
        - '*.pyc'
        - /home/*/.cache
        - /home/.local/share/Trash

    exclude_caches: true

    exclude_if_present:
        - .nobackup

storage:
    encryption_passphrase: "nude pebbles"
    compression: auto,zstd
    ssh_command: ssh -i /home/dexmexter/.ssh/backups_ed25519 -o ServerAliveInterval=30 -o ServerAliveCountMax=3
    relocated_repo_access_is_ok: true

retention:
    keep_daily: 7
    keep_weekly: 4
    keep_monthly: 6

hooks:
    on_error:
        - echo "Error during prune/create/check."
```

For details on how to make changes to this file and what the different options
mean check the [borgmatic
documentation](https://torsion.org/borgmatic/docs/reference/configuration/).

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
Description=borgmatic backup
Wants=network-online.target
After=network-online.target
ConditionACPower=true

[Service]
Type=oneshot

# Lower CPU and I/O priority.
Nice=19
CPUSchedulingPolicy=batch
IOSchedulingClass=best-effort
IOSchedulingPriority=7
IOWeight=100

Restart=no

LogRateLimitIntervalSec=0

ExecStart=systemd-inhibit --who="borgmatic" --why="Prevents interrupting scheduled backup" /usr/bin/borgmatic --syslog-verbosity 1
```

`/etc/systemd/system/autoborg.timer`

```
[Unit]
Description=Run borgmatic backup

[Timer]
OnCalendar=*-*-* 20:00
Persistent=true

[Install]
WantedBy=timers.target
```

This timer is set to run everyday at 8pm. If you need something different,
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

That's it, now you've got proper backups set up!

The next important thing to know is how to restore files when needed. I'm not
going to explain that here, maybe in another post. The `Borg Backups`
documentation has instructions so if you need to restore any documents that
have been backed up, or if I didn't explain something very well, it would be
best to [rtfm](https://en.wikipedia.org/wiki/RTFM) and see if you can figure it
out yourself.

In the future I want to use a proper network monitoring system like `Zabbix` or
`Nagios` or `Icinga` that would monitor the backups and notify me when
something has gone wrong.  In the meantime I've just been periodically checking
the logs.

## Resources

<https://borgbackup.readthedocs.io/>
<https://torsion.org/borgmatic/docs/reference/configuration/>
<https://practical-admin.com/blog/backups-using-borg/>
<https://opensource.com/article/17/10/backing-your-machines-borg>
<https://blog.andrewkeech.com/posts/170718_borg.html>
