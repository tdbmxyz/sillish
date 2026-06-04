# Viewing the Systemd User Service Logs of a System User

It's not possible to log as a system user directly since system users don't have a login shell. There's however the linger option for `loginctl`, which allows a user manager to run for a system user even when no one is logged in as that user ([See `loginctl` man page](https://www.man7.org/linux/man-pages/man1/loginctl.1.html) and Arch Wiki on [Systemd User](https://wiki.archlinux.org/title/Systemd/User#Automatic_start-up_of_systemd_user_instances)).

Once that's done, you still can't simply use `journalctl --user -u service`/`systemctl --user status -u service` while being logged as another user to see the logs of the system user (since `--user` only works for the current user).
However, you can use `--machine` to specify the system user instance you want to query:

- First I found this [StackExchange answer](https://unix.stackexchange.com/questions/245768/managing-another-users-systemd-units) about managing another user's systemd units. But it references another solution (listed below), more recent (for `systemd>=248`).
- The solution to use `--machine` / `-M` says `sudo systemctl -M testuser@ --user restart foobar.service`. So I tried (with immich user as that's the system user I want to have a service for):

```bash
$ sudo systemctl -M immich@ --user status restic-backups-immich.service
Unit restic-backups-immich.service could not be found.
```

Even though the service file was present in `[IMMICH_HOME]/.config/systemd/user/restic-backups-immich.service`. I didn't understand why it couldn't find it, so I tried the first answer's solution, even if outdated.

What it says is you need to set the XDG_RUNTIME_DIR and DBUS_SESSION_BUS_ADDRESS environment variables to point to the system user's instance. Here are the steps I followed:
1. Switch to the system user (without a shell, so using `su -s /bin/bash -`):
```bash
sudo su immich -s /run/current-system/sw/bin/bash # For NixOS since /bin/bash might not be available
```
2. Set the `XDG_RUNTIME_DIR` and `DBUS_SESSION_BUS_ADDRESS` values:
```bash
export XDG_RUNTIME_DIR="/run/user/$(id -u)"
export DBUS_SESSION_BUS_ADDRESS="unix:path=${XDG_RUNTIME_DIR}/bus"
```