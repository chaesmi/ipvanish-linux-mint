# IPVanish Linux Mint Watchdog

Scripts and systemd units for running IPVanish OpenVPN on Linux Mint with:

- boot-time VPN startup
- automatic restart checks
- desktop alerts when restart fails
- optional UFW-based kill switch
- narrow passwordless sudo helpers

This repo does not include IPVanish credentials or `.ovpn` server files.

## Expected Local Paths

The scripts default to:

```text
${HOME}/.config/ipvanish/configs
${HOME}/.config/ipvanish/auth.txt
```

Create the auth file with:

```bash
bin/ipvanish-auth-setup
```

## Install

Copy scripts into place:

```bash
install -m 0755 bin/ipvanish-* ~/.local/bin/
```

Install the systemd units, root helpers, and sudoers rule:

```bash
~/.local/bin/ipvanish-install-service
```

That command installs:

```text
/etc/systemd/system/ipvanish.service
/etc/systemd/system/ipvanish-watchdog.service
/etc/systemd/system/ipvanish-watchdog.timer
/usr/local/sbin/ipvanish-service-control
/usr/local/sbin/ipvanish-set-location-root
/usr/local/sbin/ipvanish-killswitch-root
/etc/sudoers.d/ipvanish
```

The installer renders the systemd and sudoers templates using the current
`${HOME}` and `${USER}` values, so the installed units point back to the
installing user's home directory.

You can override the default config/auth locations with:

```bash
export IPVANISH_CONFIG_DIR="${HOME}/.config/ipvanish/configs"
export IPVANISH_AUTH_FILE="${HOME}/.config/ipvanish/auth.txt"
```

## Common Commands

Identify IPVanish systemd entries:

```bash
systemctl list-unit-files '*ipvanish*'
systemctl list-units '*ipvanish*'
systemctl status ipvanish.service ipvanish-watchdog.timer ipvanish-watchdog.service
```

Start, stop, and restart:

```bash
sudo ipvanish-service-control start
sudo ipvanish-service-control stop
sudo ipvanish-service-control restart
sudo ipvanish-service-control status
```

Direct systemd commands:

```bash
sudo systemctl start ipvanish.service
sudo systemctl stop ipvanish.service
sudo systemctl restart ipvanish.service
sudo systemctl status ipvanish.service
```

Watchdog timer:

```bash
sudo systemctl start ipvanish-watchdog.timer
sudo systemctl stop ipvanish-watchdog.timer
sudo systemctl restart ipvanish-watchdog.timer
sudo systemctl status ipvanish-watchdog.timer
```

Enable or disable at boot:

```bash
sudo systemctl enable ipvanish.service
sudo systemctl disable ipvanish.service
sudo systemctl enable ipvanish-watchdog.timer
sudo systemctl disable ipvanish-watchdog.timer
```

Run the health check immediately:

```bash
sudo systemctl start ipvanish-watchdog.service
```

View logs:

```bash
journalctl -u ipvanish.service -n 100 --no-pager
journalctl -u ipvanish-watchdog.service -n 100 --no-pager
journalctl -u ipvanish.service -f
```

## Location Selection

Use all downloaded configs randomly:

```bash
ipvanish-set-location random
```

Use a matching location randomly:

```bash
ipvanish-set-location Reykjavik
ipvanish-set-location Chicago
```

## Status

```bash
ipvanish-status
ipvanish-service-status
vpn-watch
```

`ipvanish-status` shows the selected IPVanish location, `.ovpn` config path,
remote server, tunnel interface, OpenVPN process, NetworkManager VPN state, and
public network.

## Desktop Alerts

When the watchdog detects that the VPN is unhealthy, it attempts to restart
`ipvanish.service`. If restart fails, or if no tunnel is active after restart,
`ipvanish-healthcheck` calls `ipvanish-desktop-alert`.

The alert helper uses:

- `notify-send` for a desktop notification
- `zenity` for a visible warning popup window

The popup is intentionally implemented with `zenity` instead of Tk because
Linux Mint commonly has `zenity` available and it integrates with the desktop
session without needing a custom Python/Tk script.

Test the popup manually:

```bash
ipvanish-desktop-alert critical "IPVanish test" "This is a VPN warning test."
```

## Kill Switch

Enable:

```bash
ipvanish-killswitch-on
```

Disable:

```bash
ipvanish-killswitch-off
```

The kill switch uses UFW and resets UFW rules. It allows outbound traffic through `tun0`, OpenVPN UDP `1194`, DNS, and local LAN ranges. Do not enable it during remote-only access unless you have confirmed the rules fit that environment.
