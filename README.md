# Linux Security Guides

A growing collection of step-by-step guides and reference material for improving
security and privacy, with a focus on Linux-based systems and on connecting
hands-on hardening with the standards and governance practices behind it.

The aim is to be practical and honest. Every guide is written to be followed end
to end, carries a *last verified* date, and explains the reason behind each step,
not just what to type.

## What's inside

### Guides (hands-on, step-by-step)

- [Secure DNS with Quad9, DNS-over-TLS & DNSSEC on Fedora 44 (KDE)](guides/secure-dns-quad9-dot-dnssec-fedora-44.md)
- [Secure DNS with Quad9, DNS-over-TLS & DNSSEC on Ubuntu 26.04 LTS](guides/secure-dns-quad9-dot-dnssec-ubuntu-26.04.md)

### Topics (concepts & governance)

- [Shadow IT: The Silent Threat Inside Your ISMS](topics/shadow-it.md)
- [A Practical ISMS Documentation Structure for ISO/IEC 27001:2022](topics/isms-structure.md)

## Roadmap (planned)

These topics are intended for this repository but are **not written yet**. They
are listed so the direction is clear, not as available content.

**Security & hardening**
- Networking privacy & hardening
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

- File names use lowercase `kebab-case`; OS-pinned guides include the version
  (for example, `secure-dns-...-fedora-44.md`, `secure-dns-...-ubuntu-26.04.md`).
- `guides/` holds hands-on, step-by-step procedures. `topics/` holds conceptual
  and governance writing.
- Every guide carries a **Last verified** date and the versions it was tested on,
  because security guidance goes stale.

## License

Released under the [MIT License](LICENSE).
