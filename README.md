# Linux Security Guides

A growing collection of step-by-step guides and reference material for improving
security and privacy. The hands-on guides focus on Linux-based systems. The
governance and standards topics (ISMS, ISO/IEC 27001, Shadow IT) are
platform-agnostic and apply to any environment.

The aim is to be practical and realistic. Every guide is written to be followed end
to end, carries a *last verified* date, and explains the reason behind each step,
not just what to type.

## What's inside

### Guides (hands-on, step-by-step)

#### Host hardening

| Guide | What it covers | Frameworks | Distros |
| --- | --- | --- | --- |
| [Host firewall](guides/host-firewall-hardening.md) | Default-deny host firewall with firewalld, ufw, or nftables | CIS, NIST | Fedora, RHEL, Ubuntu, CachyOS |
| [Kernel sysctl](guides/kernel-network-hardening-sysctl.md) | Network-stack kernel parameters: anti-spoofing, redirects, SYN floods | CIS, NIST | Fedora, RHEL, Ubuntu, CachyOS |

#### Remote access

| Guide | What it covers | Frameworks | Distros |
| --- | --- | --- | --- |
| [SSH + fail2ban](guides/secure-ssh-with-fail2ban.md) | Key-only SSH, `sshd` hardening, and fail2ban brute-force protection | CIS, NIST | Fedora, RHEL, Ubuntu, CachyOS |

#### Network privacy

| Guide | What it covers | Frameworks | Distros |
| --- | --- | --- | --- |
| [Secure DNS](guides/secure-dns-quad9-dot-dnssec.md) | Encrypted DNS with DNS-over-TLS and DNSSEC validation via Quad9 | None | Fedora, RHEL, Ubuntu, CachyOS |
| [Wi-Fi privacy](guides/wifi-privacy-mac-randomization.md) | MAC randomization and IPv6 privacy to limit device tracking | None | Fedora, RHEL, Ubuntu, CachyOS |

#### VPN

| Guide | What it covers | Frameworks | Distros |
| --- | --- | --- | --- |
| [WireGuard VPN](guides/wireguard-vpn-self-hosted.md) | Self-hosted WireGuard VPN, full-tunnel or split-tunnel | NIST | Fedora, RHEL, Ubuntu, CachyOS |

#### Compliance

| Guide | What it covers | Frameworks | Distros |
| --- | --- | --- | --- |
| [CIS with OpenSCAP](guides/cis-hardening-with-openscap.md) | Scan and remediate against the CIS Benchmarks with OpenSCAP | CIS | Fedora, RHEL, Ubuntu |

### Topics (concepts & governance)

| Topic | What it covers |
| --- | --- |
| [Shadow IT](topics/shadow-it.md) | Why unapproved tools undermine an ISMS, and how to manage them, mapped to ISO/IEC 27001:2022 |
| [ISMS documentation structure](topics/isms-structure.md) | A reference folder layout for an ISO/IEC 27001:2022 ISMS |

## Roadmap (planned)

These topics are intended for this repository but are **not written yet**. They
are listed so the direction is clear, not as available content.

**Security & hardening**
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
