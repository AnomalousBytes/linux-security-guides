# How to Configure Secure DNS with Quad9, DNS-over-TLS, and DNSSEC on Ubuntu 24.04 LTS

> By default, DNS queries are often sent in plaintext, meaning your ISP or anyone on the path can see which sites you visit. 
> Using Quad9 with DNS-over-TLS (DoT) and DNSSEC ensures your queries are private, authenticated, and filtered against malicious domains.

This guide explains how to set up **Quad9** as your DNS provider on Ubuntu 24.04 (GNOME), enabling **DNS-over-TLS (DoT)** and **DNSSEC validation** for maximum privacy and security.

## Prerequisites
- Ubuntu 24.04 LTS (with `systemd-resolved` enabled by default)
- Root privileges (`sudo`)
- Basic familiarity with the terminal

> **Note**: This guide is adapted for Ubuntu 24.04, which uses `systemd-resolved` and `NetworkManager` by default. 
> The steps are similar to Fedora, but Ubuntu requires one small change to ensure NetworkManager integrates properly.

* * *

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
If not, fix it with:
```bash
sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

* * *

## Step 2: Configure NetworkManager to use systemd-resolved

Before changing DNS servers, ensure that **NetworkManager** is passing DNS configuration through `systemd-resolved` instead of directly using your router’s DNS.

Edit:
```bash
sudo nano /etc/NetworkManager/NetworkManager.conf
```

Add (or confirm) the following under `[main]`:
```ini
[main]
dns=systemd-resolved
```

Then restart services:
```bash
sudo systemctl restart NetworkManager
sudo systemctl restart systemd-resolved
```

* * *

## Step 3: Configure Quad9 with DNS-over-TLS

1. Edit the systemd-resolved configuration file:
```bash
sudo nano /etc/systemd/resolved.conf
```

2. Add the following:
```ini
[Resolve]
DNS=9.9.9.9#dns.quad9.net 149.112.112.112#dns.quad9.net 2620:fe::fe#dns.quad9.net 2620:fe::9#dns.quad9.net
FallbackDNS=
DNSOverTLS=yes
DNSSEC=allow-downgrade
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

* * *

## Step 4: Ensure Your Connection Uses Quad9

By default, Ubuntu’s NetworkManager may still use DNS provided by your router. 
To ensure your Wi-Fi or Ethernet connection uses Quad9:

### GUI
1. Open **Settings → Network → Wi-Fi → Gear Icon → IPv4 tab** 
   - Set **Method** to `Automatic (DHCP)`
   - Set **DNS** to `9.9.9.9, 149.112.112.112`
   - Disable the DNS toggle that says `Automatic` 
2. Repeat in the **IPv6** tab:
   - Set **Method** to `Automatic (DHCP)`
   - Set **DNS** to `2620:fe::fe, 2620:fe::9`
   - Disable the DNS toggle that says `Automatic` 
3. Click **Apply**.
4. Restart Network Manager `sudo systemctl restart NetworkManager`

* * *

## Step 5: Test DNS-over-TLS

Check that no plaintext DNS traffic is being sent on port 53:
```bash
sudo tcpdump -i wlo1 port 53
```
You should see no queries, since DNS is encrypted over port 853.

* * *

## Step 6: Test DNSSEC

Run:
```bash
dig sigfail.verteiltesysteme.net
dig sigok.verteiltesysteme.net
```

Expected results:
- `sigfail` → returns **SERVFAIL** (bad signature rejected).
- `sigok` → resolves successfully with the `ad` (Authenticated Data) flag.

* * *

## What You Achieved

- Using **Quad9 resolvers** (`9.9.9.9`, `149.112.112.112`).
- Queries are encrypted with **DNS-over-TLS**.
- Responses are validated with **DNSSEC**.
- **EDNS Client Subnet (ECS)** is disabled by Quad9 → better privacy.

* * *

## Optional: Tailscale / Docker

- **Tailscale** uses its own DNS (`100.100.100.100`) when on VPN. To override this, configure DNS preferences in the Tailscale admin panel. 
- **Docker** containers inherit DNS from the host unless overridden in `daemon.json`.

* * *
## How to check per-interface behavior

When you plug in an Ethernet dongle, run:
```
resolvectl status
```
You’ll see a new Link entry (probably enp…) and it should list:
```
Protocols: … +DNSOverTLS
DNS Servers: 9.9.9.9 149.112.112.112
```

If that’s the case, everything is consistent across Wi-Fi and Ethernet.

* * *

## Conclusion

You now have a **hardened DNS setup** on Ubuntu 24.04 using Quad9 + DoT + DNSSEC. 
This prevents ISP snooping, ensures authenticity of DNS responses, and blocks known malicious domains by default.

* * *
## References
- [Quad9 Official Site](https://www.quad9.net)
- [systemd-resolved Documentation](https://www.freedesktop.org/software/systemd/man/resolved.conf.html)
- [DNSSEC Explained (ICANN)](https://www.icann.org/resources/pages/dnssec-qaa-2014-01-29-en)
