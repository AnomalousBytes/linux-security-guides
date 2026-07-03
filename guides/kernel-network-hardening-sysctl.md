# How to Harden Linux Network Kernel Parameters with sysctl (CIS- and NIST-aligned)

> **Published:** 2026-07-03. **Last verified:** 2026-07-03. The current values and
> the applying mechanism were checked on a live Fedora Linux 44 system (kernel
> 7.0.13). Key semantics were verified against the
> [kernel networking sysctl documentation](https://docs.kernel.org/networking/ip-sysctl.html)
> and the `sysctl.d(5)` man page; the CIS mappings against the CIS Red Hat
> Enterprise Linux 9 v2.0.0, RHEL 10 v1.0.1, and Ubuntu 24.04 LTS v1.0.0
> Benchmarks. Covers Fedora 44, RHEL 9 and 10, Ubuntu 26.04 LTS, and CachyOS.

> The Linux network stack has dozens of tunable switches (`sysctl` keys) that
> decide how the kernel reacts to things like spoofed source addresses, ICMP
> redirects that try to reroute your traffic, source-routed packets, and SYN
> floods. Most of the hardening values are safe to set on any machine. A few
> interact with routing, containers, and VPNs, and setting them blindly breaks
> real things. This guide sets the CIS-recommended values, explains what each one
> stops, and flags the ones to think about first.

* * *

## Read this first: most of these are already set

On a modern distribution, several of these parameters already hold their hardened
value out of the box, so this guide is about **verifying what is set and closing
the specific gaps**, not fixing a wide-open system. On the Fedora 44 test machine,
`accept_redirects`, `accept_source_route` (IPv4 and IPv6), `icmp_echo_ignore_broadcasts`,
`icmp_ignore_bogus_error_responses`, and `tcp_syncookies` were already hardened;
`log_martians` and `secure_redirects` were not, and reverse-path filtering was in
loose mode. Your distribution and version will differ, which is why Step 1 reads
the live values before changing anything.

Two honest warnings before you start:

- **`ip_forward` is not always a mistake.** Routers, VPN gateways (including the
  [WireGuard guide](wireguard-vpn-self-hosted.md)), and container or VM hosts
  (Docker and libvirt turn it on themselves) legitimately need IP forwarding. On
  the test box it was `1` precisely because it runs Docker, libvirt, and Tailscale.
  Do not force it to `0` on such a host.
- **Strict reverse-path filtering and disabling IPv6 router advertisements can
  break connectivity** on multihomed hosts and on networks that rely on IPv6
  autoconfiguration. Both are called out below.

These are host-level packet-handling defaults. They sit alongside the
[host firewall](host-firewall-hardening.md) (which controls *ports*), and neither
replaces the other.

* * *

## How sysctl settings are applied

A `sysctl` key is a kernel setting under `/proc/sys/`. You can change one for the
running system, and you can set it persistently so it survives a reboot.

- **Read one:** `sysctl net.ipv4.tcp_syncookies`
- **Set one for now (not persistent):** `sudo sysctl -w net.ipv4.tcp_syncookies=1`
- **Set persistently:** put `key = value` lines in a file under `/etc/sysctl.d/`
  ending in `.conf`, then apply with `sudo sysctl --system`.

Two rules from `sysctl.d(5)` matter:

- **Precedence and ordering.** Files load from several directories
  (`/etc/sysctl.d/`, `/run/sysctl.d/`, `/usr/lib/sysctl.d/`), all sorted together
  by filename. For the same key, the file whose name sorts **last** wins, and
  `/etc/` beats `/usr/lib/` for a same-named file. The convention is a two-digit
  numeric prefix. A file named `90-network-hardening.conf` therefore overrides the
  distribution's `50-*` defaults, which is what you want.
- **`default` vs `all` vs the interface.** Many keys exist per interface, plus two
  special names: `all` and `default`. `default` is only a template copied to
  **new** interfaces as they appear; changing it does not touch interfaces that
  already exist. `all` combines with the per-interface value, and the combination
  rule depends on the key (see the note after the config block). The practical
  takeaway: set **both** `.all` and `.default` for each key, which is what CIS does
  and what the block below does.

* * *

## Step 1: Read the current values

Before changing anything, see what your system already enforces:

```bash
for k in net.ipv4.ip_forward \
         net.ipv4.conf.all.rp_filter net.ipv4.conf.all.accept_redirects \
         net.ipv4.conf.all.secure_redirects net.ipv4.conf.all.send_redirects \
         net.ipv4.conf.all.accept_source_route net.ipv4.conf.all.log_martians \
         net.ipv4.icmp_echo_ignore_broadcasts net.ipv4.icmp_ignore_bogus_error_responses \
         net.ipv4.tcp_syncookies \
         net.ipv6.conf.all.forwarding net.ipv6.conf.all.accept_redirects \
         net.ipv6.conf.all.accept_ra net.ipv6.conf.all.accept_source_route; do
  printf '%-46s = %s\n' "$k" "$(sysctl -n "$k" 2>/dev/null)"
done
```

Note which are already at their hardened value (`0` for the redirect, forwarding,
and source-route keys; `1` for the ignore, syncookies, and log keys). You only
need the drop-in below to fill the gaps, though writing the whole set is harmless
and documents your intent.

* * *

## Step 2: Write the hardening drop-in

Create `/etc/sysctl.d/90-network-hardening.conf`. This is the set that is safe on
essentially any host:

```ini
# Ignore ICMP redirects (stop an attacker rerouting your traffic)
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# Do not accept "secure" ICMP redirects either
net.ipv4.conf.all.secure_redirects = 0
net.ipv4.conf.default.secure_redirects = 0

# Do not send ICMP redirects (only a router should)
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Reject source-routed packets (they bypass normal routing)
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0
net.ipv6.conf.all.accept_source_route = 0
net.ipv6.conf.default.accept_source_route = 0

# Log spoofed, source-routed, and impossible ("martian") packets
net.ipv4.conf.all.log_martians = 1
net.ipv4.conf.default.log_martians = 1

# Ignore ICMP echo to broadcast addresses (Smurf amplification)
net.ipv4.icmp_echo_ignore_broadcasts = 1

# Ignore bogus ICMP error responses
net.ipv4.icmp_ignore_bogus_error_responses = 1

# SYN cookies: survive a SYN-flood without dropping real connections
net.ipv4.tcp_syncookies = 1

# Reverse-path filtering: drop packets with spoofed source addresses.
# 1 = strict. Use 2 (loose) instead on multihomed hosts or where routing
# is asymmetric (multiple uplinks, some VPNs), or replies get dropped.
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
```

**How `all` and `default` combine** (from the kernel documentation), so you know
why both are set:

- `rp_filter` uses the **maximum** of the `all` and per-interface values.
- `log_martians`, `send_redirects`, and `secure_redirects` are enabled if **either**
  the `all` or the interface value is set.
- `accept_redirects` is the special case: with forwarding off it is enabled if
  either value is set; with forwarding on it needs both. Setting `all` and
  `default` to `0` covers every case.

### The conditional settings (read the warning)

These three are genuine CIS recommendations, but each one breaks a common, valid
configuration. Add them to the same file **only** if the caveat does not apply to
your host:

```ini
# Disable IP forwarding -- ONLY on a host that is NOT a router, VPN gateway,
# or container/VM host. Docker, libvirt, and WireGuard need forwarding ON.
net.ipv4.ip_forward = 0
net.ipv6.conf.all.forwarding = 0

# Do not accept IPv6 Router Advertisements -- ONLY where IPv6 is static or
# from DHCPv6. On a normal client that gets IPv6 by autoconfiguration (SLAAC),
# this removes its IPv6 connectivity.
net.ipv6.conf.all.accept_ra = 0
net.ipv6.conf.default.accept_ra = 0
```

If you are unsure whether a host forwards packets, check `sysctl net.ipv4.ip_forward`
first: if it is already `1` and the machine runs containers, VMs, or a VPN, leave
it alone.

* * *

## Step 3: Apply and verify

```bash
sudo sysctl --system
```

This reloads every `sysctl.d` file in order and prints what it sets, so you can see
your `90-network-hardening.conf` values applied last. Re-run the Step 1 loop to
confirm the live values now match your intent.

* * *

## Distribution defaults

The kernel's own defaults are mostly *un*hardened (redirects accepted,
`log_martians` off), but distributions layer their own `sysctl.d` files on top, so
the starting point differs:

- **Fedora 44** (verified live) already hardens the redirect-accept, source-route,
  broadcast-ICMP, bogus-ICMP, and SYN-cookie keys. Its reverse-path filtering comes
  from systemd's `50-default.conf` in **loose** mode (`default = 2`), and it does
  **not** ship the RHEL strict override. `log_martians` and `secure_redirects` are
  the notable gaps.
- **RHEL** ships `50-redhat.conf`, which sets reverse-path filtering to **strict**
  (`1`) (confirmed on RHEL 9; RHEL 10 follows the same packaging convention).
  Otherwise similar to Fedora.
- **Ubuntu 26.04** enables SYN cookies by default and ships its own network
  defaults; confirm the rest with the Step 1 loop.
- **CachyOS** (Arch-based) inherits systemd's `50-default.conf` (loose `rp_filter`)
  and does not add network-hardening defaults of its own, so it benefits most from
  the full drop-in.

Because defaults move between versions, treat Step 1 as the source of truth for
your specific machine rather than assuming any of the above.

* * *

## CIS and NIST alignment

These settings are the **"Configure Network Kernel Parameters"** section of the CIS
Linux Benchmarks. As with the other guides here, CIS numbering is version-specific,
so the numbers are given per benchmark:

- **CIS RHEL 9 v2.0.0** (2024-06-24) and **CIS Ubuntu 24.04 LTS v1.0.0**
  (2024-08-26) both use section **3.3**, with eleven recommendations (3.3.1 to
  3.3.11). The table below uses that numbering.
- **CIS RHEL 10 v1.0.1** restructured section 3.3 into one recommendation per
  individual key (3.3.1.x for IPv4, 3.3.2.x for IPv6), so a single row below maps
  to several RHEL 10 items.
- **CIS Ubuntu 24.04 LTS v2.0.0** (2026) renumbers this section; a faithful copy
  was not available at verification, so confirm against your version.
- **Fedora** and **CachyOS/Arch** have no CIS benchmark; adapt the RHEL benchmark.

| Setting | CIS §3.3 (RHEL 9 v2.0.0 / Ubuntu 24.04 v1.0.0) | NIST SP 800-53 Rev 5 |
| --- | --- | --- |
| `ip_forward = 0` (conditional) | 3.3.1 | CM-6, CM-7, SC-7 |
| `send_redirects = 0` | 3.3.2 | CM-6, SC-7 |
| `icmp_ignore_bogus_error_responses = 1` | 3.3.3 | CM-6 |
| `icmp_echo_ignore_broadcasts = 1` | 3.3.4 | CM-6, SC-5 |
| `accept_redirects = 0` (IPv4 and IPv6) | 3.3.5 | CM-6, SC-7 |
| `secure_redirects = 0` | 3.3.6 | CM-6, SC-7 |
| `rp_filter = 1` (or 2) | 3.3.7 | CM-6, SC-7 |
| `accept_source_route = 0` (IPv4 and IPv6) | 3.3.8 | CM-6, SC-7 |
| `log_martians = 1` | 3.3.9 | CM-6, AU-12, SI-4 |
| `tcp_syncookies = 1` | 3.3.10 | CM-6, SC-5 |
| `accept_ra = 0` (conditional, IPv6) | 3.3.11 | CM-6, SC-7 |

The blanket NIST control is **CM-6 (Configuration Settings)**: every line is a
documented, most-restrictive kernel setting. **CM-7 (Least Functionality)** covers
turning off functions a hardened host does not need (forwarding, source routing).
**SC-7 (Boundary Protection)** covers the routing-integrity items (redirects,
source routing, reverse-path filtering, router advertisements). **SC-5
(Denial-of-service Protection)** covers SYN cookies and broadcast-ICMP. Note that
`log_martians` is a logging behavior, so it maps to **AU-12 (Audit Record
Generation)** and **SI-4 (System Monitoring)** rather than to a packet-blocking
control.

* * *

## Troubleshooting

- **Replies to some traffic vanish after enabling strict `rp_filter`.** The host
  is multihomed or the routing is asymmetric, so the reply would leave by a
  different interface than the request arrived on, and strict mode drops it. Set
  `rp_filter = 2` (loose) instead and re-apply.
- **IPv6 stopped working after `accept_ra = 0`.** The machine was getting its IPv6
  address and route from Router Advertisements (SLAAC). Remove the `accept_ra`
  lines, or use it only on hosts with static or DHCPv6 addressing.
- **Containers or the VPN lost networking after `ip_forward = 0`.** Forwarding is
  required there. Remove the `ip_forward` and IPv6 `forwarding` lines; Docker and
  libvirt set forwarding on for a reason.
- **A value did not change.** Another file sorts after yours or a service resets it
  at runtime. Check with `sysctl --system` output (it prints each file) and confirm
  your filename sorts last; Docker and some VPN tools re-assert `ip_forward` when
  they start.

* * *

## How to roll back

```bash
sudo rm /etc/sysctl.d/90-network-hardening.conf
sudo sysctl --system
```

Removing the file and reloading restores the values that other files set, but a key
that *only* your file set keeps its current runtime value until the next reboot (a
reload does not "unset" a key). Reboot, or set the specific keys back by hand with
`sysctl -w`, if you need the old values immediately.

* * *

## What you achieved

- The kernel now **ignores ICMP redirects and source-routed packets**, so an
  attacker on your network cannot reroute your traffic through them.
- **Spoofed-source packets are dropped** (reverse-path filtering) and **logged**
  (`log_martians`).
- **SYN-flood and broadcast-amplification** resistance is on.
- The settings that can break routers, VPNs, containers, and IPv6
  autoconfiguration were called out, not applied blindly.
- The configuration lives in one documented drop-in that maps to the CIS network
  kernel-parameter controls and to NIST CM-6, SC-7, and SC-5.

* * *

## References

- [Kernel IP sysctl documentation](https://docs.kernel.org/networking/ip-sysctl.html)
- [`sysctl.d(5)` manual](https://www.freedesktop.org/software/systemd/man/latest/sysctl.d.html)
- [`systemd-sysctl.service(8)` manual](https://www.freedesktop.org/software/systemd/man/latest/systemd-sysctl.service.html)
- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks)
- [NIST SP 800-53 Rev 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)
