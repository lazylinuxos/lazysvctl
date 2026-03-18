# lazysvctl

A wrapper around runit's `sv` for LazyLinux (Void Linux). Manages both
system services and per-user services, with a readable table view, coloured
status output, and log access through socklog or svlogd.

---

## How runit works (the short version)

Runit supervises services through three directories:

| Path | Purpose |
|---|---|
| `/etc/sv/<name>/` | Service definition — contains a `run` script (and optionally a `log/run` script) |
| `/var/service/<name>` | Symlink → `/etc/sv/<name>`. A service is **enabled** when this symlink exists |
| `/run/sv/<name>/` | Runtime state managed by `runsv` |

The daemon `runsvdir` watches `/var/service/` and spawns a `runsv` supervisor
for each symlink it finds. Enabling a service means creating the symlink;
disabling it means removing it.

For **user services**, there are no symlinks — the service directory lives
directly under `~/service/`. Disabling moves it to `~/service/.disabled/`;
enabling restores it from there.

---

## Installation

```sh
git clone https://github.com/lazylinuxos/lazysvctl.git
cd lazysvctl
sudo install -m 755 lazysvctl /usr/local/bin/lazysvctl
```

### Optional: shell alias

Add to your `~/.bashrc` or `~/.zshrc` for a shorter name:

```sh
alias lsv='lazysvctl'
```

---

## Logging setup

`lazysvctl` looks for logs in two places, in order:

1. **Per-service svlogd** — if the service has its own `./log/run` script,
   logs land in `<service_dir>/log/main/current`.
2. **socklog-void** — a system-wide syslog daemon that writes all service
   output to `/var/log/socklog/everything/current`. `lazysvctl` filters this
   file by service name when reading it.

### Setting up socklog-void (recommended)

```sh
sudo xbps-install socklog-void

sudo ln -s /etc/sv/socklog-unix /var/service/
sudo ln -s /etc/sv/nanoklogd    /var/service/

# Allow your user to read logs without sudo
sudo usermod -aG socklog $USER
# Log out and back in (or: newgrp socklog)
```

After this, `lazysvctl status sshd` will show recent log lines at the bottom,
and `lazysvctl log-tail sshd` will follow the live stream.

---

## Commands

### Service control

```sh
lazysvctl start   SERVICE [...]   # start and keep running (restart on exit)
lazysvctl stop    SERVICE [...]   # stop gracefully
lazysvctl restart SERVICE [...]   # send term+cont, wait for restart
lazysvctl reload  SERVICE [...]   # send HUP (reload config without restart)
lazysvctl once    SERVICE [...]   # start once, do not restart on exit
lazysvctl pause   SERVICE [...]   # send STOP signal (freeze)
lazysvctl cont    SERVICE [...]   # send CONT signal (unfreeze)
lazysvctl kill    SERVICE [...]   # send KILL signal
```

### Status

```sh
lazysvctl status SERVICE [...]
```

Shows state, PID, process name, uptime, path, and log service state.
Appends the last 10 log lines at the bottom if logs are available.

States:
- `✔ running` — supervised and running normally
- `down` — stopped manually (via `sv down`)
- `✘ failed` — supposed to be running but the process exited unexpectedly; runit is trying to restart it
- `n/a` — `runsv` is not supervising this directory

### Listing

```sh
lazysvctl list                # active (enabled) services with state, PID, command, uptime
lazysvctl list --all          # all services: both enabled and disabled
lazysvctl list --disabled     # only disabled services

# For user services, --disabled lists services parked in ~/service/.disabled/
lazysvctl --user list --disabled
```

Example output:

```
  SERVICE         STATE      ENABLED  PID   COMMAND         TIME
  ──────────────  ─────────  ───────  ────  ──────────────  ────
  NetworkManager  ✔ running  yes      829   NetworkManager  2h
  acpid           ---        no       -     -               -
  sshd            ✔ running  yes      833   sshd            2h
  chronyd         down       yes      -     -               5m
```

### Enable / disable

**System services** (default — no flag needed):

```sh
sudo lazysvctl enable  SERVICE   # create /var/service/<name> → /etc/sv/<name>
sudo lazysvctl disable SERVICE   # stop the service and remove the symlink
```

**User services:**

```sh
lazysvctl --user disable SERVICE
# Stops the service and moves ~/service/<name> → ~/service/.disabled/<name>

lazysvctl --user enable SERVICE
# Restores ~/service/.disabled/<name> back to ~/service/<name>
# If not found in .disabled, prints an error and tells you to create the service
```

### Logs

```sh
lazysvctl log-tail SERVICE   # follow live log (Ctrl+C to stop)
lazysvctl log-cat  SERVICE   # dump the entire current log to stdout
```

When using socklog, output is automatically filtered to show only lines
from the requested service.

### Scripting helpers

```sh
lazysvctl is-active  SERVICE   # exits 0 if running, 1 otherwise
lazysvctl is-enabled SERVICE   # exits 0 if symlink/directory exists, 1 otherwise
```

---

## Global options

| Flag | Effect |
|---|---|
| `--user` | Operate on user services (`~/service` or `$SVDIR`) |
| `-q`, `--quiet` | Suppress informational output |
| `-w SEC` | Override the default 7-second sv timeout |

System services are the default when `--user` is not specified.

---

## Per-user services

User-level services are useful for things that should run as your own user
(SSH tunnels, agents, daemons) without root, and that need access to your
session environment.

### 1. Bootstrap the user service infrastructure

```sh
sudo lazysvctl --user init
```

This creates `/etc/sv/runsvdir-<username>/run`, enables it in `/var/service/`,
and creates `~/service/` — all in one step.

### 2. Write a service

```sh
mkdir -p ~/service/my-tunnel/
cat > ~/service/my-tunnel/run << 'EOF'
#!/bin/sh
exec ssh -N -L 5432:localhost:5432 myserver
EOF
chmod +x ~/service/my-tunnel/run
```

### 3. Control it with `--user`

```sh
lazysvctl --user status my-tunnel
lazysvctl --user restart my-tunnel
lazysvctl --user log-tail my-tunnel
```

### 4. Disable and re-enable without deleting

```sh
lazysvctl --user disable my-tunnel
# Service is stopped and moved to ~/service/.disabled/my-tunnel

lazysvctl --user enable my-tunnel
# Service is restored to ~/service/my-tunnel and picked up by runsvdir
```

---

## Environment variables

| Variable | Default | Effect |
|---|---|---|
| `SVDIR` | `~/service` | User service directory for `--user` mode |
| `SVWAIT` | `7` | Default timeout in seconds (overridden by `-w`) |
