# Screenshots

Drop screenshot files in this folder as you capture them. Suggested filenames referenced by the docs:

| Filename | What to capture | Referenced in |
|---|---|---|
| ![Dashboard](./01-dashboard-overview.png) | Pi-hole main dashboard (queries, blocked % top clients) | 02-blocklist-strategy.md |
| ![Lists](./lists.png) | Lists page showing the four HaGeZi lists | 02-blocklist-strategy.md |
| ![Gravity Count](./gravity_count.png) | Terminal output of `sudo pihole -g` showing ~312k domains | 02-blocklist-strategy.md |
| ![Query Log](./query-log.png) | Query Log filtered to blocked entries | 02-blocklist-strategy.md |
| ![SSH Key Test](./05-ssh-key-test.png) | Terminal showing successful key-based login | 03-ssh-hardening.md |
| ![SSH Password Rejected](./06-ssh-password-rejected.png) | Terminal showing `Permission denied (publickey)` on password attempt | 03-ssh-hardening.md |
| ![Fail2ban Status](./07-fail2ban-status.png) | Output of `sudo fail2ban-client status sshd` | 03-ssh-hardening.md |
| ![Router DNS Config](./08-router-dns-config.png) | Router DHCP page showing Pi's IP as Primary DNS | 04-network-config.md |
| ![Router DHCP Reservation](./09-router-dhcp-reservation.png) | Router static IP binding for the Pi's MAC | 04-network-config.md |
| ![nslookup Verify](./10-nslookup-verify.png) | `nslookup` showing Pi.hole as server and `0.0.0.0` answer | 04-network-config.md |
| ![H2testw Fake Card](./11-h2testw-fake-card.png) | H2testw report identifying the counterfeit card | 05-troubleshooting.md |
| ![H2testw Good Card](./12-h2testw-good-card.png) | H2testw report showing a verified authentic card | 05-troubleshooting.md |


Once captured, reference them in the docs like:

```markdown
![Dashboard](../screenshots/01-dashboard-overview.png)
```

## Redaction

Before committing screenshots, blur or crop any of the following:

- Full router admin URLs beyond `192.168.1.1`
- WiFi SSID names (personal identification)
- Any internal MAC addresses (partially — first 3 octets are vendor OUI and fine; last 3 identify the device)
- Public IPs (WAN address in router pages)
- Full email addresses in Pi-hole admin
- Detailed logs showing external destinations from other household devices
