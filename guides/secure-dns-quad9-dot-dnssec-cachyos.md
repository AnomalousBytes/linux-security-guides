# How to Configure Secure DNS with Quad9, DNS-over-TLS, and DNSSEC on CachyOS

> **Last verified:** 2026-06-27 on CachyOS (rolling release) with KDE Plasma and
> `systemd` 259. Because CachyOS is a rolling release, your package versions may
> be newer.
> Tested with `systemd-resolved` and NetworkManager, the defaults on CachyOS.

> By default, DNS lookups are sent in **plaintext** over port 53. Anyone on the
> network path (your ISP, a public Wi-Fi operator, or an on-path attacker) can
> read every domain you look up, and can tamper with the answers.
> Pointing your system at **Quad9** with **DNS-over-TLS (DoT)** and **DNSSEC
> validation** encrypts those lookups, lets your machine verify it is really
> talking to Quad9, and rejects forged DNS records.

This guide sets up **Quad9** as your system resolver on CachyOS using
`systemd-resolved` and NetworkManager, with **DNS-over-TLS** and **DNSSEC**
turned on.

> **Note:** CachyOS is Arch-based, so unlike Fedora or Ubuntu it does not assume
> `systemd-resolved` is already running. CachyOS does configure NetworkManager to
> use it (see Step 1), but this guide makes sure the service is active first. The
> configuration steps are otherwise the same as on any `systemd-resolved` system
> (the [Fedora 44](secure-dns-quad9-dot-dnssec-fedora-44.md) and
> [Ubuntu 26.04](secure-dns-quad9-dot-dnssec-ubuntu-26.04.md) guides are close
> siblings).

* * *

## First, what each piece actually does

These three technologies solve **different** problems and are complementary.
Turning on one does not give you the others:

| Technology | What it protects | What it does **not** do |
| --- | --- | --- |
| **DNS-over-TLS (DoT)** | *Confidentiality + integrity in transit.* Encrypts the lookup between your machine and the resolver so the network path cannot read or alter it. | Does not prove the *answer* is authentic if the resolver itself is lying or misconfigured. |
| **DNSSEC** | *Authenticity + integrity of the data.* Cryptographic signatures let a validating resolver detect forged or tampered records (the lookup fails instead). | Does not encrypt anything. A DNSSEC-only lookup is still readable on the wire. |
| **Quad9 (the resolver)** | A recursive resolver, run by a non-profit, that **blocks known-malicious domains** and does **not** send EDNS Client Subnet (ECS) data. | Quad9 still *sees* your queries (it has to resolve them). It is not anonymity. See [Limitations](#limitations-what-this-does-and-does-not-hide). |

Used together: DoT hides and protects the lookup in transit, DNSSEC proves the
record was not forged, and Quad9 filters known threats.

* * *

## Prerequisites

- CachyOS with NetworkManager (the default on the desktop editions).
- Administrative privileges (`sudo`).
- Basic familiarity with the terminal.
- For the optional verification in Steps 6-7: `tcpdump`, `dig`, and `nc`, which
  are not always preinstalled. Install them with
  `sudo pacman -S --needed bind openbsd-netcat tcpdump` (`dig` is in `bind`, `nc`
  is in `openbsd-netcat`). The `resolvectl`-based checks need no extra packages.

> **If you use CachyOS-Welcome's encrypted-DNS option:** CachyOS-Welcome can set
> up DNS-over-HTTPS through `blocky`, which is a separate mechanism from the
> `systemd-resolved` DNS-over-TLS used in this guide. Run one or the other, not
> both. If you enabled the Welcome DoH option, disable it before following this
> guide so the two resolvers do not compete.

* * *

## Step 1: Make sure systemd-resolved is the active resolver

CachyOS ships NetworkManager configured to hand DNS to `systemd-resolved`, but on
an Arch-based system it is worth confirming the service is actually running.

Check whether it is active:

```bash
systemctl is-active systemd-resolved
```

If that prints `inactive` or `failed`, enable and start it, then restart
NetworkManager so it picks up the change:

```bash
sudo systemctl enable --now systemd-resolved
sudo systemctl restart NetworkManager
```

Confirm NetworkManager is using `systemd-resolved` as its DNS backend (CachyOS
sets this in `/usr/lib/NetworkManager/conf.d/dns.conf`):

```bash
NetworkManager --print-config | grep dns=
```

You should see `dns=systemd-resolved`. Finally, confirm the system resolver is
the local stub:

```bash
cat /etc/resolv.conf
```

You should see `nameserver 127.0.0.53`, and `resolvectl status` should report
that `systemd-resolved` is handling DNS. (The `Current DNS Server` field shows
your *upstream* resolver, which is your router now or `9.9.9.9` once you finish
this guide, not the local `127.0.0.53` stub listener.)

* * *

## Step 2: Back up the current configuration

Always keep a copy you can roll back to:

```bash
sudo cp /etc/systemd/resolved.conf /etc/systemd/resolved.conf.bak
```

* * *

## Step 3: Point systemd-resolved at Quad9 over DNS-over-TLS

Edit the resolver configuration:

```bash
sudo nano /etc/systemd/resolved.conf
```

Set the `[Resolve]` section to exactly this:

```ini
[Resolve]
DNS=9.9.9.9#dns.quad9.net 149.112.112.112#dns.quad9.net 2620:fe::fe#dns.quad9.net 2620:fe::9#dns.quad9.net
FallbackDNS=
DNSOverTLS=yes
DNSSEC=yes
```

> **Editing tip:** the shipped `resolved.conf` already contains a commented-out
> `[Resolve]` header with commented keys (`#DNS=`, `#DNSSEC=`, and so on). Edit
> the keys **under that existing header** (do not add a second `[Resolve]`
> section), and make sure no other uncommented `DNS=` / `DNSOverTLS=` /
> `DNSSEC=` lines remain.

Why each line is written this way:

- **`DNS=`** lists all four Quad9 resolvers that provide **malware blocking and
  DNSSEC validation** (the address set Quad9 recommends as its default), both
  IPv4 (`9.9.9.9`, `149.112.112.112`) and IPv6 (`2620:fe::fe`, `2620:fe::9`), so
  the setup works on dual-stack networks. **List them under `DNS=`, not
  `FallbackDNS=`.** `FallbackDNS=` is only consulted when *no other DNS server is
  known*, so servers placed there while `DNS=` is set would essentially never be
  used.
- **`#dns.quad9.net`** is the critical part for DoT. The `address#server_name`
  form tells `systemd-resolved` to validate the server's **TLS certificate
  against that hostname** and to send it as the **SNI**. Without the suffix the
  certificate is only checked against the server's IP address and SNI is
  disabled, a weaker authentication guarantee. `dns.quad9.net` is the official
  TLS authentication name for this Quad9 service.
- **`FallbackDNS=`** is intentionally **empty**. `systemd-resolved` ships with
  compiled-in public fallback resolvers (for example Quad9, Cloudflare, and
  Google) that it queries **in plaintext**, but only when *no* `DNS=` or per-link
  servers are known. Emptying it guarantees those can never be used, even if your
  `DNS=` line somehow fails to load, so the system can never silently leak
  lookups to a third party in cleartext. (This is not a runtime failover: if
  Quad9 itself is unreachable, lookups simply fail rather than falling back,
  which is the correct posture for a hardening guide.)
- **`DNSOverTLS=yes`** is *strict* mode: lookups are encrypted or they fail.
  The alternative, `opportunistic`, silently falls back to plaintext and is
  vulnerable to a trivial downgrade attack, so it is **not** used here.
- **`DNSSEC=yes`** enables strict local DNSSEC validation: forged or
  badly-signed records are rejected. Setting `DNSSEC=yes` makes strict local
  validation explicit and deterministic regardless of your build's default. The
  intermediate `allow-downgrade` mode is weak (an on-path attacker can silently
  switch it off), so `yes` is the deliberate hardening choice. Be aware that strict validation can occasionally
  break resolution for a minority of real sites with imperfect DNSSEC (and for
  captive portals). See [Troubleshooting](#troubleshooting) if that happens.

Save the file, then apply it:

```bash
sudo systemctl restart systemd-resolved
```

* * *

## Step 4: Stop your router's DHCP DNS from being used too

This step is easy to miss and is what makes the setup actually private.

`systemd-resolved` queries the global `DNS=` servers **in parallel with the
per-link DNS servers of your default-route connection**, which a normal
Wi-Fi/Ethernet connection picked up from DHCP is. NetworkManager pushes those
per-link (router) servers into `systemd-resolved` over its D-Bus API, so if you
skip this step some lookups can still be sent **in plaintext to your router**,
bypassing Quad9 and DoT entirely.

Tell NetworkManager to ignore the DNS servers advertised by DHCP so that only
your Quad9-over-DoT configuration is used. **Use one of the methods below.**

**Option A, per connection with `nmcli` (recommended, deterministic):**

List your connections, then modify the one you use (replace `"<name>"`):

```bash
nmcli connection show
sudo nmcli connection modify "<name>" \
  ipv4.ignore-auto-dns yes \
  ipv6.ignore-auto-dns yes
sudo nmcli connection up "<name>"
```

With the global `DNS=` set in Step 3, this is enough: the link now provides no
DNS servers of its own, so all queries fall through to your Quad9 DoT
configuration.

**Option B, pin Quad9 directly on the connection (also forces per-connection DoT):**

If you would rather attach Quad9 to the connection itself (useful on machines
that do not set a global `DNS=`), include the `#dns.quad9.net` hostname so
certificate validation still works, and force DoT on that connection:

```bash
sudo nmcli connection modify "<name>" \
  ipv4.dns "9.9.9.9#dns.quad9.net,149.112.112.112#dns.quad9.net" \
  ipv6.dns "2620:fe::fe#dns.quad9.net,2620:fe::9#dns.quad9.net" \
  ipv4.ignore-auto-dns yes ipv6.ignore-auto-dns yes \
  connection.dns-over-tls yes
sudo nmcli connection up "<name>"
```

**Option C, KDE Plasma GUI:**

CachyOS defaults to KDE Plasma. In the network applet, open **Configure Network
Connections**, select your connection, and open the **IPv4** tab. Set the
**Method** so DHCP provides the address but not DNS (in the KDE connection editor
this is the *Automatic (Only addresses)* method), then enter the DNS servers
manually. Repeat on the **IPv6** tab, then apply and reconnect.

> **Avoid the common mistake:** the KDE GUI lets you enter only plain IPs
> (for example `9.9.9.9`). It cannot express the `#dns.quad9.net` hostname needed
> for TLS certificate-name validation, and it does not by itself force strict DoT
> on the connection. If you use the GUI, you are relying entirely on the global
> `DNSOverTLS=yes` / `DNS=` from Step 3 for the encryption and certificate name,
> so making DHCP DNS be ignored matters more than the IPs you type. For full
> per-connection certificate validation, prefer **Option A** or **Option B**.

* * *

## Step 5: Verify the configuration

```bash
resolvectl status
```

In the **Global** section (and on your active **Link**) you should see something
like:

```
       Protocols: +DefaultRoute +LLMNR -mDNS +DNSOverTLS DNSSEC=yes/supported
Current DNS Server: 9.9.9.9#dns.quad9.net
       DNS Servers: 9.9.9.9#dns.quad9.net 149.112.112.112#dns.quad9.net 2620:fe::fe#dns.quad9.net 2620:fe::9#dns.quad9.net
```

On `systemd` 259 the DNSSEC status is **folded into the `Protocols` line** as
`DNSSEC=yes/supported`, with no separate `DNSSEC setting:` line. The exact layout
can vary between `systemd` versions; the parts that matter are `+DNSOverTLS`,
`DNSSEC=yes/supported`, and that the Quad9 servers (with the `#dns.quad9.net`
suffix) are the ones in use.

* * *

## Step 6: Confirm traffic is encrypted (port 853, not 53)

These checks use `tcpdump` (install it first if needed, see Prerequisites). Each
`tcpdump` command runs until you press **Ctrl-C**, so you will need **two
terminals**: one capturing, one generating a lookup. The commands capture on all
interfaces (`-i any`); if you prefer to watch a single interface, find it with
`ip -br link` (which lists all interfaces) and replace `-i any` with `-i <iface>`.

**Negative check.** No plaintext DNS should leave your machine to an *external*
server on port 53. Loopback traffic to the local stub (`127.0.0.53` /
`127.0.0.54`) is normal, so exclude all loopback:

```bash
sudo tcpdump -n -i any 'port 53 and not net 127.0.0.0/8'
```

This should stay quiet. Press **Ctrl-C** to stop.

**Positive check.** Encrypted DNS to Quad9 should appear on port 853. In one
terminal start the capture (the active server can be any of the four Quad9
addresses, so match the port, not a single host):

```bash
sudo tcpdump -n -i any 'port 853'
```

In a second terminal, flush the cache so the lookup really goes upstream, then
query:

```bash
sudo resolvectl flush-caches
resolvectl query example.com
```

You should see TLS traffic to a Quad9 address on 853. Activity on 853, and
nothing on port 53 to an external server, confirms DoT is carrying your lookups.

* * *

## Step 7: Confirm DNSSEC validation

Use the well-known DNSSEC test pair: `sigok` is correctly signed and must
resolve; `sigfail` is deliberately broken and must be rejected.

**Authoritative check (version-independent):**

```bash
resolvectl query sigok.verteiltesysteme.net
resolvectl query sigfail.verteiltesysteme.net
```

Expected:

- `sigok.verteiltesysteme.net` resolves, and the output includes
  **`-- Data is authenticated: yes`**. (The full line also reports
  `Data was acquired via local or encrypted transport: yes`. That second flag
  reflects DNS-over-TLS, not DNSSEC.)
- `sigfail.verteiltesysteme.net` **fails** with a message like
  *"... resolve call failed: DNSSEC validation failed: invalid"* (the exact
  reason token can vary).

**Optional `dig` cross-check** (needs `dig`, see Prerequisites):

```bash
dig sigfail.verteiltesysteme.net    # expect status: SERVFAIL
dig sigok.verteiltesysteme.net      # expect an answer
```

> Why `resolvectl query` is the reliable test rather than the `dig` `ad`
> (Authenticated Data) flag: the `127.0.0.53` stub's `ad`-bit handling is
> unreliable. It can be cleared exactly when a query sets the DO bit to request
> DNSSEC data (for example `dig +dnssec`), and this behaviour has shifted between
> `systemd` versions. `resolvectl`'s explicit *"Data is authenticated"* line is
> unambiguous on every current version.

Note that Quad9's recommended service also validates DNSSEC **upstream**, so a
failed `sigfail` lookup confirms DNSSEC is enforced **end-to-end** (Quad9 plus
your local resolver). Local `DNSSEC=yes` adds defense-in-depth on top of Quad9.

* * *

## Limitations: what this does (and does not) hide

Encrypted DNS is an important layer, but it is **not** full browsing privacy. Be
honest with yourself about the threat model:

- **Destination IPs are still visible.** After the lookup, your device connects
  to the site's IP address, which your ISP can see and often map back to a site.
- **TLS SNI usually leaks the hostname.** Most HTTPS handshakes still send the
  site name in cleartext in the TLS `ClientHello` unless **Encrypted Client
  Hello (ECH)** is in use end-to-end.
- **Quad9 still sees your queries.** You are moving trust from your ISP to Quad9
  (a non-profit that states it does not log personal data and disables ECS), not
  eliminating it.
- **This is not anonymity.** For that you need a VPN or Tor, which change *where*
  your traffic appears to originate.

In short: DoT + DNSSEC removes one significant metadata leak and prevents
tampering, and pairs well with ECH and (if you need it) a VPN.

* * *

## Troubleshooting

- **A specific site suddenly fails to resolve.** Its zone may have broken
  DNSSEC, or you may be behind a **captive portal** (hotel or airport Wi-Fi)
  whose sign-in page hijacks DNS. Captive portals frequently break strict
  DoT/DNSSEC, and because `DNSOverTLS=yes` with an empty `FallbackDNS=` is
  strict, the portal's own login page may not even resolve. To get online,
  temporarily relax the settings: set `DNSOverTLS=opportunistic` (or `no`) and
  `DNSSEC=allow-downgrade` in `resolved.conf`, run
  `sudo systemctl restart systemd-resolved`, sign in, then **revert to `yes` /
  `yes`** and restart again.
- **All DNS stops working.** Because `FallbackDNS=` is empty, an unreachable
  Quad9 means no resolution by design. Check connectivity to Quad9 on 853:
  `nc -vz 9.9.9.9 853`.
- **DNS works only after restarting `systemd-resolved`.** On some CachyOS setups
  resolution can fail at boot until the service settles. Confirm it is enabled
  (`systemctl is-enabled systemd-resolved`) and, if the problem persists, check
  the service ordering with `systemctl status systemd-resolved NetworkManager`.
- **Queries still appear on port 53 to an external server.** A per-link/DHCP DNS
  server is still configured. Recheck Step 4 (`ignore-auto-dns`) and confirm with
  `resolvectl status` that no Link lists your router's DNS address.
- **`resolvectl` shows `DNSOverTLS` is off on a Link.** A VPN or another tool
  (including CachyOS-Welcome's `blocky` DoH option) may own that link's DNS. Pick
  one resolver and disable the others.

* * *

## Optional: VPNs, Tailscale, and Docker

- **Tailscale** installs its own resolver (`100.100.100.100`) and, with MagicDNS
  *"Override local DNS"* enabled, captures **all** queries ahead of your Quad9
  config while the VPN is up. If you want Quad9-over-DoT to remain in effect,
  either disable that override or set Quad9 as the global nameserver in the
  Tailscale admin DNS settings.
- **Other VPNs** typically push their own DNS for the duration of the
  connection; that is usually desirable (it keeps lookups inside the tunnel).
- **Docker** containers inherit DNS from the host unless overridden in
  `/etc/docker/daemon.json`.

* * *

## How to roll back

```bash
sudo mv /etc/systemd/resolved.conf.bak /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved
# If you changed a NetworkManager connection in Step 4 (Option A or B):
sudo nmcli connection modify "<name>" ipv4.ignore-auto-dns no ipv6.ignore-auto-dns no \
  ipv4.dns "" ipv6.dns "" connection.dns-over-tls default
sudo nmcli connection up "<name>"
```

If you used the **KDE GUI** (Option C), open the connection editor again, set the
IPv4 and IPv6 **Method** back to *Automatic (DHCP)*, clear any servers you typed,
then apply and reconnect.

* * *

## What you achieved

- DNS lookups go to **Quad9's recommended malware-blocking, DNSSEC-validating
  resolvers** (`9.9.9.9`, `149.112.112.112`, and their IPv6 equivalents).
- Lookups are **encrypted and authenticated** with DNS-over-TLS (strict),
  validated against the `dns.quad9.net` certificate.
- Responses are **DNSSEC-validated**, so forged records are rejected.
- Your **router's plaintext DHCP DNS is suppressed**, so lookups are not leaked.
- **EDNS Client Subnet (ECS)** is disabled by Quad9 on this service for better
  privacy.

* * *

## References

- [Quad9: Service Addresses & Features](https://www.quad9.net/service/service-addresses-and-features/)
- [systemd `resolved.conf(5)` manual](https://www.freedesktop.org/software/systemd/man/latest/resolved.conf.html)
- [ArchWiki: systemd-resolved](https://wiki.archlinux.org/title/Systemd-resolved)
- [ArchWiki: NetworkManager (systemd-resolved integration)](https://wiki.archlinux.org/title/NetworkManager#systemd-resolved)
- [`nm-settings-nmcli(5)`: NetworkManager connection settings](https://networkmanager.dev/docs/api/latest/nm-settings-nmcli.html)
- [DNSSEC (ICANN)](https://www.icann.org/resources/pages/dnssec-qaa-2014-01-29-en)
