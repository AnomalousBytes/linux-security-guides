# How to Configure Secure DNS with Quad9, DNS-over-TLS, and DNSSEC on Fedora 42 (KDE)

This guide explains how to set up **Quad9** as your DNS provider on Fedora 42 with KDE, enabling **DNS-over-TLS (DoT)** and **DNSSEC validation** for maximum privacy and security.

---

## Step 1: Check Current DNS Setup

Run:
```bash
resolvectl status
```

- If you see `127.0.0.53` as the DNS server, then **systemd-resolved** is in use.
- Confirm whether your `/etc/resolv.conf` points to the stub resolver:
```bash
ls -l /etc/resolv.conf
```
You should see:
```
/etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf
```

---

## Step 2: Configure Quad9 with DNS-over-TLS

1. Edit the systemd-resolved configuration file:
```bash
sudo nano /etc/systemd/resolved.conf
```

2. Add the following:
```ini
[Resolve]
DNS=9.9.9.9 149.112.112.112
FallbackDNS=2620:fe::fe 2620:fe::9
DNSOverTLS=yes
DNSSEC=yes
```

3. Save and restart the service:
```bash
sudo systemctl restart systemd-resolved
```

4. Verify the configuration:
```bash
resolvectl status
```
You should see:
- `DNSOverTLS=yes`
- `DNSSEC=yes/supported`
- `DNS Servers: 9.9.9.9 149.112.112.112`

---

## Step 3: Test DNS-over-TLS

Check that no plaintext DNS traffic is being sent on port 53:
```bash
sudo tcpdump -i wlp2s0 port 53
```
You should see no queries, since DNS is encrypted over port 853.

---

## Step 4: Test DNSSEC

Run:
```bash
dig sigfail.verteiltesysteme.net
dig sigok.verteiltesysteme.net
```

Expected results:
- `sigfail` → returns **SERVFAIL** (bad signature rejected).
- `sigok` → resolves successfully with the `ad` (Authenticated Data) flag.

---

## What You Achieved

- Using **Quad9 resolvers** (`9.9.9.9`, `149.112.112.112`).
- Queries are encrypted with **DNS-over-TLS**.
- Responses are validated with **DNSSEC**.
- **EDNS Client Subnet (ECS)** is disabled by Quad9 → better privacy.

---

## Optional: Tailscale / Docker

- **Tailscale** uses its own DNS (`100.100.100.100`) when on VPN. To override this, configure DNS preferences in the Tailscale admin panel.  
- **Docker** containers inherit DNS from the host unless overridden in `daemon.json`.

* * *
## How to check per-interface behavior

For example; When plugging in an Ethernet dongle, just run:
`resolvectl status`

You’ll see a new Link entry (probably enp…) and it should list:
```
Protocols: … +DNSOverTLS
DNS Servers: 9.9.9.9 149.112.112.112
```

If that’s the case → everything is consistent across Wi-Fi and Ethernet.

---

## Conclusion

You now have a **hardened DNS setup** on Fedora 42 with KDE using Quad9 + DoT + DNSSEC.  
This prevents ISP snooping, ensures authenticity of DNS responses, and blocks known malicious domains by default.


