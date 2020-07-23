---
title: "Mpd not working? Try starting with --user"
date: 2020-07-22T21:44:40-07:00
draft: false
---

This is an issue that has happened more than once now. I'll restart my computer
and `ncmpcpp` will not be able to locate the `mpd` database. If this happens
here is what you should try.

Check if an `mpd` server is running with:

```bash
systemctl status mpd
```

If the service is active, stop it with:

```bash
systemctl stop mpd.socket
systemctl stop mpd
```

Once it is stopped you can start it with the current user:

```bash
systemctl --user start mpd
```

I'm certain there is a cleaner fix for this but it's only an issue when I
restart my computer which does not happen very often. Maybe what I'll try is
disabling the service with:

```bash
systemctl disable mpd
```

Then I'll need to find a way to start it with the `--user` flag on boot.
