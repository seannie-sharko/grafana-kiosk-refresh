# Raspberry Pi Grafana Dashboard Kiosk (Wayland + systemd)

Run a Grafana dashboard fullscreen on a Raspberry Pi using Chromium, the **labwc Wayland compositor**, a **systemd user service**, and a **systemd user timer** that restarts Chromium every two hours. **Cron is not used.**

This guide is based on a working Raspberry Pi OS Bookworm/labwc deployment. Hostnames, usernames, local IP addresses, Grafana dashboard identifiers, and other potentially identifying information have been replaced with generic examples or variables.

## What the configuration does

- Opens Grafana in Chromium's true **kiosk** mode, with no tabs or address bar.
- Uses native Wayland (`--ozone-platform=wayland`) instead of Chromium's XWayland backend.
- Starts the kiosk after the labwc desktop launches.
- Restarts Chromium automatically if it exits unexpectedly.
- Restarts the browser at **00:00, 02:00, 04:00, ...** (system local time) using a systemd timer.
- Keeps Chromium logs in the user systemd journal.
- Uses a dedicated Chromium profile instead of removing browser preferences or lock files.

## Prerequisites

- Raspberry Pi running Raspberry Pi OS with a graphical **labwc/Wayland** session.
- Chromium installed (`/usr/bin/chromium-browser` on the system used here).
- A user account that automatically logs into the graphical desktop or otherwise starts a labwc session.
- A reachable Grafana instance and a dashboard URL accessible to that account.
- `systemd --user` and the `systemctl` command.

Run all commands **as the desktop account**, not with `sudo`, unless explicitly stated. The examples use `$HOME` so that no actual username is exposed.

**Dashboard authentication:** A kiosk URL does not bypass Grafana authentication. Ensure the dashboard is accessible to the kiosk's Chromium profile through the intended login, organization permissions, or appropriately restricted anonymous access. Do not embed credentials or tokens in the URL.

## 1. Check the graphical environment

An SSH session can report `tty` for `XDG_SESSION_TYPE` even when the Raspberry Pi desktop is running normally. Check actual processes and sockets instead:

```bash
loginctl list-sessions
ps -ef | grep -E 'labwc|Xwayland|chromium' | grep -v grep
ls -l "/run/user/$(id -u)/wayland-0"
command -v chromium-browser
```

Expected: a running `labwc` compositor, a Wayland socket named `wayland-0`, and a Chromium launcher. `Xwayland` may also exist; its presence is not a problem, but Chromium will be explicitly configured to use native Wayland.

**Other systems:** If the socket is named `wayland-1` (or something else), substitute its name consistently in the files below. These instructions target labwc specifically; X11/Openbox setups require different environment settings.

## 2. Create the launch script

Create the file used in the original setup:

```bash
nano "$HOME/relaunch_grafana_wayland.sh"
```

Paste this script, replacing `GRAFANA_HOST`, `DASHBOARD_UID`, and the dashboard slug with your own Grafana address and dashboard details:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Graphical-session variables are not necessarily inherited by systemd --user.
export XDG_RUNTIME_DIR="/run/user/$(id -u)"
export WAYLAND_DISPLAY="wayland-0"
export XDG_SESSION_TYPE="wayland"

# Example only; customize for your installation.
URL='http://GRAFANA_HOST:3000/d/DASHBOARD_UID/example-dashboard?orgId=1&from=now-10m&to=now&timezone=browser&refresh=5s&kiosk'

# exec keeps Chromium as the service's main process; DO NOT append '&'.
exec /usr/bin/chromium-browser \
  --ozone-platform=wayland \
  --user-data-dir="$HOME/.config/grafana-kiosk" \
  --kiosk \
  --no-first-run \
  --no-default-browser-check \
  --noerrdialogs \
  --disable-session-crashed-bubble \
  "$URL"
```

Mark it executable:

```bash
chmod +x "$HOME/relaunch_grafana_wayland.sh"
```

**Why these flags?** `--kiosk` removes Chromium's normal tabs, address bar, and window decorations. The Grafana `&kiosk` query parameter hides Grafana interface elements, but **does not fullscreen the browser** by itself. `--ozone-platform=wayland` avoids the XWayland backend observed in the original deployment. The dedicated profile avoids interference from other Chromium windows.

Do **not** delete `~/.config/chromium/Default/Preferences` or forcibly remove profile lock files on every launch. Do not add `pkill` to this script; systemd handles the browser process lifecycle.

## 3. Create the systemd user service

```bash
mkdir -p "$HOME/.config/systemd/user"
nano "$HOME/.config/systemd/user/grafana-kiosk.service"
```

```ini
[Unit]
Description=Grafana Chromium Wayland Kiosk

[Service]
Type=exec
Environment=XDG_RUNTIME_DIR=%t
Environment=WAYLAND_DISPLAY=wayland-0
Environment=XDG_SESSION_TYPE=wayland
ExecStart=%h/relaunch_grafana_wayland.sh
Restart=always
RestartSec=10
TimeoutStopSec=15
KillMode=control-group

# Intentionally no [Install] section: the graphical session starts this
# service via labwc autostart, not by enabling it at the user-manager boot.
```

`Type=exec` and `exec chromium-browser` allow systemd to track the browser launch. `Restart=always` relaunches Chromium after exits. When the service is stopped or restarted, `KillMode=control-group` ensures its subprocesses are also stopped.

This service must **not** be started before Wayland is available. That is why the labwc autostart step comes next.

## 4. Start the service from labwc autostart

Edit the desktop user's labwc autostart file:

```bash
mkdir -p "$HOME/.config/labwc"
nano "$HOME/.config/labwc/autostart"
```

**Preserve existing lines** and add this line:

```sh
systemctl --user start grafana-kiosk.service &
```

Remove or comment out any *previous direct launch* of `relaunch_grafana_wayland.sh` or Chromium. Only systemd should launch the kiosk, otherwise duplicate windows and profile conflicts can occur.

The `&` here is appropriate: it backgrounds the **autostart command**, not Chromium inside the managed service.

> On initial installation, if you are currently connected over SSH, the Wayland desktop must already be running before you manually start this service. When the user logs into labwc in the future, the autostart entry starts it automatically. Do not enable the kiosk service under `default.target` as a replacement for graphical-session startup.

## 5. Create the two-hour restart service

Create a short-lived helper service that requests a restart of the kiosk service:

```bash
nano "$HOME/.config/systemd/user/grafana-kiosk-restart.service"
```

```ini
[Unit]
Description=Restart Grafana Chromium Kiosk

[Service]
Type=oneshot
ExecStart=/usr/bin/systemctl --user restart grafana-kiosk.service
```

This is distinct from the long-running kiosk service; it exits once the restart request has been processed.

## 6. Create the two-hour systemd timer

```bash
nano "$HOME/.config/systemd/user/grafana-kiosk-restart.timer"
```

```ini
[Unit]
Description=Restart Grafana Kiosk Every Two Hours

[Timer]
OnCalendar=*-*-* 00/2:00:00
Persistent=false
Unit=grafana-kiosk-restart.service

[Install]
WantedBy=timers.target
```

The timer fires on even-numbered hours (midnight, 02:00, 04:00, and so on) according to the Raspberry Pi's local timezone. `Persistent=false` means missed calendar events while the timer is inactive are not replayed later. The next scheduled run can be checked with `systemctl --user list-timers`.

## 7. Load and start systemd units

Run as the desktop user:

```bash
systemctl --user daemon-reload
systemctl --user enable --now grafana-kiosk-restart.timer
systemctl --user start grafana-kiosk.service
```

The last command assumes labwc and its `wayland-0` socket are already running. It starts the browser immediately for testing; on subsequent desktop logins, labwc autostart starts it.

If you previously ran Chromium manually, **close that Chromium instance first**, especially if it was using the same kiosk profile.

**Do not use `sudo systemctl`** for these user units. No system-wide service or root privileges are required.

## 8. Validate the setup

Check the kiosk service:

```bash
systemctl --user status grafana-kiosk.service
```

Check the timer and its next run:

```bash
systemctl --user status grafana-kiosk-restart.timer
systemctl --user list-timers --all | grep grafana
```

Check logs:

```bash
journalctl --user -u grafana-kiosk.service -n 100 --no-pager
journalctl --user -u grafana-kiosk-restart.service -n 50 --no-pager
journalctl --user -u grafana-kiosk.service -f
```

Confirm Chromium is using native Wayland:

```bash
pgrep -af 'chromium.*--type=gpu-process'
```

Look for `--ozone-platform=wayland`, not `--ozone-platform=x11`. Chromium's main process should also show `--kiosk`.

Test a manual restart **without waiting two hours**:

```bash
systemctl --user restart grafana-kiosk.service
```

Or exercise the one-shot timer target:

```bash
systemctl --user start grafana-kiosk-restart.service
```

Verify that the old browser closes, a new window opens, and Grafana occupies the full screen without browser tabs or an address bar.

## 9. Everyday operation

```bash
# Start, stop, and restart the browser
systemctl --user start grafana-kiosk.service
systemctl --user stop grafana-kiosk.service
systemctl --user restart grafana-kiosk.service

# Start or stop scheduled two-hour restarts
systemctl --user start grafana-kiosk-restart.timer
systemctl --user stop grafana-kiosk-restart.timer

# Disable or re-enable the timer for future user-manager startups
systemctl --user disable --now grafana-kiosk-restart.timer
systemctl --user enable --now grafana-kiosk-restart.timer

# Reload after editing any unit files
systemctl --user daemon-reload
```

There is no need to `enable` the kiosk *service* because labwc autostart is responsible for starting it during a graphical session. The *timer* is enabled so it can run automatically under the user's systemd manager.

## Troubleshooting

### Browser shows tabs/address bar

- Confirm the launch script contains `--kiosk`, not just `--start-fullscreen`.
- Confirm the browser is being started by the systemd unit: `systemctl --user status grafana-kiosk.service`.
- Check for another Chromium instance or another autostart entry: `pgrep -af chromium`.
- Confirm the standalone kiosk profile `~/.config/grafana-kiosk` is not simultaneously used elsewhere.
- Check the main Chromium process arguments and journal for errors.

### `Failed to connect to Wayland display` / browser does not launch

- Verify `ls -l /run/user/$(id -u)/wayland-0` succeeds.
- Verify labwc is running: `pgrep -af labwc`.
- Match the `WAYLAND_DISPLAY` value in **both** launch script and unit to the actual socket name.
- Run `systemctl --user start grafana-kiosk.service` **after** the graphical session is available.
- Check `journalctl --user -u grafana-kiosk.service -b --no-pager`.
- If the service has hit the start-rate limit after repeated failures, run `systemctl --user reset-failed grafana-kiosk.service` after fixing the problem.

### SSH reports `XDG_SESSION_TYPE=tty`

This describes the SSH shell, not necessarily the desktop session. Inspect `labwc`, the Wayland socket, and the browser process rather than assuming that the display server is X11 or absent.

### Grafana dashboard does not load

- Test network connectivity from the Pi to `GRAFANA_HOST:3000`.
- Confirm the hostname/IP, port, dashboard UID, path, and Grafana organization ID.
- Check Grafana login/session access from the dedicated Chromium profile.
- The dashboard query's `refresh=5s` controls Grafana data refreshes; it is different from the systemd timer, which restarts the entire Chromium browser every two hours.

### The timer is active but the kiosk never restarts

```bash
systemctl --user list-timers --all
systemctl --user status grafana-kiosk-restart.service
journalctl --user -u grafana-kiosk-restart.service -n 100 --no-pager
```

Manually exercise the helper with `systemctl --user start grafana-kiosk-restart.service`. Check that the timer is enabled and that the user systemd manager is running. The user manager's lifetime is independent of the desktop's graphical session; if the desktop is closed, the timer may fire while Wayland is unavailable. For a dedicated Pi configured to remain logged into labwc, the recommended setup is usually sufficient.

### Chromium seems to open twice

Check the labwc autostart file, user service state, and any residual cron job. There should be exactly **one** mechanism that starts Chromium: `grafana-kiosk.service`. Labwc starts that service; the timer restarts it.

### A browser process becomes unresponsive

`Restart=always` only restarts after a process exits; it **cannot detect a hung but still-running browser**. The scheduled two-hour restart helps recover from some long-running glitches. Additional health monitoring would be necessary to detect a frozen dashboard before the next timer event.

## Architecture

```text
Desktop login
    |
    v
labwc / Wayland (wayland-0)
    |
    v
~/.config/labwc/autostart
    |
    v
systemctl --user start grafana-kiosk.service
    |
    v
~/relaunch_grafana_wayland.sh
    |
    v
Chromium --ozone-platform=wayland --kiosk
    |
    v
Grafana dashboard

Separately:

grafana-kiosk-restart.timer (every even hour)
    |
    v
grafana-kiosk-restart.service
    |
    v
systemctl --user restart grafana-kiosk.service
```

## Files involved

| Path | Purpose |
| --- | --- |
| `~/relaunch_grafana_wayland.sh` | Launches Chromium in native Wayland kiosk mode |
| `~/.config/systemd/user/grafana-kiosk.service` | Tracks Chromium and restarts it after exits |
| `~/.config/systemd/user/grafana-kiosk-restart.service` | One-shot helper that restarts the kiosk |
| `~/.config/systemd/user/grafana-kiosk-restart.timer` | Triggers the helper every two hours |
| `~/.config/labwc/autostart` | Starts kiosk service when labwc launches |
| `~/.config/grafana-kiosk/` | Dedicated Chromium profile |

## Security and operational notes

- Keep Grafana limited to trusted networks or an authenticated reverse proxy; kiosk mode is not a security boundary.
- Avoid embedding passwords, bearer tokens, private hostnames, personal IP addresses, or real dashboard IDs in shared documentation.
- The examples intentionally use `GRAFANA_HOST` and `DASHBOARD_UID` placeholders. Customize locally.
- The two-hour restart briefly interrupts the displayed dashboard. Choose another schedule if an interruption at exactly even hours is undesirable.
- Systemd user units require an active user manager. Desktop autologin is one practical approach for an unattended display; a headless boot alone does not provide a usable Wayland compositor.
