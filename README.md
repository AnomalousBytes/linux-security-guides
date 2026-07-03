# Linux Security Guides

A growing collection of step-by-step guides and reference material for improving
security and privacy. The hands-on guides focus on Linux-based systems. The
governance and standards topics (ISMS, ISO/IEC 27001, Shadow IT) are
platform-agnostic and apply to any environment.

The aim is to be practical and honest. Every guide is written to be followed end
to end, carries a *last verified* date, and explains the reason behind each step,
not just what to type.

## What's inside

### Guides (hands-on, step-by-step)

- [Secure DNS with Quad9, DNS-over-TLS & DNSSEC (Fedora, RHEL, Ubuntu, CachyOS)](guides/secure-dns-quad9-dot-dnssec.md)
- [Secure and harden SSH with fail2ban, CIS/NIST-aligned (Fedora, RHEL, Ubuntu, CachyOS)](guides/secure-ssh-with-fail2ban.md)
- [Host firewall hardening with firewalld, ufw & nftables, CIS/NIST-aligned (Fedora, RHEL, Ubuntu, CachyOS)](guides/host-firewall-hardening.md)
- [Kernel network hardening with sysctl, CIS/NIST-aligned (Fedora, RHEL, Ubuntu, CachyOS)](guides/kernel-network-hardening-sysctl.md)
- [Wi-Fi & on-the-move privacy: MAC randomization and IPv6 privacy (Fedora, RHEL, Ubuntu, CachyOS)](guides/wifi-privacy-mac-randomization.md)
- [Self-hosted WireGuard VPN, NIST-aligned (Fedora, RHEL, Ubuntu, CachyOS)](guides/wireguard-vpn-self-hosted.md)

### Topics (concepts & governance)

- [Shadow IT: The Silent Threat Inside Your ISMS](topics/shadow-it.md)
- [A Practical ISMS Documentation Structure for ISO/IEC 27001:2022](topics/isms-structure.md)

## Roadmap (planned)

These topics are intended for this repository but are **not written yet**. They
are listed so the direction is clear, not as available content.

**Security & hardening**
- CIS Benchmarks with OpenSCAP
- System configuration best practices

**Broader security & governance**
- Infrastructure as Code (IaC) and automation
- DevOps and cloud security practices
- Cybersecurity fundamentals and threat management
- Governance, Risk, and Compliance (GRC)
- Information Security Management Systems (ISMS): full implementation guidance, building on the [structure](topics/isms-structure.md) and [Shadow IT](topics/shadow-it.md) topics already published
- ISO/IEC 27001: per-control deep dives and implementation guidance

## Why this repository?

Good security takes deliberate effort, and the technical and governance sides are
usually treated separately. This repository aims to connect them:

- technical hardening and automation,
- security standards and benchmarks, and
- governance and risk management practices.

It is an early-stage, work-in-progress collection. The [roadmap](#roadmap-planned)
above shows where it is heading.

## Conventions

- File names use lowercase `kebab-case`, named by topic (for example,
  `host-firewall-hardening.md`). Each guide covers all the distributions it
  supports in a single file, with per-distribution commands where they differ.
- `guides/` holds hands-on, step-by-step procedures. `topics/` holds conceptual
  and governance writing.
- Every guide carries a **Last verified** date and the versions it was tested on,
  because security guidance goes stale.

## Contributing

Corrections and improvements are welcome. If a guide is outdated, inaccurate, or
unclear, open an issue or a pull request. Please keep the existing structure and
update the **Last verified** date when you re-test a guide.

## Disclaimer

These guides are provided as-is, without warranty, for educational purposes. They
involve administrative (`sudo`) changes to system configuration. Read each step,
understand what it does, and test on a non-critical system before applying it
anywhere important. You are responsible for changes you make to your own systems.

## License

The content in this repository is licensed under
[Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE). You may
share and adapt it, including commercially, with attribution to anoBytes
Technology Solutions.

© 2026 anoBytes Technology Solutions
