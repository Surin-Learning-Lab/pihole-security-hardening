# 05 — Troubleshooting

Real operational issues encountered during this build and how they were diagnosed.

## Counterfeit SD card

### Symptom

After Pi-hole install and configuration, filesystem errors began appearing in `dmesg`:

```
EXT4-fs error (device mmcblk0p2): ext4_validate_block_bitmap:423: comm ext4lazyinit: bg 112: bad block bitmap checksum
EXT4-fs error (device mmcblk0p2): ext4_validate_block_bitmap:423: comm ext4lazyinit: bg 245: bad block bitmap checksum
EXT4-fs error (device mmcblk0p2): ext4_validate_block_bitmap:423: comm apt: bg 113: bad block bitmap checksum
EXT4-fs error (device mmcblk0p2) in ext4_mb_clear_bb:6683: Filesystem failed CRC
```

`apt` operations began failing (`Unable to parse package file`). Individual binaries were corrupted — `fail2ban-client` returned shell syntax errors because its file contents were binary garbage instead of a Python script.

### Diagnosis

Initial suspicion: hard power-cycling during writes. Ruled out — the Pi had been shut down cleanly. Undervoltage next: `vcgencmd get_throttled` returned `throttled=0x0`, indicating no power issues.

Tested the card with **H2testw** (Windows tool for detecting counterfeit and defective flash storage):

1. Format the card as a single partition using Disk Management (Windows Explorer's format won't clear leftover Pi partition tables).
2. Run H2testw against the full capacity.

Result:

```
Warning: Only 31981 of 31982 MByte tested.
The media is likely to be defective.
7.8 GByte OK (16413568 sectors)
23.4 GByte DATA LOST (49083520 sectors)
```

The card claimed 32GB but only ~7.8GB of real flash existed. Data written beyond that either wrapped around or was silently discarded. Write speed of 12 MB/s and read speed of 16.9 MB/s (below Class 10 spec) were additional indicators.

### Root cause

Counterfeit SD card — physically re-labeled lower-capacity flash sold as a higher-capacity card. Extremely common in markets where consumer electronics have weak counterfeit enforcement. The card was purchased locally from a general marketplace, not from a brand-verified retailer.

### Resolution

- Discarded the counterfeit card.
- Replaced with a verified authentic card from a Lazada LazMall verified brand store.
- **Every card now tested with H2testw before flashing**, regardless of source.

### Lessons

- Filesystem-level corruption on a supposedly-clean system is a strong signal to check the underlying storage, not just software.
- `dmesg | grep -iE "ext4|error|i/o"` is the fastest way to catch storage issues.
- H2testw is the definitive test for capacity fraud — it writes the entire card with test patterns and reads them back.
- Brand storefronts on Lazada/Shopee (LazMall verified) have higher authenticity rates than general sellers, though not 100%.

## Zero 2 W WiFi rejoin failure

### Symptom

After a reboot, the Pi's LED indicated a successful boot (solid green), but the router listed it as offline, and it did not respond to ping. HDMI output was absent.

### Diagnosis

The Pi Zero 2 W's WiFi (Broadcom BCM43436) is documented as unreliable for reconnection after certain reboot scenarios. Some observations:

- WiFi driver occasionally fails to re-associate after `systemd-networkd` restarts.
- Weak or congested 2.4 GHz signal exacerbates the failure rate.
- The Zero 2 W does not support 5 GHz at all (single-band 2.4 GHz radio only).

### Resolution during the incident

Immediate: power cycle the Pi (unplug 30 seconds, plug back in). If that failed, reflash the SD card.

Long-term recommendation: use a USB-to-Ethernet adapter on the Pi's data USB port for wired networking. Eliminates WiFi flakiness entirely, at the cost of one Ethernet cable run.

### Lessons

- WiFi is acceptable for user devices (which reconnect on their own) but not ideal for infrastructure the whole network depends on.
- For a 24/7 role, wired networking should be the default. The Zero 2 W is capable of this with a $5 adapter.

## SSH lockout via cloud-init drop-in override

Documented in detail in [SSH Hardening](03-ssh-hardening.md). Summary:

Editing `PasswordAuthentication no` in `/etc/ssh/sshd_config` appeared successful but had no effect. The drop-in file `/etc/ssh/sshd_config.d/50-cloud-init.conf` (created by Raspberry Pi Imager when a password was preset during flashing) contained `PasswordAuthentication yes`. Drop-in files load first and the first occurrence of a setting wins.

**Fix:** edit the drop-in file, not the main config.

**Verification tool:** `sudo sshd -T | grep -iE "passwordauthentication|permitrootlogin"` shows the effective runtime configuration regardless of which file it comes from.

## Gravity build failure due to blocklist size

Adding the full HaGeZi `tif.txt` (~2.2M domains) caused gravity build to fail on the Zero 2 W:

```
[✗] Unable to build gravity tree in /etc/pihole/gravity.db_temp
Error in 5th command line argument: database disk image is malformed
```

Root cause: insufficient RAM to complete the gravity tree build with that many domains. The database temp file was left in a partial state.

Resolution: switched to `tif.mini.txt` (~149k domains, same category, sized appropriately for 512MB RAM). Cleaned up the failed temp file and rebuilt:

```bash
sudo rm -f /etc/pihole/gravity.db_temp
sudo pihole -g
```

## VPN bypassing Pi-hole

Symptom: after configuring Pi-hole and verifying it worked on other devices, this laptop's `nslookup` still returned real IPs for known-blocked domains, with a server address of `172.17.3.1`.

Diagnosis: `172.17.x.x` is a common VPN internal DNS range. Confirmed via `ipconfig /all` that a VPN adapter was active.

Resolution: disable VPN, `ipconfig /release && /renew && /flushdns` to refresh DHCP lease and DNS cache. See [Network Config](04-network-config.md) for VPN + Pi-hole compatibility options.
