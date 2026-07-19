# 02 — Blocklist Strategy

Pi-hole ships with the StevenBlack unified hosts list as its default. This project replaces it with a curated set of HaGeZi's lists for both broader coverage and additional security categories.

## Default vs chosen configuration

| List | Domains | Category | Status |
|---|---|---|---|
| StevenBlack unified hosts | ~76,000 | Ads + malware (combined) | **Replaced** |
| HaGeZi Multi Normal | ~177,000 | Ads, trackers, telemetry | Added |
| HaGeZi PopupAds | ~56,000 | Pop-up / pop-under ads | Added |
| HaGeZi Fake | ~17,000 | Fake shops, scam sites | Added |
| HaGeZi TIF (mini) | ~149,000 | Malware, phishing, C2 | Added |
| **Total unique** | **~312,000** | | |

## Why HaGeZi over StevenBlack

**StevenBlack** is the community-standard baseline — well-maintained, safe defaults, low false-positive rate. It's a reasonable choice for a "just make ads go away" setup and is a superset of several older lists (MVPS HOSTS, Peter Lowe's list, etc.).

**HaGeZi** offers three specific advantages for a security-oriented deployment:

1. **Category separation.** HaGeZi splits by purpose — ads/trackers, malware/phishing, fake shops, pop-ups — so each can be added or removed independently based on the environment's risk posture. StevenBlack combines categories into one list.

2. **Broader ad/tracker coverage.** Multi Normal (~177k domains at the "normal" tier) covers significantly more tracking and telemetry than StevenBlack. HaGeZi also offers Light and Pro tiers for different aggression levels.

3. **Dedicated threat intelligence.** The `tif` (Threat Intelligence Feeds) lists focus specifically on known malicious infrastructure — malware distribution, phishing sites, command-and-control servers. This is genuine security value that a general ad-blocking list doesn't provide.

## Why not add every list

Adding more lists is not strictly better. Considerations:

- **False positives.** Aggressive lists break sites or services, causing user complaints and eroding trust in the system.
- **Memory footprint.** The full `tif.txt` (2.2M domains) exceeded the Zero 2 W's practical capacity during a gravity rebuild — see below.
- **Diminishing returns.** Beyond ~300k well-curated domains, additional lists mostly add duplicates and stale entries.

## Sizing for constrained hardware

The initial attempt used the full `tif.txt` (2.2M domains). Gravity build failed with a database corruption error partway through:

```
[✗] Unable to build gravity tree in /etc/pihole/gravity.db_temp
Error in 5th command line argument: database disk image is malformed
```

Root cause: the Pi Zero 2 W's 512MB of RAM couldn't complete the gravity tree build with that domain count.

Fix: switched to `tif.mini.txt` (~149k domains), which retains threat intelligence value at ~15x smaller footprint. Same category coverage, sized to hardware.

## Configuration

Adlists are configured via the admin UI → **Lists**. Each URL is added separately. After adding, gravity is rebuilt:

```bash
sudo pihole -g
```

The URLs used:

```
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/multi.txt
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/popupads.txt
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/fake.txt
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/tif.mini.txt
```

## Verifying the configuration

```bash
# Should return 0.0.0.0 (blocked)
dig ad.doubleclick.net @127.0.0.1
dig google-analytics.com @127.0.0.1
dig flurry.com @127.0.0.1

# Memory check after gravity build
free -h
```

Post-configuration memory profile on the Zero 2 W:

```
              total        used        free      shared  buff/cache   available
Mem:          424Mi       130Mi       154Mi       7.0Mi       196Mi       294Mi
Swap:         423Mi          0B       423Mi
```

294 MiB available with all four lists loaded — comfortable headroom. Swap has not been touched.
