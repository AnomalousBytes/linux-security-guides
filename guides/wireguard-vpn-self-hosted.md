# How to Set Up a Self-Hosted WireGuard VPN on Linux (Fedora, RHEL, Ubuntu, CachyOS)

> **Published:** 2026-07-03. **Last verified:** 2026-07-03 against the `wg(8)` and
> `wg-quick(8)` man pages, [wireguard.com](https://www.wireguard.com/), and the
> official Ubuntu Server documentation. The WireGuard kernel module was confirmed
> present on a live Fedora Linux 44 system (kernel 7.0.13; the module has shipped
> in-tree since Linux 5.6, so no DKMS is needed). Covers Fedora 44, RHEL 9 and 10,
> Ubuntu 26.04 LTS, and CachyOS. A live two-host tunnel bring-up is not part of
> this verification; the configuration is verified against the references above.
> Where a command differs by distribution, the form for each is given.

> WireGuard wraps your traffic in an encrypted UDP tunnel between two machines you
> control. People use it two ways: to reach their own devices and home network
> securely from anywhere, and to route a laptop's entire connection through a
> server they run so the local network and ISP cannot see it. This guide covers
> both. Be clear-eyed about the second use: a VPN you host yourself changes *who*
> can see your traffic. It does not make you anonymous.

* * *

## What this does, and what it does not do

A self-hosted WireGuard VPN is a strong, simple tool with honest limits. Know them
before you rely on it:

- **It encrypts and authenticates traffic in transit.** Between your device and
  your server, nobody on the path (the coffee-shop Wi-Fi, the ISP) can read or
  alter your traffic. They see only encrypted UDP going to your server's address.
- **It moves trust to your server's host, it does not remove trust.** For a
  full-tunnel setup, the tunnel ends at your server (often a rented VPS), which
  decrypts your traffic and forwards it to the internet. That host, and its
  provider, can see your decrypted traffic and the real sites you reach. You have
  moved trust from the local network and your ISP to your VPS provider.
- **It changes your apparent source IP, it is not anonymity.** Your traffic
  appears to come from the server. But this is a single hop with no mixing, so
  anyone who can see both ends (the provider, or a well-resourced observer) can
  link you to it. For anonymity you need Tor or a multi-hop design, which are a
  different tool.
- **It does not protect the endpoints.** WireGuard secures data on the wire. A
  compromised laptop or server, and the applications on them, are outside its
  scope. Keep host hardening in place (see the SSH and firewall guides here).
- **DNS must go through the tunnel or it leaks.** If your device keeps using the
  local network's resolver, your lookups still reveal every site you visit to that
  network. Step 6 sets an in-tunnel resolver. Pair it with the
  [Secure DNS guide](secure-dns-quad9-dot-dnssec-fedora-44.md).

In short, a self-hosted WireGuard VPN is excellent for securing access to your own
systems and for taking your traffic off an untrusted local network. It is not a
privacy silver bullet.

* * *

## Two ways to use this guide

- **Remote access.** Reach a home server, a lab, or a private subnet from
  anywhere, without exposing those services to the public internet. The tunnel
  carries only traffic bound for that subnet (a "split tunnel").
- **Full-tunnel VPN.** Send *all* of a device's traffic through your server. Use
  this to protect a laptop on public Wi-Fi, or to present a stable exit IP. It
  needs routing and NAT on the server (Step 4).

The setup is the same up to Step 3. The difference is the `AllowedIPs` value on
the client (Step 6) and whether the server does NAT (Step 4).

* * *

## Distribution cheat-sheet

The WireGuard kernel module ships with the Linux kernel (since 5.6), so on current
systems you install only the `wireguard-tools` userspace package. The tools give
you `wg` and `wg-quick`.

| | Install command | Notes |
| --- | --- | --- |
| Fedora 44 | `sudo dnf install wireguard-tools` | Module in-tree. |
| RHEL 9 / 10 | `sudo dnf install wireguard-tools` | In **AppStream**, no EPEL needed. Red Hat ships WireGuard as a **Technology Preview** (not covered by production support). |
| Ubuntu 26.04 | `sudo apt install wireguard` | The `wireguard` package pulls in `wireguard-tools`. |
| CachyOS | `sudo pacman -S wireguard-tools` | From the Arch `extra` repository. |

* * *

## Prerequisites

- **A server with a reachable address.** A VPS with a public IP is the common
  case. A home server works too if you can forward a UDP port to it. This is the
  machine peers connect *to*.
- **Root or `sudo`** on both the server and each client.
- **A client device** running Linux (this guide's client steps use `wg-quick`;
  WireGuard also has apps for phones, Windows, and macOS that read the same config).
- Basic comfort editing files and using the terminal.

Throughout, replace placeholders: `server.example.com` is your server's public
address, `eth0` is the server's internet-facing interface (find it with
`ip route get 1.1.1.1`), and the `10.x.x.x` tunnel addresses are examples you can
keep or change.

* * *

## Step 1: Install WireGuard

Install `wireguard-tools` on the server and on each client, using the command from
the cheat-sheet. Confirm it is there and that the kernel module is available:

```bash
wg --version
sudo modprobe wireguard && echo "module OK"
```

On current kernels the module loads on demand when you bring up a tunnel, so a
clean `modprobe` with no error is all you need.

* * *

## Step 2: Generate keys

Each machine (the server and every client) gets its own key pair. The private key
never leaves the machine that owns it; you share only the public key. Generate
them with a tight umask so the private key is not world-readable:

```bash
umask 077
wg genkey | tee privatekey | wg pubkey > publickey
```

`privatekey` now holds this machine's private key, `publickey` its public key. Do
this once per machine. For an optional extra symmetric layer shared between a
server and one client, also generate a preshared key **once** and put the same
value on both ends:

```bash
wg genpsk > presharedkey
```

Keep private keys secret and never reuse one key pair across devices. Treat the
private key like an SSH private key (NIST SC-12, key management).

* * *

## Step 3: Configure the server

Create `/etc/wireguard/wg0.conf` on the server. The `[Interface]` block is the
server itself; each `[Peer]` block is one client. On the server, a peer's
`AllowedIPs` is that client's address *inside* the tunnel, as a `/32`. This is how
WireGuard knows which peer a packet belongs to (its authors call this "cryptokey
routing").

**Remote-access server** (clients reach the `10.10.10.0/24` tunnel and whatever
you route to them):

```ini
[Interface]
Address = 10.10.10.1/24
ListenPort = 51820
PrivateKey = <server-private-key>

[Peer]
# laptop
PublicKey = <client-public-key>
AllowedIPs = 10.10.10.2/32
```

- **`ListenPort = 51820`** is set explicitly. WireGuard has no built-in default
  port (it picks a random one if you omit this); `51820/udp` is the conventional
  choice used in WireGuard's own examples. It is **UDP**.
- **`Address`** is the server's tunnel IP. `Address`, and the client's `DNS` line
  later, are features of `wg-quick`, the tool this guide uses to bring tunnels up.

Add one `[Peer]` block per client, each with a unique `AllowedIPs` address.

Lock down the file (it holds a private key):

```bash
sudo chmod 600 /etc/wireguard/wg0.conf
```

For a **full-tunnel VPN gateway**, use the same file and continue to Step 4, which
adds routing and NAT so client traffic can reach the internet through the server.

* * *

## Step 4: Turn on routing and NAT (full-tunnel only)

Skip this step for a pure remote-access setup that does not route clients to the
internet.

**1. Enable IP forwarding.** A gateway must be allowed to route packets between
interfaces. Add a sysctl drop-in (this is the same mechanism as the
[kernel network hardening guide](kernel-network-hardening-sysctl.md), and it is
the one legitimate reason to set `ip_forward=1`):

```bash
sudo tee /etc/sysctl.d/70-wireguard-routing.conf >/dev/null <<'EOF'
net.ipv4.ip_forward = 1
net.ipv6.conf.all.forwarding = 1
EOF
sudo sysctl --system
```

Add the IPv6 line only if you route IPv6 to clients.

**2. Masquerade the tunnel subnet out of the internet-facing interface**, so
replies find their way back. Do this in your host firewall rather than in the
tunnel config, which keeps it persistent and in one place. Use the tool your
distribution runs (see the [host firewall guide](host-firewall-hardening.md)):

```bash
# Fedora and RHEL (firewalld): masquerade plus open the port
sudo firewall-cmd --permanent --add-masquerade
sudo firewall-cmd --permanent --add-port=51820/udp
sudo firewall-cmd --reload
```

On Ubuntu, CachyOS, or a native nftables host, add a NAT chain (replace `eth0`
with your WAN interface and the subnet with your tunnel's):

```
# /etc/nftables.conf, alongside your filter table
table inet nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        ip saddr 10.10.10.0/24 oifname "eth0" masquerade
    }
}
```

Then `sudo nft -f /etc/nftables.conf`. If you prefer the traditional tool, the
equivalent verified rule is
`iptables -t nat -A POSTROUTING -s 10.10.10.0/24 -o eth0 -j MASQUERADE`.

* * *

## Step 5: Open the firewall port

WireGuard listens on **UDP**, so open its port on the server:

```bash
# Fedora and RHEL (firewalld) -- skip if you already opened it in Step 4
sudo firewall-cmd --add-port=51820/udp --permanent
sudo firewall-cmd --reload

# Ubuntu and CachyOS (ufw)
sudo ufw allow 51820/udp
```

If SSH to this server is your only way in, confirm SSH still works before you rely
on the tunnel. See the [host firewall guide](host-firewall-hardening.md).

* * *

## Step 6: Configure the client

Create `/etc/wireguard/wg0.conf` on the client. Here the `[Peer]` is the *server*,
and `AllowedIPs` means "what to send into the tunnel," which is the setting that
decides split vs full tunnel.

**Full-tunnel client** (all traffic through the server):

```ini
[Interface]
PrivateKey = <client-private-key>
Address = 10.10.10.2/24
DNS = 10.10.10.1

[Peer]
PublicKey = <server-public-key>
Endpoint = server.example.com:51820
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
```

**Split-tunnel client** (only the private subnet through the tunnel, everything
else direct): change `AllowedIPs` to the target networks, for example
`AllowedIPs = 10.10.10.0/24, 192.168.50.0/24`, and usually drop the `DNS` line.

What the notable lines do:

- **`AllowedIPs = 0.0.0.0/0, ::/0`** captures all IPv4 and IPv6 traffic. A narrower
  value gives a split tunnel. This is the single most important client setting.
- **`DNS = 10.10.10.1`** points name resolution at a resolver reachable through the
  tunnel, so lookups do not leak to the local network. `wg-quick` applies it using
  a `resolvconf` command; on systems with `systemd-resolved` that integration is
  usually present, otherwise install `openresolv`. Point it at your own resolver
  and pair with the [Secure DNS guide](secure-dns-quad9-dot-dnssec-fedora-44.md).
- **`PersistentKeepalive = 25`** keeps the tunnel alive through NAT by sending a
  keepalive every 25 seconds. It is WireGuard's recommended value and is needed
  when the client sits behind NAT (most do). Leave it off for a server that only
  receives.
- **`Endpoint`** is only on the client. The server learns each peer's current
  address from its incoming packets, which is what lets a client roam.

Lock the file down: `sudo chmod 600 /etc/wireguard/wg0.conf`.

Finally, tell the server about this client: add a `[Peer]` block to the server's
`wg0.conf` with the client's public key and its `/32` tunnel address, then reload
the server tunnel (Step 7).

* * *

## Step 7: Start it, verify, and make it persistent

Bring the tunnel up on both ends by the interface name (`wg0` maps to
`/etc/wireguard/wg0.conf`):

```bash
sudo wg-quick up wg0
```

Check the tunnel status and that a handshake happened:

```bash
sudo wg show
```

A line reading `latest handshake:` with a recent time means the two ends
authenticated and exchanged keys. If it never appears, see Troubleshooting.

**Verify it is actually carrying your traffic** (full-tunnel):

```bash
curl https://ifconfig.me      # should print your SERVER's public IP, not your local one
resolvectl query example.com  # confirm DNS resolves; pair with the Secure DNS guide's checks
```

If `ifconfig.me` shows the server's IP, your traffic is exiting through the tunnel.

**Make it start at boot** with the shipped systemd unit:

```bash
sudo systemctl enable --now wg-quick@wg0
```

To take the tunnel down: `sudo wg-quick down wg0` (and
`sudo systemctl disable --now wg-quick@wg0` to stop it starting at boot).

* * *

## The cryptography, and a FIPS caveat

WireGuard uses one fixed set of modern primitives, described on its
[protocol page](https://www.wireguard.com/protocol/): **Curve25519** for the
key-exchange (an ECDH handshake built on the Noise protocol framework),
**ChaCha20-Poly1305** for authenticated encryption, and **BLAKE2s** for hashing.
These are not negotiable. WireGuard "intentionally lacks cipher and protocol
agility": there is no handshake where a downgrade could be forced, and if a
primitive were ever broken, the fix is a new version that every endpoint updates
to. That design removes a whole class of downgrade attacks, at the cost of the
flexibility some environments require.

That cost is real in one place: **WireGuard cannot be used where FIPS 140-validated
cryptography is mandated** (many US federal and regulated systems). Two of its
primitives, ChaCha20-Poly1305 and BLAKE2s, are not among the FIPS-approved
algorithms, and its X25519 key agreement is not an approved key-establishment
scheme. (Ed25519 *signatures* were approved in FIPS 186-5, but WireGuard does not
use those; it uses X25519 for key agreement, a different and non-approved use.) If
you need FIPS compliance, use an IPsec or TLS VPN with approved algorithms instead.

* * *

## NIST alignment

There is no CIS Benchmark specific to WireGuard. A self-hosted VPN maps to these
**NIST SP 800-53 Rev 5** controls:

| What it provides | NIST SP 800-53 Rev 5 |
| --- | --- |
| Encrypted, authenticated tunnel (confidentiality and integrity in transit) | SC-8, SC-8(1) |
| The VPN server as a controlled network boundary | SC-7 (Boundary Protection) |
| Protected remote access to a private network | AC-17, AC-17(2) |
| Key pairs, preshared keys, and rotation | SC-12 (Cryptographic Key Establishment and Management) |
| Use of cryptographic protection | SC-13 |

NIST **SP 800-77** (Guide to IPsec VPNs) and **SP 800-113** (Guide to SSL VPNs)
are the background references for VPN policy; both predate WireGuard but their
principles (protect the tunnel, control the boundary, manage keys) apply.

* * *

## Troubleshooting

- **No handshake in `wg show`.** The server's UDP port is not reachable, or a key
  or endpoint is wrong. Confirm the firewall allows `51820/udp` (Step 5), the
  client's `Endpoint` is correct, and each side has the *other* side's **public**
  key (a common mistake is pasting your own).
- **Handshake works but no internet (full tunnel).** Routing or NAT is missing on
  the server. Recheck Step 4: `ip_forward` must be `1` (`sysctl net.ipv4.ip_forward`)
  and masquerade must be active for the tunnel subnet.
- **DNS does not resolve, or leaks to the local network.** The `DNS=` line needs a
  `resolvconf` provider. On `systemd-resolved` systems it usually works; otherwise
  install `openresolv`. Confirm your lookups use the tunnel resolver with the
  checks in the Secure DNS guide.
- **Tunnel drops when idle.** Add `PersistentKeepalive = 25` on the client (Step 6);
  a NAT device timed out the mapping.
- **`ifconfig.me` still shows your local IP.** `AllowedIPs` on the client is not
  `0.0.0.0/0, ::/0`, so traffic is not being routed into the tunnel.

* * *

## How to roll back

```bash
sudo systemctl disable --now wg-quick@wg0    # stop and unset boot start
sudo wg-quick down wg0                        # if still up
sudo rm /etc/wireguard/wg0.conf               # remove the config (keeps keys elsewhere if you saved them)
```

If you enabled forwarding and NAT only for this VPN, remove
`/etc/sysctl.d/70-wireguard-routing.conf` (then `sudo sysctl --system`) and undo
the masquerade rule from Step 4.

* * *

## What you achieved

- An encrypted, authenticated **WireGuard tunnel** between your own machines, using
  modern fixed cryptography with no downgrade path.
- Either **secure remote access** to a private subnet, or a **full-tunnel VPN**
  that takes a device's traffic off the local network and presents your server's
  exit IP.
- **In-tunnel DNS**, so lookups do not leak to the network you are on.
- A clear-eyed understanding that this protects your traffic in transit and moves
  trust to your server's host; it is not anonymity.

* * *

## References

- [WireGuard](https://www.wireguard.com/) and its [protocol page](https://www.wireguard.com/protocol/)
- [`wg(8)` manual](https://man7.org/linux/man-pages/man8/wg.8.html)
- [`wg-quick(8)` manual](https://man7.org/linux/man-pages/man8/wg-quick.8.html)
- [WireGuard whitepaper (Donenfeld, NDSS 2017)](https://www.wireguard.com/papers/wireguard.pdf)
- [Ubuntu Server: WireGuard VPN](https://ubuntu.com/server/docs/how-to/wireguard-vpn/)
- [NIST SP 800-53 Rev 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)
- [NIST SP 800-77 Rev 1: Guide to IPsec VPNs](https://csrc.nist.gov/pubs/sp/800/77/r1/final)
