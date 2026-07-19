# Pi-hole Security Hardening

Network-wide DNS filtering with security-oriented hardening, built on a Raspberry Pi Zero 2 W.

This project documents the end-to-end deployment of Pi-hole as a home network's authoritative DNS server, followed by a series of hardening measures aimed at making the box appropriate for a small production role. The write-up includes the reasoning behind each choice, the operational issues encountered during the build, and the recovery procedures used to resolve them.

## Overview

- **Platform:** Raspberry Pi Zero 2 W, Raspberry Pi OS Lite (32-bit, Debian Trixie)
- **Role:** Authoritative DNS resolver for a home LAN (~10-15 devices)
- **Blocklists:** HaGeZi Multi Normal + PopupAds + Fake + Threat Intelligence (mini)
- **Hardening:** SSH key-only authentication, root login disabled, fail2ban intrusion prevention
- **Network integration:** Router-level DNS pointing, DHCP reservation, fallback plan documented

## Repository contents

| File | Purpose |
|---|---|
| [docs/01-installation.md](docs/01-installation.md) | Pi-hole install, apt mirror correction |
| [docs/02-blocklist-strategy.md](docs/02-blocklist-strategy.md) | Why HaGeZi over the default StevenBlack list |
| [docs/03-ssh-hardening.md](docs/03-ssh-hardening.md) | Key-based auth, cloud-init override fix, fail2ban |
| [docs/04-network-config.md](docs/04-network-config.md) | Router DNS pointing, DHCP reservation, fallback |
| [docs/05-troubleshooting.md](docs/05-troubleshooting.md) | Counterfeit SD card discovery, WiFi rejoin failures |
| [docs/06-recovery-procedures.md](docs/06-recovery-procedures.md) | SSH lockout recovery via SD card boot partition |
| [configs/](configs/) | Example config file diffs |
| [screenshots/](screenshots/) | Dashboard and configuration screenshots |

## Skills demonstrated

- Linux system administration (Debian/Raspberry Pi OS, systemd, apt)
- DNS configuration and troubleshooting (Pi-hole, dig, nslookup)
- SSH hardening (key management, sshd configuration, drop-in file precedence)
- Network configuration (DHCP, DNS delegation, router integration)
- Security defense-in-depth (key auth + fail2ban + firewall planning)
- Operational recovery (SD card side-door key installation, filesystem-level debugging)
- Hardware verification (counterfeit SD card detection via H2testw)
- Documentation of failure modes and recovery paths

## Status

- [x] Pi-hole install and blocklist configuration
- [x] Router DNS pointing and DHCP reservation
- [x] SSH key-only authentication
- [x] fail2ban configuration
- [ ] ufw firewall rules
- [ ] Unattended-upgrades for automated security patches
- [ ] Unbound recursive resolver (encrypted upstream DNS)
- [ ] Configuration backup automation

## Author

[Surin-Learning-Lab](https://github.com/Surin-Learning-Lab)

## License

MIT — see [LICENSE](LICENSE).
