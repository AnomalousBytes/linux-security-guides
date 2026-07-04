# How to Configure Secure DNS with Quad9, DNS-over-TLS, and DNSSEC on Linux (Fedora, RHEL, Ubuntu, CachyOS)

> **Published:** 2026-07-03. **Last verified:** 2026-07-03. Field-tested on Fedora
> Linux 44 (KDE, `systemd` 259); the resolver configuration was verified against
> the systemd `resolved.conf(5)` man page, Quad9's service documentation, and, for
> RHEL, Red Hat's networking documentation. Covers Fedora 44, RHEL 9 and 10,
> Ubuntu 26.04 LTS, and CachyOS, all using `systemd-resolved` with NetworkManager.
> Where a command or default differs by distribution, the form for each is given.

> By default, DNS lookups are sent in **plaintext** over port 53. Anyone on the
> network path (your ISP, a public Wi-Fi operator, or an on-path attacker) can
> read every domain you look up, and can tamper with the answers.
> Pointing your system at **Quad9** with **DNS-over-TLS (DoT)** and **DNSSEC
> validation** encrypts those lookups, lets your machine verify it is really
> talking to Quad9, and rejects forged DNS records.

This guide sets up **Quad9** as your system resolver using `systemd-resolved` and
NetworkManager, with **DNS-over-TLS** and **DNSSEC** turned on. The procedure is
the same across these distributions once `systemd-resolved` is the active
resolver; only the package names, the desktop's network GUI, and one RHEL-specific
enablement step differ.

> **Read this if you run RHEL.** On Fedora, Ubuntu, and CachyOS, `systemd-resolved`
> is the default resolver. On **RHEL it is not**: NetworkManager writes
> `/etc/resolv.conf` directly, and Red Hat classifies `systemd-resolved` as a
> **Technology Preview** that is **unsupported on RHEL 9**, and on **RHEL 10**
> supported except for its local DNSSEC validation, which stays a preview. The DoT setup
> here works on both, and Quad9 validates DNSSEC upstream regardless (so forged
> records are still rejected end to end), but if you run RHEL under a production
> support contract, weigh that before enabling it. [Step 1](#step-1-confirm-or-on-rhel-enable-systemd-resolved)
> has the extra step RHEL needs.

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

```
          application (browser, mail client, ...)
                              │  loopback only, plaintext port 53
                              ▼
      ┌──────────────────────────────────────────────┐
      │  systemd-resolved stub   127.0.0.53:53       │
      │  strict local DNSSEC validation (DNSSEC=yes) │
      └───────────────────────┬──────────────────────┘
                              ║  DNS-over-TLS strict, port 853
                              ║  SNI / cert name dns.quad9.net
      ════════════ your machine ends here ════════════
      ISP / LAN sees only TLS to a Quad9 IP on 853, not
      the domain names; no plaintext port 53 on the wire
                              ▼
      ┌──────────────────────────────────────────────┐
      │  Quad9   9.9.9.9    149.112.112.112          │
      │          2620:fe::fe    2620:fe::9           │
      │  upstream DNSSEC validation                  │
      └──────────────────────────────────────────────┘
```

Plaintext DNS stays on loopback (`127.0.0.53`); on the wire it is DNS-over-TLS to
Quad9 on 853.

* * *

## Distribution cheat-sheet

The procedure is the same everywhere once `systemd-resolved` is running. These are
the only per-distribution differences:

| | Fedora 44 | RHEL 9 / 10 | Ubuntu 26.04 | CachyOS |
| --- | --- | --- | --- | --- |
| `systemd-resolved` default? | yes | **no, enable it (Step 1)** | yes | yes (since late 2024) |
| Verification tools (Step 6-7) | `sudo dnf install -y bind-utils nmap-ncat tcpdump` | `sudo dnf install -y bind-utils nmap-ncat tcpdump` | `sudo apt install -y bind9-dnsutils netcat-openbsd tcpdump` | `sudo pacman -S --needed bind openbsd-netcat tcpdump` |
| Desktop GUI (Step 4, Option C) | KDE | usually headless, use Option A | GNOME | KDE |

`dig` comes from `bind-utils`/`bind9-dnsutils`/`bind`; `nc` from
`nmap-ncat`/`netcat-openbsd`/`openbsd-netcat`. The `resolvectl`-based checks need
no extra packages.

* * *

## Prerequisites

- One of the distributions above, with administrative privileges (`sudo`).
- Basic familiarity with the terminal.
- Desktop editions of Fedora, Ubuntu, and CachyOS use NetworkManager by default.
  RHEL uses NetworkManager too; the RHEL steps below assume it.

> **CachyOS-Welcome DoH:** CachyOS-Welcome can set up DNS-over-HTTPS through
> `blocky`, a separate mechanism from the `systemd-resolved` DNS-over-TLS used
> here. Run one or the other, not both. If you enabled the Welcome DoH option,
> disable it before following this guide so the two resolvers do not compete.

* * *

## Step 1: Confirm (or, on RHEL, enable) systemd-resolved

**On Fedora, Ubuntu, and CachyOS**, `systemd-resolved` is already the resolver.
Confirm it:

```bash
resolvectl status
```

- `resolv.conf mode: stub` means `systemd-resolved` is handling DNS. (The
  `Current DNS Server` field shows your *upstream* resolver, your router now or
  `9.9.9.9` once you finish, not the local `127.0.0.53` stub listener.)
- Confirm `/etc/resolv.conf` points at the stub:

```bash
ls -l /etc/resolv.conf
```

You should see a symlink such as
`/etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf` (an absolute target,
`/run/systemd/resolve/stub-resolv.conf`, is equivalent). If it is not, restore it:

```bash
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

On **CachyOS**, if `systemctl is-active systemd-resolved` prints `inactive` (some
installs older than late 2024), enable it and restart NetworkManager:

```bash
sudo systemctl enable --now systemd-resolved
sudo systemctl restart NetworkManager
```

**On RHEL 9 and 10**, `systemd-resolved` is not the default and must be enabled
first. (Note the support caveat in the RHEL heads-up above.) Install it if needed,
tell NetworkManager to use it, point `/etc/resolv.conf` at the stub, and start it:

```bash
sudo dnf install -y systemd-resolved      # separate package on RHEL 9+; skip if present
sudo tee /etc/NetworkManager/conf.d/10-resolved.conf >/dev/null <<'EOF'
[main]
dns=systemd-resolved
EOF
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
sudo systemctl enable --now systemd-resolved
sudo systemctl restart NetworkManager
```

Confirm the result with `resolvectl status` (it should now report
`resolv.conf mode: stub`), then continue as for the other distributions.

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
> `[Resolve]` header with commented keys (`#DNS=`, `#DNSSEC=`, and so on). Edit the
> keys **under that existing header** (do not add a second `[Resolve]` section),
> and make sure no other uncommented `DNS=` / `DNSOverTLS=` / `DNSSEC=` lines
> remain.

Why each line is written this way:

- **`DNS=`** lists all four Quad9 resolvers that provide **malware blocking and
  DNSSEC validation** (the address set Quad9 recommends as its default), both IPv4
  (`9.9.9.9`, `149.112.112.112`) and IPv6 (`2620:fe::fe`, `2620:fe::9`), so the
  setup works on dual-stack networks. **List them under `DNS=`, not
  `FallbackDNS=`.** `FallbackDNS=` is only consulted when *no other DNS server is
  known*, so servers placed there while `DNS=` is set would essentially never be
  used.
- **`#dns.quad9.net`** is the critical part for DoT. The `address#server_name` form
  tells `systemd-resolved` to validate the server's **TLS certificate against that
  hostname** and to send it as the **SNI**. Without the suffix the certificate is
  only checked against the server's IP address and SNI is disabled, a weaker
  authentication guarantee. `dns.quad9.net` is the official TLS authentication name
  for this Quad9 service.
- **`FallbackDNS=`** is intentionally **empty**, though what it overrides depends on
  the build. Fedora and Ubuntu build `systemd` with an **empty** compiled-in
  fallback list, so a stock system has nothing to fall back to. Arch-based builds
  (CachyOS) compile in public fallback resolvers (Quad9, Cloudflare, Google) that
  carry DoT hostnames and follow your `DNSOverTLS=` setting, so they would not be
  plaintext. Emptying the line is worth it either way: it guarantees a resolver you
  did not choose can never be consulted, even if your `DNS=` line fails to load.
  (This is not a runtime failover: if Quad9 is unreachable, lookups simply fail
  rather than falling back, which is the correct posture for a hardening guide.)
- **`DNSOverTLS=yes`** is *strict* mode: lookups are encrypted or they fail. The
  alternative, `opportunistic`, silently falls back to plaintext and is vulnerable
  to a trivial downgrade attack, so it is **not** used here.
- **`DNSSEC=yes`** enables strict local DNSSEC validation: forged or badly-signed
  records are rejected. `systemd-resolved` does not do strict validation by default
  (the shipped default is off, or the weak `allow-downgrade` mode that an on-path
  attacker can silently switch off), so `yes` is a deliberate hardening choice. On
  **RHEL**, this local validation is the part Red Hat still classifies as a
  Technology Preview; it functions, and Quad9 validates DNSSEC upstream regardless,
  so forged records are still rejected end to end. Strict validation can
  occasionally break resolution for a minority of real sites with imperfect DNSSEC
  (and for captive portals); see [Troubleshooting](#troubleshooting) if that
  happens.

Save the file, then apply it:

```bash
sudo systemctl restart systemd-resolved
```

* * *

## Step 4: Stop your router's DHCP DNS from being used too

This step is easy to miss, and it is what keeps every lookup on the resolver you
chose.

`systemd-resolved` queries the global `DNS=` servers **in parallel with the
per-link DNS servers of your default-route connection**, which a normal
Wi-Fi/Ethernet connection picked up from DHCP is. NetworkManager pushes those
per-link (router) servers into `systemd-resolved` over its D-Bus API, so if you
skip this step some lookups can still be sent **to your router** instead of Quad9.
The global `DNSOverTLS=yes` from Step 3 does still apply to per-link servers, so
those lookups are not plaintext, but they target a server you did not choose, with
no hostname for certificate validation, and they bypass Quad9's filtering.

Tell NetworkManager to ignore the DNS servers advertised by DHCP so that only your
Quad9-over-DoT configuration is used. **Use one of the methods below.**

**Option A, per connection with `nmcli` (recommended, deterministic, works
everywhere including headless RHEL):**

List your connections, then modify the one you use (replace `"<name>"`):

```bash
nmcli connection show
sudo nmcli connection modify "<name>" \
  ipv4.ignore-auto-dns yes \
  ipv6.ignore-auto-dns yes
nmcli connection up "<name>"
```

> Run the `connection up` command **without** `sudo`. On desktop Wi-Fi the password
> is often stored in your user's wallet (GNOME Keyring or KWallet), which root
> cannot read, so `sudo nmcli connection up` fails with *"Secrets were required,
> but not provided"*. Running it as your own user lets the desktop agent supply the
> password; alternatives are `--ask` or toggling the connection off and on in the
> network applet. The `modify` command is saved either way, so any reconnect
> applies it.

With the global `DNS=` set in Step 3, this is enough: the link now provides no DNS
servers of its own, so all queries fall through to your Quad9 DoT configuration.

**Option B, pin Quad9 directly on the connection (also forces per-connection DoT):**

If you would rather attach Quad9 to the connection itself (useful on machines that
do not set a global `DNS=`), include the `#dns.quad9.net` hostname so certificate
validation still works, and force DoT on that connection:

```bash
sudo nmcli connection modify "<name>" \
  ipv4.dns "9.9.9.9#dns.quad9.net,149.112.112.112#dns.quad9.net" \
  ipv6.dns "2620:fe::fe#dns.quad9.net,2620:fe::9#dns.quad9.net" \
  ipv4.ignore-auto-dns yes ipv6.ignore-auto-dns yes \
  connection.dns-over-tls yes
nmcli connection up "<name>"
```

**Option C, desktop network GUI:**

- **GNOME (Ubuntu):** Open **Settings > Network** (or **Wi-Fi**), click the gear
  next to your connection, and on the **IPv4** tab leave **Method** on *Automatic
  (DHCP)* but turn the **DNS** switch from *Automatic* to **Off**. Repeat on
  **IPv6**, click **Apply**, and reconnect.
- **KDE (Fedora, CachyOS):** In the network applet, open **Configure Network
  Connections**, select your connection, and on the **IPv4** tab set the **Method**
  to *Automatic (Only addresses)* so DHCP provides the address but not DNS. Repeat
  on **IPv6**, apply, and reconnect.

> **Avoid the common mistake:** the desktop GUIs let you enter only plain IPs (for
> example `9.9.9.9`). They cannot express the `#dns.quad9.net` hostname needed for
> TLS certificate-name validation, and they do not by themselves force strict DoT
> on the connection. If you use the GUI, you are relying entirely on the global
> `DNSOverTLS=yes` / `DNS=` from Step 3 for the encryption and certificate name, so
> turning DHCP-provided DNS **off** matters more than the IPs you type. For full
> per-connection certificate validation, prefer **Option A** or **Option B**.

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
`DNSSEC setting:` line), and only the part **before** its slash is your setting:
`DNSSEC=yes/...` confirms strict validation is on. The part after the slash is
`systemd-resolved`'s probe of what the current server supports, and under strict
DNS-over-TLS it often reads `unsupported` even while validation works, so do not
chase it; Step 7 is the authoritative test. The exact layout and the LLMNR token
vary between `systemd` versions and distributions. The parts that matter are
`+DNSOverTLS`, `DNSSEC=yes/...`, and that the Quad9 servers (with the
`#dns.quad9.net` suffix) are the ones in use. After Step 4 (Option A) your active
**Link** lists no DNS servers of its own, so queries use the Global ones; that is
the intended state.

* * *

## Step 6: Confirm traffic is encrypted (port 853, not 53)

These checks use `tcpdump` (install it first if needed, see the cheat-sheet). Each
`tcpdump` command runs until you press **Ctrl-C**, so you will need **two
terminals**: one capturing, one generating a lookup. The commands capture on all
interfaces (`-i any`); if you prefer to watch a single interface, find it with
`ip -br link` and replace `-i any` with `-i <iface>`. A warning that the `any`
device does not support promiscuous mode is normal and harmless.

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

You should see TLS traffic to a Quad9 address on 853. Activity on 853, and nothing
on port 53 to an external server, confirms DoT is carrying your lookups.

* * *

## Step 7: Confirm DNSSEC validation

Use the well-known DNSSEC test pair: `sigok` is correctly signed and must resolve;
`sigfail` is deliberately broken and must be rejected. The `verteiltesysteme.net`
names used below are long-standing aliases for the same test, now also served under
the current canonical names `sigok.ippacket.stream` and `sigfail.ippacket.stream`;
either form works.

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

**Optional `dig` cross-check** (needs `dig`; see the cheat-sheet):

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
failed `sigfail` lookup confirms DNSSEC is enforced **end-to-end** (Quad9 plus your
local resolver). This is also why the setup still rejects forged records on RHEL,
where the *local* validation is a Technology Preview: Quad9's upstream validation
does not depend on it.

* * *

## Limitations: what this does (and does not) hide

Encrypted DNS is an important layer, but it is **not** full browsing privacy. Think
carefully about the threat model:

- **Destination IPs are still visible.** After the lookup, your device connects to
  the site's IP address, which your ISP can see and often map back to a site.
- **TLS SNI usually leaks the hostname.** Most HTTPS handshakes still send the site
  name in cleartext in the TLS `ClientHello` unless **Encrypted Client Hello (ECH)**
  is in use end-to-end.
- **Quad9 still sees your queries.** You are moving trust from your ISP to Quad9 (a
  non-profit that states it does not log personal data and disables ECS), not
  eliminating it.
- **This is not anonymity.** For that you need a VPN or Tor, which change *where*
  your traffic appears to originate. See the
  [WireGuard VPN guide](wireguard-vpn-self-hosted.md).

In short: DoT + DNSSEC removes one significant metadata leak and prevents
tampering, and pairs well with ECH and (if you need it) a VPN.

* * *

## Troubleshooting

- **A specific site suddenly fails to resolve.** Its zone may have broken DNSSEC,
  or you may be behind a **captive portal** (hotel or airport Wi-Fi) whose sign-in
  page hijacks DNS. Captive portals frequently break strict DoT/DNSSEC, and because
  `DNSOverTLS=yes` with an empty `FallbackDNS=` is strict, the portal's own login
  page may not even resolve. To get online, temporarily relax the settings: set
  `DNSOverTLS=opportunistic` (or `no`) and `DNSSEC=allow-downgrade` in
  `resolved.conf`, run `sudo systemctl restart systemd-resolved`, sign in, then
  **revert to `yes` / `yes`** and restart again.
- **All DNS stops working.** Because `FallbackDNS=` is empty, an unreachable Quad9
  means no resolution by design. Check connectivity to Quad9 on 853:
  `nc -vz 9.9.9.9 853`.
- **(RHEL) DoT fails to connect even though Quad9 is reachable.** RHEL's system-wide
  cryptographic policy governs the TLS handshake. The `DEFAULT` policy works with
  Quad9; a hardened custom policy or an unusual `FUTURE` setting could restrict the
  ciphers or protocol versions DoT needs. Check with `update-crypto-policies --show`.
- **(CachyOS) DNS works only after restarting `systemd-resolved`.** On some setups
  resolution can fail at boot until the service settles. Confirm it is enabled
  (`systemctl is-enabled systemd-resolved`) and check service ordering with
  `systemctl status systemd-resolved NetworkManager`.
- **Queries still appear on port 53 to an external server.** A per-link/DHCP DNS
  server is still configured. Recheck Step 4 (`ignore-auto-dns`, or the GUI DNS
  switch **Off**) and confirm with `resolvectl status` that no Link lists your
  router's DNS address.
- **`resolvectl` shows `DNSOverTLS` is off on a Link.** A VPN or another tool
  (including CachyOS-Welcome's `blocky` DoH option) may own that link's DNS. Pick
  one resolver and disable the others.

* * *

## Optional: VPNs, Tailscale, and Docker

- **Tailscale** installs its own resolver (`100.100.100.100`) and, with its
  global-nameserver override enabled (the admin console's *"Override DNS servers"*
  option), captures **all** queries ahead of your Quad9 config while the VPN is up.
  If you want Quad9-over-DoT to remain in effect, either disable that override or set
  Quad9 as the global nameserver in the Tailscale admin DNS settings.
- **Other VPNs** typically push their own DNS for the duration of the connection;
  that is usually desirable (it keeps lookups inside the tunnel). The
  [WireGuard guide](wireguard-vpn-self-hosted.md) sets an in-tunnel resolver for
  exactly this reason.
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

If you used a **desktop GUI** (Option C), reopen the connection editor, set the
IPv4 and IPv6 DNS back to *Automatic*, clear any servers you typed, then apply and
reconnect. On **RHEL**, to fully undo Step 1: remove
`/etc/NetworkManager/conf.d/10-resolved.conf`, delete the stub symlink with
`sudo rm /etc/resolv.conf` (NetworkManager will not rewrite a symlinked
`resolv.conf`), `sudo systemctl disable --now systemd-resolved`, then
`sudo systemctl restart NetworkManager` to regenerate a managed `/etc/resolv.conf`.

* * *

## What you achieved

- DNS lookups go to **Quad9's recommended malware-blocking, DNSSEC-validating
  resolvers** (`9.9.9.9`, `149.112.112.112`, and their IPv6 equivalents).
- Lookups are **encrypted and authenticated** with DNS-over-TLS (strict), validated
  against the `dns.quad9.net` certificate.
- Responses are **DNSSEC-validated** (locally where supported, and always upstream
  by Quad9), so forged records are rejected.
- Your **router's DHCP-advertised DNS is suppressed**, so every lookup stays on the
  resolver you chose.
- **EDNS Client Subnet (ECS)** is disabled by Quad9 on this service for better
  privacy.

* * *

## References

- [Quad9: Service Addresses & Features](https://www.quad9.net/service/service-addresses-and-features/)
- [systemd `resolved.conf(5)` manual](https://www.freedesktop.org/software/systemd/man/latest/resolved.conf.html)
- [`nm-settings-nmcli(5)`: NetworkManager connection settings](https://networkmanager.dev/docs/api/latest/nm-settings-nmcli.html)
- [Red Hat: Configuring the order of DNS servers (RHEL networking)](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/10/html/configuring_and_managing_networking/configuring-the-order-of-dns-servers)
- [ArchWiki: systemd-resolved](https://wiki.archlinux.org/title/Systemd-resolved)
- [DNSSEC (ICANN)](https://www.icann.org/resources/pages/dnssec-qaa-2014-01-29-en)
