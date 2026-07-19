# 03 — SSH Hardening

Password-based SSH is the default on a fresh Raspberry Pi OS install. This chapter documents the process of moving to key-based authentication, disabling password login, and adding fail2ban as a defense-in-depth layer.

## Threat model

An always-on device with SSH exposed to the LAN (and, if the router ever exposes it, the internet) is a permanent brute-force target. Passwords are guessable; keys are not. The goal is:

- No password-based login over SSH
- No root login
- Automated banning of brute-force attempts

## Step 1 — Generate an SSH key pair (client-side)

On the Windows client:

```
ssh-keygen -t ed25519
```

- Ed25519 is chosen over RSA for smaller size and better security margins.
- Accept the default file location (`C:\Users\<user>\.ssh\id_ed25519`).
- A passphrase is optional; used here for defense against laptop theft.

Verify the files exist:

```
dir C:\Users\<user>\.ssh
```

Expected output includes both `id_ed25519` (private) and `id_ed25519.pub` (public).

## Step 2 — Copy the public key to the Pi

Method used (edit the file directly rather than pipe through SSH, which can silently fail):

```bash
# On the Pi:
mkdir -p ~/.ssh && chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
```

Then paste the contents of `id_ed25519.pub` (one long line starting `ssh-ed25519 AAAA...`), save (Ctrl+O, Enter, Ctrl+X), and set permissions:

```bash
chmod 600 ~/.ssh/authorized_keys
```

Verify:

```bash
cat ~/.ssh/authorized_keys
wc -l ~/.ssh/authorized_keys
```

Expected: one line of key data, `1` line count.

## Step 3 — Verify key login works BEFORE disabling passwords

**This is the step that prevents lockouts.** From a new terminal window on the client, force key-only authentication:

```
ssh -o PreferredAuthentications=publickey -o PasswordAuthentication=no pi@192.168.1.5
```

If this succeeds, the key is correctly installed. If it fails with `Permission denied (publickey)`, the key was not written correctly — fix before proceeding. Password login is still enabled at this point, so the previous SSH session is preserved as a recovery path.

## Step 4 — Disable password authentication

The main SSH config file (`/etc/ssh/sshd_config`) contains the `PasswordAuthentication` setting, but on Raspberry Pi OS this is overridden by a drop-in file. Check where the setting lives:

```bash
sudo grep -rn "PasswordAuthentication" /etc/ssh/sshd_config /etc/ssh/sshd_config.d/
```

Typical output:

```
/etc/ssh/sshd_config:57:PasswordAuthentication no
/etc/ssh/sshd_config.d/50-cloud-init.conf:1:PasswordAuthentication yes
```

**Files in `/etc/ssh/sshd_config.d/` are loaded first, and the first occurrence of a setting wins.** The cloud-init file was created by Raspberry Pi Imager when a password was preset during flashing, and it overrides the main config.

Fix: edit the drop-in file, not the main config.

```bash
sudo nano /etc/ssh/sshd_config.d/50-cloud-init.conf
```

Change:
```
PasswordAuthentication yes
```
To:
```
PasswordAuthentication no
```

Also disable root login in the main config:

```bash
sudo nano /etc/ssh/sshd_config
```

Change:
```
#PermitRootLogin prohibit-password
```
To:
```
PermitRootLogin no
```

Verify effective config:

```bash
sudo sshd -T | grep -iE "passwordauthentication|permitrootlogin"
```

Expected:
```
passwordauthentication no
permitrootlogin no
```

Restart SSH:

```bash
sudo systemctl restart ssh
```

Existing sessions are not terminated by the restart. Test the change from a **new** terminal window before closing the current one.

## Step 5 — Verify both angles

Positive test (should succeed):
```
ssh pi@192.168.1.5
```

Negative test (should fail):
```
ssh -o PubkeyAuthentication=no pi@192.168.1.5
```

Expected result of the negative test:
```
pi@192.168.1.5: Permission denied (publickey).
```

If both behave as expected, password authentication is fully disabled and key auth is confirmed working.

## Step 6 — Install fail2ban

```bash
sudo apt install fail2ban -y
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

Expected status output:

```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     0
|  `- Journal matches:  _SYSTEMD_UNIT=ssh.service + _COMM=sshd
`- Actions
   |- Currently banned: 0
   |- Total banned:     0
   `- Banned IP list:
```

fail2ban watches the systemd journal for SSH failures and applies iptables/nftables bans automatically. Default policy is 5 failures within 10 minutes = 10-minute ban.

### Why fail2ban when passwords are disabled

Password auth is off, so password brute-forcing already fails. fail2ban still adds value:

- Suppresses log noise from internet-wide SSH scanners
- Bans IPs attempting key-based brute force (rare but possible)
- Extensible to other services (Pi-hole web UI, nginx, etc.)
- Documents active response to hostile probes

## Backup and recovery

The client's private key is the sole means of access. If lost, recovery requires physical access to the SD card — see [Recovery Procedures](06-recovery-procedures.md).

Recommended: back up `id_ed25519` and `id_ed25519.pub` to an offline location (encrypted USB stick, password manager with file attachment support). **Never commit the private key to git or store it in cloud sync.**

## Configuration diff summary

See [`configs/sshd_config.d-50-cloud-init.conf.example`](../configs/sshd_config.d-50-cloud-init.conf.example) and [`configs/fail2ban-jail.local.example`](../configs/fail2ban-jail.local.example) for reference.
