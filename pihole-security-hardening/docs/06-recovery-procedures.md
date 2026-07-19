# 06 — Recovery Procedures

Documented recovery paths for common failure scenarios.

## SSH lockout — client key lost or corrupted

### Scenario

The client machine's SSH key is unavailable (lost, corrupted, or wrong file location) and password authentication has been disabled on the Pi. No open SSH session remains as a fallback.

### Recovery: SD card boot partition side-door

The Pi's root filesystem is ext4 and unreadable from Windows without additional tools. However, the `bootfs` FAT partition is readable and writable from any OS, and files there are read by the Pi during early boot. This provides a supported channel for injecting configuration changes without reflashing.

### Procedure

1. **Shut down the Pi.** Unplug the SD card.

2. **Insert the SD card into the client machine.** The `bootfs` partition appears as a mounted drive. Windows may prompt to format additional (ext4) partitions — cancel.

3. **Copy a fresh public key to the boot partition:**

   ```
   copy %USERPROFILE%\.ssh\id_ed25519.pub D:\newkey.pub
   ```

4. **Create an install script on the boot partition** that will run on next boot:

   In PowerShell (or write manually with correct LF line endings):
   
   ```powershell
   $s = "#!/bin/bash`nset -e`ninstall -d -m 700 -o pi -g pi /home/pi/.ssh`ncat /boot/firmware/newkey.pub >> /home/pi/.ssh/authorized_keys`nchown pi:pi /home/pi/.ssh/authorized_keys`nchmod 600 /home/pi/.ssh/authorized_keys`nsed -i 's| systemd.run=/boot/firmware/fix.sh||;s| systemd.run_success_action=reboot||;s| systemd.unit=kernel-command-line.target||' /boot/firmware/cmdline.txt`n"
   [IO.File]::WriteAllText("D:\fix.sh", $s)
   ```

5. **Modify `cmdline.txt`** (single line file, must remain a single line). Append at the end:

   ```
    systemd.run=/boot/firmware/fix.sh systemd.run_success_action=reboot systemd.unit=kernel-command-line.target
   ```

6. **Eject the card, boot the Pi.** The Pi runs the script on boot, installs the new key, cleans up `cmdline.txt` so subsequent boots are normal, and reboots.

7. **SSH in** normally with the new key.

### Security implications

This recovery path requires **physical possession of the SD card**. That is not a vulnerability — it is a universal property of unencrypted storage. The defenses against this are disk encryption (out of scope for a home DNS box) or physical security of the device.

The procedure is not exposed over the network. An attacker on the LAN or internet cannot use it.

### When to reflash instead

If the filesystem is also corrupted (see [Troubleshooting](05-troubleshooting.md)), reflashing is faster and cleaner than trying to repair in place. The router's DHCP reservation and DNS pointing remain valid across reflashes because they're keyed to MAC address and IP, not to any OS state.

## Whole-network DNS outage

### Scenario

Pi-hole is down (crashed, SD card failure, power outage). Every device on the LAN shows "internet is broken" — WiFi is connected but nothing loads.

### Recovery

1. **Access the router at `http://192.168.1.1`.** This works regardless of Pi-hole status because it's an IP address, not a domain name.

2. **Clear the Primary DNS field** in DHCP settings. Save.

3. **Devices resume normal internet** on their next DHCP lease renewal (or after WiFi toggle). Clients now use the ISP's default DNS.

4. **Debug the Pi** without pressure.

### Preparation

- Bookmark `http://192.168.1.1` on at least one device so it can be reached without DNS.
- Document the router admin credentials somewhere retrievable offline.

## Full rebuild from scratch

If the SD card fails or corruption is unrecoverable:

1. **Reflash a new SD card** with the same Imager configuration (same hostname, WiFi credentials, SSH enabled).
2. **Boot and connect via SSH** using the temporary password.
3. **Apply the apt mirror fix** (see [Installation](01-installation.md)).
4. **Install Pi-hole** and configure the same upstream DNS and blocklists.
5. **Re-add SSH public key** via the same `nano ~/.ssh/authorized_keys` method.
6. **Re-apply SSH hardening** via `/etc/ssh/sshd_config.d/50-cloud-init.conf`.
7. **Reinstall fail2ban.**

The router configuration (DNS pointing, DHCP reservation) requires no changes because the Pi will come up on the same IP.

Total rebuild time from a working SD card: approximately 30 minutes.

## Backup strategy

Currently manual — Pi-hole's Teleporter feature (Settings → Teleporter) exports the full configuration to a downloadable archive. This should be captured periodically and stored off-Pi.

Automated backup is planned as a follow-up hardening item.
