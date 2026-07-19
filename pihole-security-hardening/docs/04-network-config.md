# 04 — Network Configuration

Pi-hole only affects the network if devices are actually configured to use it for DNS. This is done at the router level so every device on the LAN is filtered without per-device configuration.

## Router: Huawei/ZTE (ISP-supplied fiber router)

3BB fiber routers in Thailand vary by hardware batch (Huawei HG-series, ZTE F-series, FiberHome). Menu paths differ but the settings are equivalent.

### Setting the DNS server

Log into the router at `http://192.168.1.1`. The DNS override lives in the **DHCP server** configuration — not the WAN configuration. Setting DNS at the WAN level only changes what the router itself uses; setting it in DHCP is what gets pushed to clients.

Typical location:
- Huawei: **LAN → DHCP Server Configuration → Primary DNS**
- ZTE: **Network → LAN → DHCP Server → DNS Server**

Set **Primary DNS** to the Pi's IP (`192.168.1.5`). Leave secondary DNS blank.

**Do not add a secondary DNS pointing at Google or the ISP.** Devices treat DNS servers as fallbacks and will route around Pi-hole to whichever answers first. This produces a subtle failure mode: blocking mostly works, but tracking domains still resolve intermittently.

### DHCP reservation for the Pi

The Pi's IP must remain constant. If DHCP re-assigns it, the DNS setting above points at nothing and the whole network loses DNS.

Location:
- Huawei: **LAN → DHCP Static IP Configuration**
- ZTE: **Network → LAN → DHCP Binding**

Bind the Pi's MAC address (from `ip link show wlan0` on the Pi) to `192.168.1.5`.

## Applying the change to existing devices

Devices only pick up the new DNS setting when their DHCP lease renews. To force immediate refresh:

- **Windows:** `ipconfig /release && ipconfig /renew && ipconfig /flushdns`
- **Phones/tablets:** toggle WiFi off and on
- **Everything else:** wait for lease renewal (typically 24 hours)

## Verifying per-device DNS routing

```
nslookup flurry.com
```

The Server line should show `192.168.1.5` (or `pi.hole`). The Address answer should be `0.0.0.0` (blocked).

Common outcomes:
- **Server `192.168.1.5`, answer `0.0.0.0`** — working correctly.
- **Server `192.168.1.1`, answer `0.0.0.0`** — router is acting as DNS relay to Pi-hole. Still filtered but Pi-hole loses per-device visibility.
- **Server not the Pi, real IP returned** — DNS is bypassing Pi-hole. Common causes: VPN client with hard-coded DNS, DoH/DoT enabled in the browser, device with hardcoded upstream (Chromecast, some smart TVs).

## VPN interaction

VPN clients tunnel DNS to their own servers, bypassing Pi-hole. Test:

```
nslookup flurry.com
# Server: 172.17.3.1 → VPN is intercepting DNS
```

Resolution options:

1. Configure the VPN client with a custom DNS pointing at the Pi (works when at home; WireGuard, Mullvad, Proton, IVPN all support this).
2. Enable split tunneling to exclude local LAN traffic from the tunnel.
3. Use a public ad-blocking DNS (NextDNS, AdGuard) via the VPN when away from home.

Some consumer VPNs (McAfee, some NordVPN configurations) do not permit custom DNS. In those cases the device is unfilterable while the VPN is active.

## Failure planning

### If Pi-hole is down

Symptom: WiFi still shows connected, but websites fail to load ("can't find server"). Cached lookups may work for a few minutes.

Recovery:
1. Log into router at `192.168.1.1` (works without DNS — it's an IP, not a name).
2. Clear the Primary DNS field in DHCP settings.
3. Apply — clients revert to ISP DNS on next lease renewal.
4. Debug the Pi without pressure.

### If the Pi is unreachable via SSH

See [Recovery Procedures](06-recovery-procedures.md) for the SD card side-door method.
