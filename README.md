# IPVanish Linux Mint

This repository is the source of truth for the local IPVanish OpenVPN helper
scripts on this machine.

It provides:

- boot-time VPN startup with systemd
- automatic health checks and restart attempts
- desktop alerts when the watchdog cannot recover the tunnel
- location selection for the next service start
- UFW-based kill switch managed automatically with the VPN
- narrow root helpers for systemd, UFW, and service override changes

Credentials and IPVanish `.ovpn` server files are not stored in this repo.

## Everyday Commands

Use `ipvanish` as the single entry point. Run it without sudo; it requests
privileges internally when needed. Installation is only required for initial
setup or after updating scripts, helpers, or service definitions.

| Command | Action |
| --- | --- |
| `ipvanish stop` | Stop VPN/watchdog and release kill switch |
| `ipvanish start` | Enable kill switch and connect |
| `ipvanish restart` | Enable kill switch and reconnect |
| `ipvanish disable` | Stop, release kill switch, disable boot startup |
| `ipvanish enable` | Enable protection, connect, enable boot startup |
| `ipvanish status` | Show VPN and network status |
| `ipvanish diagnose` | Show detailed network diagnostics |
| `ipvanish ip` | Show public IP, location, and provider |
| `ipvanish location Stockholm` | Choose location for the next connection |
| `ipvanish list Chicago` | List matching locations |
| `ipvanish install` | Install/refresh after updates |
| `ipvanish help` | Show available commands |

Location changes take effect on the next start/restart. A successful start
request does not mean the tunnel has finished connecting; check `ipvanish status`.
Existing `ipvanish-*` commands and `ipstatus` remain available for compatibility.

## Layout

Source files live here:

```text
~/Documents/IPVanish/
  bin/ipvanish   main user command
  bin/           supporting scripts and compatibility commands
  root-helpers/  root-owned helper commands installed to /usr/local/sbin
  systemd/       systemd unit templates
  sudoers.d/     sudoers template
```

Installed runtime locations:

```text
~/.local/bin/ipvanish            main command (symlink to source)
~/.local/bin/ipvanish-*          symlinks to ~/Documents/IPVanish/bin/
/usr/local/sbin/ipstatus         compact status shortcut
/usr/local/sbin/ipvanish-*       root-owned installed helpers
/etc/systemd/system/ipvanish*    rendered systemd units
/etc/sudoers.d/ipvanish          rendered sudoers rule
```

The root helpers intentionally remain installed as root-owned files in
`/usr/local/sbin`. Do not replace them with symlinks to a user-writable
directory while they are allowed by sudoers.

## Requirements

Expected packages:

```bash
sudo apt install openvpn systemd ufw curl libnotify-bin zenity
```

Expected IPVanish files:

```text
~/.config/ipvanish/auth.txt
~/.config/ipvanish/configs/*.ovpn
```

Create or replace the auth file:

```bash
ipvanish-auth-setup
```

The auth file contains your IPVanish username on the first line and password on
the second line. It is written with mode `600`.

## Install Or Refresh

For first installation, run the installer directly from the source tree:

```bash
/home/ces/Documents/IPVanish/bin/ipvanish-install-service
```

After initial installation, refresh systemd units, root helpers, sudoers,
and user command symlinks using:

```bash
ipvanish install
```

The installer resolves service paths from the invoking user's account, including
when run through sudo with `HOME=/root`. To repair older units containing
`/root/.local/bin`, rerun the installer and restart the VPN:

```bash
sudo /home/ces/.local/bin/ipvanish-install-service
sudo ipvanish-service-control restart
```

Installed files:

```text
/etc/systemd/system/ipvanish.service
/etc/systemd/system/ipvanish-watchdog.service
/etc/systemd/system/ipvanish-watchdog.timer
/usr/local/sbin/ipstatus
/usr/local/sbin/ipvanish-service-control
/usr/local/sbin/ipvanish-set-location-root
/usr/local/sbin/ipvanish-killswitch-root
/etc/sudoers.d/ipvanish
```

## Common Usage

Show VPN status:

```bash
ipvanish-status
ipvanish-service-status
ipstatus
ipstatus --diagnose
```

Status output is intentionally compact:

```text
IPVanish: up
Settings: location="IL Tel Aviv" remote=tlv-c11.ipvanish.com:1194 config=ipvanish-IL-Tel-Aviv-tlv-c11.ovpn
Tunnel: tun0
OpenVPN: running pid=1201441
Public network: AS62651 Strong Technology, LLC.
ipvanish.service: active/enabled
ipvanish-watchdog.timer: active/enabled
ipvanish-watchdog.service: inactive/static last=success
```

`ipstatus` also shows NetworkManager, Wi-Fi radio/device/connection/signal,
driver, and default-route state. Use `ipstatus --diagnose` when networking is
stuck (or immediately after reboot) to include addresses, routes, DNS,
gateway reachability, and recent NetworkManager/kernel events. Individual
checks have short timeouts so a wedged network service does not hang the status
command.

`Public network: unknown` means the external organization lookup did not
complete. The following `Network assessment` line combines that result with
Wi-Fi, NetworkManager, and default-route state to distinguish a likely local
Wi-Fi failure from an isolated external lookup failure.

List available locations:

```bash
ipvanish-list
ipvanish-list Chicago
```

Connect manually:

```bash
ipvanish-connect random
ipvanish-connect Chicago
ipvanish-connect ipvanish-US-Chicago-chi-b42
```

Set the location used by `ipvanish.service` on the next start:

```bash
ipvanish-set-location random
ipvanish-set-location Reykjavik
ipvanish-set-location Chicago
```

Temporarily stop the VPN and use normal internet access, then turn it back on:

```bash
sudo ipvanish-service-control stop
# When ready to reconnect:
sudo ipvanish-service-control start
ipstatus
```

`stop` stops the watchdog timer, any running watchdog check, and the VPN. If
UFW is active with a default deny/reject outgoing policy, it saves that policy
and temporarily changes it to allow outgoing traffic. Existing firewall rules
and the incoming policy remain intact; explicit blocking rules still apply.
There is no need to run `ipvanish-killswitch-off` separately.

`start`, `restart`, and `enable` turn on the kill switch **before** starting
the VPN and watchdog timer, even if UFW was inactive or you used the standalone
kill-switch-off command. They ensure the VPN, DNS, and LAN firewall allowances
exist without resetting existing rules, then restrict default outgoing traffic
(deny, or the reject policy saved during a pause). Existing explicit allow rules
remain effective. Wait for `ipstatus` to show the VPN is up before using it.
If connection fails, the kill switch remains on. Use `stop` to regain normal
internet access. Repeated stops preserve the original saved policy.

`pause` and `resume` are aliases for `stop` and `start`. `stop` does not change
the boot startup setting. The saved policy lives in the root-only directory
`/var/lib/ipvanish/`; boot startup also restores it before connecting, so a
reboot ends the pause. The existing kill-switch on/off commands reset UFW rules;
the combined stop/start commands do not.

After updating the source scripts, install the root helper and updated systemd
unit using `ipvanish-install-service`. This refreshes the installed files and
enables boot startup without stopping or starting the current VPN connection.

Restart or inspect the managed service:

```bash
sudo ipvanish-service-control start
sudo ipvanish-service-control stop
sudo ipvanish-service-control restart
sudo ipvanish-service-control status
```

Start the VPN and enable automatic startup, or stop it and disable startup:

```bash
sudo ipvanish-service-control enable
sudo ipvanish-service-control disable
```

`disable` performs the same VPN/watchdog stop and firewall release as `stop`,
then disables startup at boot. `enable` turns on the kill switch, enables
startup at boot, and connects immediately. No separate kill-switch commands
are needed for these managed stop/start operations.

Run the watchdog health check immediately:

```bash
sudo systemctl start ipvanish-watchdog.service
```

View logs:

```bash
journalctl -u ipvanish.service -n 100 --no-pager
journalctl -u ipvanish-watchdog.service -n 100 --no-pager
journalctl -u ipvanish.service -f
```

## Kill Switch

Enable the UFW kill switch:

```bash
ipvanish-killswitch-on
```

Disable the UFW kill switch:

```bash
ipvanish-killswitch-off
```

The kill switch resets UFW rules. It denies default outgoing traffic, allows
traffic through `tun0`, allows OpenVPN UDP `1194`, allows DNS, and allows common
RFC1918 LAN ranges. Review this before enabling it on a remote-only machine.

## Configuration

Useful environment overrides:

```bash
export IPVANISH_CONFIG_DIR="$HOME/.config/ipvanish/configs"
export IPVANISH_AUTH_FILE="$HOME/.config/ipvanish/auth.txt"
export IPVANISH_MATCH="random"
export VPN_EXPECTED_ORG="IPVanish"
export IPVANISH_ALERT_TIMEOUT="12"
export IPVANISH_ALERT_WIDTH="360"
export IPVANISH_ALERT_HEIGHT="110"
```

`IPVANISH_MATCH` is used by the systemd service. `VPN_EXPECTED_ORG` is optional
and lets the watchdog mark the VPN unhealthy if `https://ipinfo.io/org` does not
match the expected provider string. `IPVANISH_ALERT_TIMEOUT`,
`IPVANISH_ALERT_WIDTH`, and `IPVANISH_ALERT_HEIGHT` tune the warning popup.

## Troubleshooting

Check installed unit files:

```bash
systemctl list-unit-files '*ipvanish*'
systemctl list-units '*ipvanish*'
systemctl status ipvanish.service ipvanish-watchdog.timer ipvanish-watchdog.service
```

Check OpenVPN and tunnel state:

```bash
ipvanish-status
pgrep -a openvpn
ip -brief link show
```

Test desktop alerts:

```bash
ipvanish-desktop-alert critical "IPVanish test" "This is a VPN warning test."
```
