# How to Improve Wi-Fi and On-the-Move Privacy on Linux (MAC Randomization and IPv6 Privacy)

> **Published:** 2026-07-03. **Last verified:** 2026-07-03 on a live Fedora Linux
> 44 system running NetworkManager 1.56.1. NetworkManager settings were checked
> against the [NetworkManager reference](https://networkmanager.dev/docs/api/latest/)
> and the headless settings against the `systemd.network(5)`, `systemd.link(5)`,
> and Netplan documentation. Covers NetworkManager systems (Fedora 44, Ubuntu
> 26.04 Desktop, CachyOS, RHEL) and headless `systemd-networkd`/Netplan hosts
> (Ubuntu Server).

> When your laptop uses Wi-Fi, it hands the network a hardware MAC address that is
> unique and permanent, and it can also broadcast your hostname and settle on a
> stable IPv6 address. Networks and nearby observers use these to recognize your
> device across visits and across locations: the same MAC seen at a café last week
> and an airport today is the same laptop. This guide lowers those identifiers,
> so a device is harder to track as it moves between networks.

* * *

## What this protects, and what it does not

This is **link-layer privacy**. It changes what the local network and anyone
listening nearby can use to fingerprint your device on the first hop. Be clear
about the boundary:

- **It hides device identifiers from the local network:** the MAC address, the
  hostname, and (with IPv6 privacy) a long-lived address.
- **It does not hide your traffic, your public IP, the sites you visit, TLS SNI,
  or your browser fingerprint.** Routers past the first hop never saw your MAC
  anyway. For traffic privacy use the [WireGuard VPN](wireguard-vpn-self-hosted.md)
  and [Secure DNS](secure-dns-quad9-dot-dnssec.md) guides.
- **Some of it is already on.** Modern NetworkManager randomizes the MAC used
  while *scanning* by default, and Fedora randomizes the *connection* MAC per
  network by default. Step 1 shows you what your system already does.
- **There is no CIS or NIST control for this.** MAC randomization and IPv6 privacy
  addresses are not covered by the CIS Linux Benchmarks (whose IPv6 items are about
  disabling unused IPv6 and rejecting rogue router advertisements, a different
  concern). This guide is privacy hardening, not benchmark compliance, and it is
  presented honestly as such.

* * *

## The one trade-off that matters: random vs stable

A randomized MAC comes in two useful flavors, and picking the right one avoids most
of the friction:

- **`random`:** a brand-new MAC every time you connect. Best against tracking, but
  it **breaks captive portals** (the sign-in page ties your session to a MAC, so a
  new MAC logs you out) and **MAC allowlists** (networks that admit only known
  addresses), and it resets your DHCP lease each time.
- **`stable`:** a generated MAC that stays the same for a given network but differs
  between networks. It still hides your real hardware MAC, and it works with
  portals and allowlists because each network keeps seeing the same address. This
  is the sweet spot for a laptop that roams, and it is what Fedora uses by default
  (its `stable-ssid` setting keys the address to the network name).

Use `stable` (or `stable-ssid`) as your everyday default. Reach for `random` only
on networks where you actively want to defeat tracking and do not need a portal or
an allowlist.

* * *

## Step 1: See what your system already does

Compare your permanent hardware MAC with the address currently in use. If they
differ, randomization is already active:

```bash
# Permanent (burned-in) MAC of the Wi-Fi device
sudo ethtool -P wlp2s0
# Address currently in use
ip link show wlp2s0 | awk '/link\/ether/ {print $2}'
```

Replace `wlp2s0` with your interface (`ip -br link` lists them). Then check the
NetworkManager settings on your active connection:

```bash
nmcli -f 802-11-wireless.cloned-mac-address,ipv6.ip6-privacy,ipv4.dhcp-send-hostname \
  connection show "<connection-name>"
```

`nmcli connection show` (no name) lists your connections. A `cloned-mac-address` of
`--` means the connection uses the global default, which on stock NetworkManager is
`preserve` (your real MAC) but on Fedora is `stable-ssid` (already randomized).

* * *

## For NetworkManager systems (Fedora, Ubuntu Desktop, CachyOS, RHEL)

### Randomize the connection MAC

Set one connection to a stable per-network MAC:

```bash
nmcli connection modify "<connection-name>" wifi.cloned-mac-address stable
nmcli connection up "<connection-name>"
```

Or make it the default for **all** current and future Wi-Fi and Ethernet
connections with a drop-in. Create
`/etc/NetworkManager/conf.d/30-mac-privacy.conf`:

```ini
[device]
wifi.scan-rand-mac-address=yes

[connection]
wifi.cloned-mac-address=stable
ethernet.cloned-mac-address=stable
connection.stable-id=${CONNECTION}/${BOOT}
```

Then apply it with `sudo nmcli general reload`. The `${BOOT}` in `stable-id` makes
the generated MAC change on each reboot while staying stable within a session; drop
it if you want the same address to persist across reboots. For a network where you
want a fresh MAC every time, set that one connection to `random` instead:

```bash
nmcli connection modify "<captive-portal-free-network>" wifi.cloned-mac-address random
```

### Turn on IPv6 privacy (temporary addresses)

Without this, your IPv6 address is stable and trackable over time even if the MAC
changes. Enable temporary addresses (RFC 8981), which the system rotates and
prefers for outbound connections:

```bash
nmcli connection modify "<connection-name>" ipv6.ip6-privacy 2
nmcli connection up "<connection-name>"
```

The value `2` means "enabled, prefer temporary address" (`1` keeps a public address
preferred, `0` disables). This is separate from Fedora's default
`addr-gen-mode=stable-privacy`, which already stops the IPv6 address from embedding
your MAC but does not rotate it over time; the two work together.

### Stop leaking your hostname over DHCP

By default the DHCP client sends your machine's hostname, which a network can log to
recognize you across locations even behind a randomized MAC. Turn it off:

```bash
nmcli connection modify "<connection-name>" ipv4.dhcp-send-hostname no
nmcli connection modify "<connection-name>" ipv6.dhcp-send-hostname no
nmcli connection up "<connection-name>"
```

On NetworkManager 1.52 and newer (Fedora 44 and Ubuntu 26.04 both qualify) these two
per-family settings are consolidated behind `dhcp-send-hostname-v2`, but setting the
pair above is safe on every version and has the same effect.

* * *

## For headless and server hosts (systemd-networkd and Netplan)

A machine without NetworkManager (a typical Ubuntu Server) uses `systemd-networkd`,
often configured through Netplan. The same three protections apply.

**Randomize the MAC** with a `.link` file, `/etc/systemd/network/10-wifi-rand.link`:

```ini
[Match]
Type=wlan

[Link]
MACAddressPolicy=random
```

`MACAddressPolicy=random` is not the default, so you set it explicitly (`persistent`
keeps a stable generated address; `none` keeps the hardware MAC). Or in Netplan
(`/etc/netplan/*.yaml`):

```yaml
network:
  version: 2
  wifis:
    wlan0:
      macaddress: random
      dhcp4: true
      access-points:
        "your-ssid": {password: "your-password"}
```

**Turn on IPv6 privacy and DHCP anonymity** in the `systemd-networkd` `.network`
file:

```ini
[Match]
Name=eth0

[Network]
DHCP=yes
IPv6PrivacyExtensions=yes

[DHCPv4]
Anonymize=true
```

`IPv6PrivacyExtensions=yes` enables temporary addresses; `Anonymize=true` turns on
the RFC 7844 anonymity profile, which suppresses the hostname, sets the client
identifier to the MAC, and drops the vendor-class option in one switch. In Netplan,
`ipv6-privacy: true` on an interface enables the address rotation.

Apply Netplan changes with `sudo netplan apply`, or `systemd-networkd` changes with
`sudo systemctl restart systemd-networkd`.

* * *

## Verify

Reconnect and confirm the changes took effect:

```bash
# MAC in use should now differ from the ethtool -P permanent address
ip link show wlp2s0 | awk '/link\/ether/ {print $2}'

# IPv6: a "temporary" address should appear alongside the main one
ip -6 addr show dev wlp2s0 | grep -i temporary
```

A `temporary` (or `mngtmpaddr`) IPv6 address confirms privacy extensions are
active. A MAC that differs from the permanent one confirms randomization.

* * *

## Troubleshooting

- **A captive portal will not load, or keeps logging you out.** A per-connection
  `random` MAC is the usual cause. Switch that network to `stable`
  (`nmcli connection modify "<name>" wifi.cloned-mac-address stable`).
- **A network rejects you (MAC allowlist).** The network admits only registered
  addresses. Use `permanent` on that one connection, or register your stable MAC
  with the network's administrator.
- **The MAC never changes.** Some Wi-Fi drivers or firmware do not support
  randomization and silently ignore it. It is a hardware and driver limitation, not
  a configuration error.
- **The hostname is still sent.** On NetworkManager 1.52 and newer, confirm
  `dhcp-send-hostname-v2` is not overriding your per-family settings, or set it to
  `no`.

* * *

## How to roll back

```bash
# NetworkManager: restore defaults on a connection
nmcli connection modify "<connection-name>" wifi.cloned-mac-address preserve \
  ipv6.ip6-privacy 0 ipv4.dhcp-send-hostname yes ipv6.dhcp-send-hostname yes
nmcli connection up "<connection-name>"
# and remove the global drop-in if you added one
sudo rm -f /etc/NetworkManager/conf.d/30-mac-privacy.conf && sudo nmcli general reload
```

On headless hosts, remove the `.link`, `.network`, or Netplan additions and re-apply.

* * *

## What you achieved

- Your laptop presents a **randomized MAC** that is stable per network, so it works
  with captive portals and allowlists while hiding the permanent hardware address.
- **IPv6 temporary addresses** rotate, so a long-lived address no longer tracks you.
- The **hostname is no longer broadcast** over DHCP.
- You know the limit: this is first-hop, link-layer privacy that pairs with, but
  does not replace, a VPN and encrypted DNS.

* * *

## References

- [NetworkManager: Wi-Fi settings reference](https://networkmanager.dev/docs/api/latest/settings-802-11-wireless.html)
- [NetworkManager: IPv6 settings reference](https://networkmanager.dev/docs/api/latest/settings-ipv6.html)
- [NetworkManager.conf reference](https://networkmanager.dev/docs/api/latest/NetworkManager.conf.html)
- [`systemd.link(5)` (MACAddressPolicy)](https://www.freedesktop.org/software/systemd/man/latest/systemd.link.html)
- [`systemd.network(5)` (IPv6PrivacyExtensions, Anonymize)](https://www.freedesktop.org/software/systemd/man/latest/systemd.network.html)
- [Netplan configuration reference](https://netplan.readthedocs.io/en/stable/netplan-yaml/)
- [RFC 8981: Temporary Address Extensions for SLAAC in IPv6](https://www.rfc-editor.org/rfc/rfc8981)
- [RFC 7844: Anonymity Profiles for DHCP Clients](https://www.rfc-editor.org/rfc/rfc7844)
