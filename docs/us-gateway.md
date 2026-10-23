# US gateway on p73 (P73-US Wi-Fi)

p73 makes a Wi-Fi access point, **P73-US**. Its clients exit to the internet in the **US**
through NordVPN. Used for the Zeam Roku stick (Zeam's HLS CDN is US-only) and an iPhone.
The script is `bin/us-gateway`.

```
phone ──USB tether──► usb0 ─┐
                            p73 ── tun0 (OpenVPN, NordVPN US) ──► internet (US exit)
Roku / iPhone ──Wi-Fi──► wlan0 (hostapd + dnsmasq, 192.168.50.1/24, SSID P73-US)
```

## Use

| Command | Effect |
|---|---|
| `us-gateway up` | Start the AP, firewall/NAT and the tunnel |
| `us-gateway status` | AP state, DHCP clients, tunnel, exit country (tunnel vs. p73) |
| `us-gateway down` | Stop all and give `wlan0` back to ConnMan |
| `us-gateway vpn-up` / `vpn-down` | Restart / stop only the tunnel |
| `us-gateway server [cc]` | Get NordVPN's recommended OpenVPN-UDP server (default `us`) |
| `us-gateway auth` | Prompt for NordVPN **service** credentials, store them root-only |

The setup is **not persistent**. After a reboot, run `us-gateway up` again.

## Secrets (not in this repo — the remote is public)

- `/etc/us-gateway/wifi-passphrase` — WPA2 passphrase of P73-US (root, 0600, one line).
- `/etc/openvpn/nord/auth.txt` — NordVPN service credentials (root, 0600). Get them from
  Nord Account → NordVPN → Advanced settings → *Set up NordVPN manually*. They are not the login.

## How the routing works

- Only the AP subnet goes into the tunnel: `ip rule from 192.168.50.0/24 lookup 100`, and
  table 100 has `default dev tun0`. The p73's own traffic (SSH, Tailscale) stays on `usb0`, so the
  box stays reachable when the tunnel fails.
- **No leak:** table 100 also has `blackhole default metric 1000`. If `tun0` goes away, AP clients
  lose internet. They do not fall back to the Romanian uplink.
- Table 100 **must** contain `192.168.50.0/24 dev wlan0`. Without it, the "from 192.168.50.0/24"
  rule also catches the p73's own replies from 192.168.50.1 (dnsmasq DNS), sends them into the
  tunnel, and the clients show "no internet".
- dnsmasq forwards DNS to NordVPN's resolvers (103.86.96.100 / 103.86.99.100). Host routes send
  them through `tun0`.
- OpenVPN runs with `--route-nopull --route-noexec`. The script itself is the `--up`/`--down` hook
  (`us-gateway hook`).

## Gotchas found while building it (2026-10-23)

- **Windows could not do this.** On the Yoga, NordVPN uses NordLynx or `ovpn-dco` adapters.
  Windows Internet Connection Sharing cannot share them, and NordVPN does not fall back to its
  TAP adapter.
- **ConnMan's `tether wifi` does not work with this AX200.** Clients associate, but only ARP
  passes: no DHCP, no IP. Also, connmand runs with `--nodnsproxy`, so its tether gives clients a
  DNS server that does not answer. Use hostapd + dnsmasq.
- **Docker sets the `FORWARD` policy to DROP.** The accept rules go into `DOCKER-USER`.
- antiX has no `/etc/init.d/connman`, so do not plan on restarting ConnMan to change
  `main.conf`. `down` gives `wlan0` back with `connmanctl disable/enable wifi`.
- **Reaching the Roku for deploys:** join P73-US from the laptop. The Roku was at 192.168.50.84.
