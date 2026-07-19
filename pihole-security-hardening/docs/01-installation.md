# 01 — Installation

## Hardware and OS

- **Board:** Raspberry Pi Zero 2 W (quad-core Cortex-A53, 512MB RAM, 2.4 GHz WiFi)
- **Storage:** 32GB microSD card (see [troubleshooting](05-troubleshooting.md) for card selection notes)
- **OS:** Raspberry Pi OS Lite (32-bit, Debian Trixie / 13)

The Zero 2 W's 512MB of RAM is a constraint that shapes several later decisions (blocklist size selection, avoiding heavier services). The 32-bit OS was chosen over 64-bit specifically to leave more RAM available.

## Flashing the SD card

Used the official Raspberry Pi Imager with pre-configured settings in the advanced options (gear icon):

- Hostname: `piHole`
- User account: `pi` with password set
- WiFi SSID + password + country code
- SSH enabled with password authentication (temporary — replaced with keys later)

Pre-configuring these avoids needing a keyboard and monitor during first boot on a headless deployment.

## First boot and SSH access

```bash
ssh pi@piHole.local
# or by IP once known:
ssh pi@192.168.1.5
```

## System update

```bash
sudo apt update && sudo apt upgrade -y
```

### apt mirror correction

The initial `apt update` failed with connection errors to `raspbian.mirror.net.in` — the geo-based mirror redirector was sending traffic to a broken mirror whose DNS resolved to `127.0.0.1`. Symptom:

```
Err:2 http://raspbian.mirror.net.in/raspbian/... 
  Could not connect to raspbian.mirror.net.in:80 (127.0.0.1). - connect (111: Connection refused)
```

Fix: bypass the redirector and pin to the primary archive directly.

```bash
sudo nano /etc/apt/sources.list.d/raspbian.sources
```

Change:
```
URIs: http://raspbian.raspberrypi.com/raspbian/
```
To:
```
URIs: http://archive.raspbian.org/raspbian/
```

Then:
```bash
sudo apt update
```

## Pi-hole install

```bash
curl -sSL https://install.pi-hole.net | bash
```

Wizard selections:

| Setting | Choice | Reasoning |
|---|---|---|
| Static IP | Continue past warning | IP will be static via DHCP reservation (see [network config](04-network-config.md)) |
| Upstream DNS | Cloudflare (1.1.1.1) | Fast, privacy-respecting, well-maintained |
| Blocklists | Accept default StevenBlack | Replaced later — see [blocklist strategy](02-blocklist-strategy.md) |
| Web admin interface | Yes | Query log and dashboard access |
| Web server (lighttpd) | Yes | Required for admin UI |
| Log queries | Yes | Baseline monitoring |
| Privacy mode | 0 (Show everything) | Per-device stats for troubleshooting; home network context |

The admin password is displayed at the end of install. It can be reset any time with:

```bash
pihole -a -p
```

## Verifying blocking

```bash
dig ad.doubleclick.net @127.0.0.1
```

Expected: `0.0.0.0` in the answer section. Note that the bare domain `doubleclick.net` is not typically on default blocklists; only subdomains like `ad.doubleclick.net` are.

The Pi-hole dashboard is accessible at `http://<pi-ip>/admin`.
