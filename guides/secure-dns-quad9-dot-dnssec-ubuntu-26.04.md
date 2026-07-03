# How to Configure Secure DNS with Quad9, DNS-over-TLS, and DNSSEC on Ubuntu 26.04 LTS

> **Last verified:** 2026-06-27 on Ubuntu 26.04 LTS ("Resolute Raccoon", GNOME),
> `systemd` 259.
> Tested with `systemd-resolved` and NetworkManager, the defaults on Ubuntu
> 26.04 Desktop.

> By default, DNS lookups are sent in **plaintext** over port 53. Anyone on the
> network path (your ISP, a public Wi-Fi operator, or an on-path attacker) can
> read every domain you look up, and can tamper with the answers.
> Pointing your system at **Quad9** with **DNS-over-TLS (DoT)** and **DNSSEC
> validation** encrypts those lookups, lets your machine verify it is really
> talking to Quad9, and rejects forged DNS records.

This guide sets up **Quad9** as your system resolver on Ubuntu 26.04 LTS using
`systemd-resolved` and NetworkManager, with **DNS-over-TLS** and **DNSSEC**
turned on.

> **Note:** The steps are nearly identical on any Linux distribution that uses
> `systemd-resolved` (the [Fedora 44 guide](secure-dns-quad9-dot-dnssec-fedora-44.md)
> is a close sibling). Only the desktop's network GUI and the default interface
> names differ between distributions.

* * *

## First, what each piece actually does

These three technologies solve **different** problems and are complementary. Turning on one does not give you the others:

| Technology | What it protects | What it does **not** do |
| --- | --- | --- |
| **DNS-over-TLS (DoT)** | *Confidentiality + integrity in transit.* Encrypts the lookup between your machine and the resolver so the network path cannot read or alter it. | Does not prove the *answer* is authentic if the resolver itself is lying or misconfigured. |
| **DNSSEC** | *Authenticity + integrity of the data.* Cryptographic signatures let a validating resolver detect forged or tampered records (the lookup fails instead). | Does not encrypt anything. A DNSSEC-only lookup is still readable on the wire. |
| **Quad9 (the resolver)** | A recursive resolver, run by a non-profit, that **blocks known-malicious domains** and does **not** send EDNS Client Subnet (ECS) data. | Quad9 still *sees* your queries (it has to resolve them). It is not anonymity. See [Limitations](#limitations-what-this-does-and-does-not-hide). |

Used together: DoT hides and protects the lookup in transit, DNSSEC proves the
record was not forged, and Quad9 filters known threats.

* * *

## Prerequisites

- Ubuntu 26.04 LTS Desktop with `systemd-resolved` enabled (the default).
- Administrative privileges (`sudo`).
- Basic familiarity with the terminal.
- For the optional verification in Steps 6-7: `tcpdump`, `dig`, and `nc`, which
  are not always preinstalled. Install them with
  `sudo apt install -y bind9-dnsutils netcat-openbsd tcpdump` (`dig` is in
  `bind9-dnsutils`, `nc` is in `netcat-openbsd`). The `resolvectl`-based checks
  need no extra packages.

* * *

## Step 1: Confirm systemd-resolved is in use

```bash
resolvectl status
```

- If `resolvectl status` reports **`resolv.conf mode: stub`**, then
  `systemd-resolved` is handling DNS for the system. (The `Current DNS Server`
  field shows your *upstream* resolver, which is your router now or `9.9.9.9` once you
  finish this guide, not the local `127.0.0.53` stub listener.)
- Confirm `/etc/resolv.conf` points at that stub:

```bash
ls -l /etc/resolv.conf
```

You should see a symlink to the systemd stub, for example:

```
/etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf
```

(An absolute target, `/run/systemd/resolve/stub-resolv.conf`, is equivalent.)
If it is **not** such a symlink, restore it:

```bash
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

> On a default Ubuntu 26.04 Desktop install, NetworkManager already routes DNS
> through `systemd-resolved` because it auto-detects this symlink, so no
> `dns=systemd-resolved` edit to `NetworkManager.conf` is required. (That edit is
> only needed where `/etc/resolv.conf` is *not* a recognized systemd resolver
> symlink, e.g. some server/`netplan` + `systemd-networkd` setups. To force it
> explicitly, add `dns=systemd-resolved` under `[main]` in
> `/etc/NetworkManager/NetworkManager.conf` and restart NetworkManager.)

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
> `[Resolve]` header with commented keys (`#DNS=`, `#DNSSEC=`, ...). Edit the keys
> **under that existing header** (do not add a second `[Resolve]` section), and
> make sure no other uncommented `DNS=` / `DNSOverTLS=` / `DNSSEC=` lines remain.

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
- **`FallbackDNS=`** is intentionally **empty**. Upstream `systemd` compiles in
  public fallback resolvers (Cloudflare, Google, and Quad9) that are consulted
  only when *no* `DNS=` or per-link servers are known. Ubuntu builds `systemd`
  with that list empty, so a stock Ubuntu system has nothing to fall back to.
  Keeping the explicit empty line still documents the intent, and it makes the
  same file safe on builds that do include fallbacks (or if your `DNS=` line
  ever fails to load): a resolver you did not choose can never be consulted. (This is not a runtime failover: if Quad9 itself is
  unreachable, lookups simply fail rather than falling back, which is the
  correct posture for a hardening guide.)
- **`DNSOverTLS=yes`** is *strict* mode: lookups are encrypted or they fail.
  The alternative, `opportunistic`, silently falls back to plaintext and is
  vulnerable to a trivial downgrade attack, so it is **not** used here.
- **`DNSSEC=yes`** enables strict local DNSSEC validation: forged or
  badly-signed records are rejected. Ubuntu ships `systemd-resolved` with DNSSEC
  validation effectively **disabled** (`DNSSEC=no`), and the intermediate `allow-downgrade` mode is also weak (which an on-path attacker can silently
  switch off), so `yes` is a deliberate hardening choice. Be aware that strict
  validation can occasionally break resolution for a minority of real sites with
  imperfect DNSSEC (and for captive portals). See
  [Troubleshooting](#troubleshooting) if that happens.

Save the file, then apply it:

```bash
sudo systemctl restart systemd-resolved
```

* * *

## Step 4: Stop your router's DHCP DNS from being used too

This step is easy to miss, and it is what keeps every lookup on the resolver
you chose.

`systemd-resolved` queries the global `DNS=` servers **in parallel with the
per-link DNS servers of your default-route connection**, which a normal
Wi-Fi/Ethernet connection picked up from DHCP is. NetworkManager pushes those
per-link (router) servers into `systemd-resolved` over its D-Bus API, so if you
skip this step some lookups can still be sent **to your router** instead of
Quad9. The global `DNSOverTLS=yes` from Step 3 does still apply to per-link
servers, so those lookups are not plaintext, but they target a server you did
not choose, with no hostname for certificate validation, and they bypass
Quad9's filtering.

Tell NetworkManager to ignore the DNS servers advertised by DHCP so that only
your Quad9-over-DoT configuration is used. **Use one of the methods below.**

**Option A, per connection with `nmcli` (recommended, deterministic):**

List your connections, then modify the one you use (replace `"<name>"`):

```bash
nmcli connection show
sudo nmcli connection modify "<name>" \
  ipv4.ignore-auto-dns yes \
  ipv6.ignore-auto-dns yes
nmcli connection up "<name>"
```

> Run the `connection up` command **without** `sudo`. On desktop Wi-Fi the
> password is often stored in your user's wallet (GNOME Keyring or KWallet),
> which root cannot read, so `sudo nmcli connection up` fails with *"Secrets
> were required, but not provided"*. Running it as your own user lets the
> desktop agent supply the password; alternatives are `--ask` or toggling the
> connection off and on in the network applet. The `modify` command is saved
> either way, so any reconnect applies it.

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
nmcli connection up "<name>"
```

**Option C, GNOME Settings GUI:**

1. Open **Settings**, go to **Network** (or **Wi-Fi**), and click the gear icon
   next to your connection. On the **IPv4** tab, leave **Method** on
   *Automatic (DHCP)* but turn the **DNS** switch from *Automatic* to **Off**,
   then enter the manual servers.
2. Repeat on the **IPv6** tab.
3. Click **Apply**, then reconnect (toggle the connection off/on).

> **Avoid the common mistake:** the GNOME GUI lets you enter only plain IPs
> (e.g. `9.9.9.9`). It cannot express the `#dns.quad9.net` hostname needed for
> TLS certificate-name validation, and it does not by itself force strict DoT on
> the connection. If you use the GUI, you are relying entirely on the global
> `DNSOverTLS=yes` / `DNS=` from Step 3 for the encryption and certificate name,
> so turning the GUI **DNS** switch **Off** (so DHCP DNS is ignored) matters more
> than the IPs you type. For full per-connection certificate validation, prefer
> **Option A** or **Option B**.

* * *

## Step 5: Verify the configuration

```bash
resolvectl status
```

In the **Global** section you should see something like:

```
         Protocols: LLMNR=resolve -mDNS +DNSOverTLS DNSSEC=yes/unsupported
  resolv.conf mode: stub
Current DNS Server: 9.9.9.9#dns.quad9.net
       DNS Servers: 9.9.9.9#dns.quad9.net 149.112.112.112#dns.quad9.net
                    2620:fe::fe#dns.quad9.net 2620:fe::9#dns.quad9.net
```

The DNSSEC status sits **inside the `Protocols` line** (there is no separate
`DNSSEC setting:` line), and only the part **before** its slash is your
setting: `DNSSEC=yes/...` confirms strict validation is on. The part after the
slash is `systemd-resolved`'s probe of what the current server supports, and
under strict DNS-over-TLS it often reads `unsupported` even while validation
works, so do not chase it; Step 7 is the authoritative test. The exact layout
and the LLMNR token vary between `systemd` versions and distributions. The
parts that matter are `+DNSOverTLS`, `DNSSEC=yes/...`, and that the Quad9
servers (with the `#dns.quad9.net` suffix) are the ones in use. After Step 4
(Option A) your active **Link** lists no DNS servers of its own, so queries use
the Global ones; that is the intended state.

* * *

## Step 6: Confirm traffic is encrypted (port 853, not 53)

These checks use `tcpdump` (install it first if needed, see Prerequisites).
Each `tcpdump` command runs until you press **Ctrl-C**, so you will need **two
terminals**: one capturing, one generating a lookup. The commands capture on all
interfaces (`-i any`); if you prefer to watch a single interface, find it with
`ip -br link` (which lists all interfaces) and replace `-i any` with `-i <iface>`.
A warning that the `any` device does not support promiscuous mode is normal and
harmless.

**Negative check.** No plaintext DNS should leave your machine to an *external*
server on port 53. Loopback traffic to the local stub (`127.0.0.53` / `127.0.0.54`)
is normal, so exclude all loopback:

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

You should see TLS traffic to a Quad9 address on 853. Activity on 853, and nothing on port 53 to an external server, confirms DoT is carrying your lookups.

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
  *"... resolve call failed: DNSSEC validation failed: invalid"* (the exact reason
  token can vary).

**Optional `dig` cross-check** (needs `dig`; see Prerequisites):

```bash
dig sigfail.verteiltesysteme.net    # expect status: SERVFAIL
dig sigok.verteiltesysteme.net      # expect an answer
```

> Why `resolvectl query` is the reliable test rather than the `dig` `ad`
> (Authenticated Data) flag: the `127.0.0.53` stub's `ad`-bit handling is
> unreliable. It can be cleared exactly when a query sets the DO bit to request
> DNSSEC data (e.g. `dig +dnssec`), and this behaviour has shifted between
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
  DNSSEC, or you may be behind a **captive portal** (hotel/airport Wi-Fi) whose
  sign-in page hijacks DNS. Captive portals frequently break strict DoT/DNSSEC, and because `DNSOverTLS=yes` with an empty `FallbackDNS=` is strict, the
  portal's own login page may not even resolve. To get online, temporarily relax
  the settings: set `DNSOverTLS=opportunistic` (or `no`) and
  `DNSSEC=allow-downgrade` in `resolved.conf`, run
  `sudo systemctl restart systemd-resolved`, sign in, then **revert to `yes` /
  `yes`** and restart again.
- **All DNS stops working.** Because `FallbackDNS=` is empty, an unreachable
  Quad9 means no resolution by design. Check connectivity to Quad9 on 853:
  `nc -vz 9.9.9.9 853`.
- **Queries still appear on port 53 to an external server.** A per-link/DHCP DNS
  server is still configured. Recheck Step 4 (`ignore-auto-dns` / GUI DNS switch
  **Off**) and confirm with `resolvectl status` that no Link lists your router's
  DNS address.
- **`resolvectl` shows `DNSOverTLS` is off on a Link.** A VPN or another tool may
  own that link's DNS (see below).

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
nmcli connection up "<name>"
```

If you used the **GNOME GUI** (Option C), open **Settings**, go to **Network**,
and click the gear icon. On the **IPv4** tab, turn the **DNS** switch back to
*Automatic* and clear any servers you typed. Repeat on the **IPv6** tab, then
click **Apply** and reconnect.

* * *

## What you achieved

- DNS lookups go to **Quad9's recommended malware-blocking, DNSSEC-validating
  resolvers** (`9.9.9.9`, `149.112.112.112`, and their IPv6 equivalents).
- Lookups are **encrypted and authenticated** with DNS-over-TLS (strict),
  validated against the `dns.quad9.net` certificate.
- Responses are **DNSSEC-validated**, so forged records are rejected.
- Your **router's DHCP-advertised DNS is suppressed**, so every lookup stays on
  the resolver you chose.
- **EDNS Client Subnet (ECS)** is disabled by Quad9 on this service for better
  privacy.

* * *

## References

- [Quad9: Service Addresses & Features](https://www.quad9.net/service/service-addresses-and-features/)
- [systemd `resolved.conf(5)` manual](https://www.freedesktop.org/software/systemd/man/latest/resolved.conf.html)
- [`nm-settings-nmcli(5)`: NetworkManager connection settings](https://networkmanager.dev/docs/api/latest/nm-settings-nmcli.html)
- [DNSSEC (ICANN)](https://www.icann.org/resources/pages/dnssec-qaa-2014-01-29-en)
