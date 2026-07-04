# How to Establish a Secure Server Baseline on Linux (Debian, Ubuntu, RHEL), Mapped to CIS, NIST, and ISO 27001

> **Published:** 2026-07-04. **Last verified:** 2026-07-04 against current Debian,
> Ubuntu, and Red Hat documentation; ComplianceAsCode content v0.1.81; the CIS
> Benchmark listings for Debian, Ubuntu, and RHEL; and, for the control mappings,
> NIST SP 800-53 Rev 5 and ISO/IEC 27001:2022. Covers Debian 12 and 13, Ubuntu
> Server 24.04 LTS, and RHEL 9 and 10. Commands assume `root` or `sudo`. All are
> systemd-based, so `hostnamectl`, `timedatectl`, `localectl`, and `systemctl`
> behave the same except where noted.

> A server baseline is the small set of configuration every server should have
> before it runs anything, and before you measure it against a benchmark. This
> guide is the spine: it sets the foundation that a CIS scan assumes or cannot
> automate, points to the focused guides for firewall, SSH, kernel, and DNS work,
> and then hands broad hardening to the CIS Benchmarks through OpenSCAP. Each step
> is mapped to where CIS, NIST SP 800-53, and ISO/IEC 27001 apply, so the same work
> serves both engineering and audit.

## How this guide fits

The rule that keeps this from repeating the other guides: **every setting lives in
exactly one guide.** Firewall, SSH, kernel parameters, and DNS already have their
own guides, so here they get one line and a link, never repeated steps. This guide
only performs the foundational configuration that no other guide covers.

It is also deliberately not a full hardening manual. A CIS Benchmark applied with
OpenSCAP does the broad, automatable hardening far more completely than prose can.
What sits outside that automation, and therefore belongs here, is three things:

- **Foundation a CIS scan assumes or will not create for you:** the hostname, time
  synchronization, the administrative account, trusted repositories, and automatic
  security updates.
- **Build-time decisions CIS flags but cannot remediate:** disk partitioning (the
  separate-mount rules such as `partition_for_tmp`) and the bootloader password.
  The [CIS with OpenSCAP](cis-hardening-with-openscap.md) guide notes these have no
  automated fix, so they are decided here.
- **The framework story:** when to choose CIS Level 1 versus Level 2, and how the
  baseline maps to NIST and ISO controls for an auditor.

## Build order

```
  1. Foundation (this guide)
     identity, time, admin user, automatic updates, packages,
     logging + audit, mandatory access control, filesystem, bootloader
        │
        ▼
  2. Focused hardening (the linked guides)
     host firewall, SSH + fail2ban, kernel sysctl, encrypted DNS
        │
        ▼
  3. Broad hardening: CIS via OpenSCAP
     scan, remediate, re-scan; Level 1 general, Level 2 high-security
        │
        ▼
  4. Verify and maintain
     re-scan on a schedule, watch for configuration drift
```

Do the foundation on a fresh build before exposing the server. Steps 2 and 3 can
follow in either order, but running the CIS scan last shows what the focused guides
already fixed.

* * *

## Step 1: Identity (hostname, timezone, locale)

Set a fully qualified hostname and map it locally so the machine knows its own
name before logging, certificates, or mail depend on it.

```bash
sudo hostnamectl set-hostname host.example.com
sudo timedatectl set-timezone Region/City        # e.g. America/New_York
```

Add the name to `/etc/hosts`. Debian and Ubuntu use the `127.0.1.1` convention;
RHEL uses the machine's real address:

```
# Debian / Ubuntu
127.0.1.1   host.example.com host
# RHEL (use the actual IP)
192.0.2.10  host.example.com host
```

Locale is the one identity setting that differs by family. On RHEL the locale
ships in a langpack; on Debian it must be generated before it can be set:

```bash
# RHEL 9/10
sudo dnf install glibc-langpack-en
sudo localectl set-locale LANG=en_US.UTF-8

# Debian 12/13
sudo sed -i 's/^# en_US.UTF-8/en_US.UTF-8/' /etc/locale.gen
sudo locale-gen && sudo update-locale LANG=en_US.UTF-8

# Ubuntu 24.04
sudo locale-gen en_US.UTF-8 && sudo update-locale LANG=en_US.UTF-8
```

Verify with `hostnamectl status`, `hostname -f`, and `localectl status`.

## Step 2: Time synchronization

Accurate time underpins logs, certificate validation, authentication, and audit
correlation. The default daemon differs by family, so confirm one is running
rather than installing a second.

- **Debian 12/13 and Ubuntu 24.04** use **`systemd-timesyncd`** by default.
- **RHEL 9/10** use **`chrony`** by default.

```bash
timedatectl status                       # want: "System clock synchronized: yes"

# Debian / Ubuntu (systemd-timesyncd)
sudo systemctl enable --now systemd-timesyncd
timedatectl show-timesync --all

# RHEL (chrony)
sudo systemctl enable --now chronyd
chronyc sources -v
```

Run only one implementation. Installing `chrony` on Debian or Ubuntu removes the
`systemd-timesyncd` package automatically.

## Step 3: Administrative account and sudo

Create a named non-root account for administration and grant it `sudo`. The group
that carries sudo rights differs: Debian and Ubuntu use **`sudo`**, RHEL uses
**`wheel`** (already enabled in `/etc/sudoers`).

```bash
# Debian / Ubuntu
sudo adduser alice
sudo usermod -aG sudo alice

# RHEL (adduser is a non-interactive symlink to useradd)
sudo useradd alice && sudo passwd alice
sudo usermod -aG wheel alice
```

Confirm with `id alice` and `sudo -lU alice`. Disabling direct root login over SSH
is part of the [SSH + fail2ban](secure-ssh-with-fail2ban.md) guide; do that once
this account works. Password quality and lockout policy (PAM `pwquality` and
`faillock`) are enforced by the CIS profile in Step 10.

## Step 4: Automatic security updates

Unpatched packages are one of the most common ways a server is compromised, so apply
security updates without waiting for a human. The tool differs by family.

**Debian 12/13 and Ubuntu 24.04** use `unattended-upgrades`. Ubuntu Server enables
it by default; on Debian confirm it, since minimal and cloud images may omit it.

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure -plow unattended-upgrades
```

The default in `/etc/apt/apt.conf.d/50unattended-upgrades` installs the `-security`
origin only; the general `-updates` line is commented out. `20auto-upgrades` enables
the daily unattended run. Dry-run with `sudo unattended-upgrade --dry-run -d`.

**RHEL 9/10** use `dnf-automatic` (not installed by default):

```bash
sudo dnf install dnf-automatic
```

Set security-only apply in `/etc/dnf/automatic.conf`, then enable the one timer
that reads that config:

```ini
[commands]
upgrade_type = security
apply_updates = yes
```

```bash
sudo systemctl enable --now dnf-automatic.timer
```

Enable only `dnf-automatic.timer`. The alternative preset timers
(`dnf-automatic-install.timer` and friends) override `automatic.conf`'s download
and apply behavior with fixed presets. On CentOS Stream, note that `upgrade_type = security` matches nothing,
because Stream ships no security errata metadata.

## Step 5: Minimize installed software and services

Every listening service is attack surface. Inventory what runs, then disable or
remove what the server does not need.

```bash
systemctl list-unit-files --type=service --state=enabled
ss -tulpn                               # trace a listening port to its process

sudo systemctl disable --now <unit>     # stop now and at boot
# Debian / Ubuntu
sudo apt purge <pkg> && sudo apt autoremove --purge
# RHEL
sudo dnf remove <pkg> && sudo dnf autoremove
```

The CIS profile in Step 10 flags many of these by name; removing the obvious ones
first makes the scan shorter to read.

## Step 6: Persistent logging and auditing

Two log layers matter: the systemd journal, and the kernel audit trail.

**Make the journal persistent.** The default keeps logs in RAM unless
`/var/log/journal/` exists. This matters most on Debian 12/13, which no longer
install `rsyslog`, so without persistence the system journal does not survive a
reboot. Ubuntu 24.04 already ships a persistent journal.

```bash
sudo mkdir -p /etc/systemd/journald.conf.d
printf '[Journal]\nStorage=persistent\n' | \
  sudo tee /etc/systemd/journald.conf.d/99-persistent.conf
sudo systemctl restart systemd-journald
```

**Enable auditd.** It records security-relevant events (logins, privilege use,
file changes) and is what the CIS logging rules configure.

```bash
# RHEL: the audit package is installed; enable it
sudo systemctl enable --now auditd
# Debian / Ubuntu: install first
sudo apt install auditd audispd-plugins && sudo systemctl enable --now auditd
```

On RHEL, stop or restart auditd with `service auditd restart`, not `systemctl`; the
unit refuses a manual stop by design.

## Step 7: Mandatory access control

Keep the kernel's mandatory access control enabled. It confines a compromised
service to what its policy allows, which is a different layer from the firewall or
file permissions.

- **RHEL 9/10** ship **SELinux** in enforcing mode with the `targeted` policy.
  Keep it that way; use `permissive` only to debug, never `disabled`.

  ```bash
  getenforce        # want: Enforcing
  sestatus
  ```

- **Debian 12/13 and Ubuntu 24.04** ship **AppArmor** enabled by default.

  ```bash
  sudo aa-status
  ```

If an application genuinely needs an exception, write a policy for that one case
rather than turning the whole system off. The reverse-proxy SELinux example in the
[nginx guide](nginx-reverse-proxy-letsencrypt-aws.md) shows the pattern.

## Step 8: Filesystem layout and bootloader password

These two are build-time decisions. OpenSCAP reports them as failures but cannot
create the partitions or set the password for you, so handle them here.

- **Separate mounts with restrictive options.** CIS expects `/tmp`, `/var`,
  `/var/log`, `/var/tmp`, `/home`, and `/dev/shm` on their own mounts carrying
  `nodev`, `nosuid`, and `noexec` where applicable. This is far easier at install
  time; on an existing single-volume cloud image, record it as an accepted risk in
  your baseline rather than repartitioning a live disk.
- **Set a bootloader password** so no one with console access can edit the kernel
  command line to bypass controls.

```bash
# RHEL 9/10 (writes the hash; no regeneration needed)
sudo grub2-setpassword

# Debian / Ubuntu
grub-mkpasswd-pbkdf2                     # copy the grub.pbkdf2.sha512... hash
# append to /etc/grub.d/40_custom:
#   set superusers="admin"
#   password_pbkdf2 admin grub.pbkdf2.sha512.10000.XXXX...
sudo update-grub
```

On Debian and Ubuntu, add `--unrestricted` to the menu entry if you want the
password required only to edit entries, not to boot unattended.

* * *

## Step 9: Apply the focused guides

The rest of the host hardening lives in dedicated guides. Apply the ones that fit
the server's role:

- **[Host firewall](host-firewall-hardening.md)** for a default-deny inbound
  policy with firewalld, ufw, or nftables.
- **[SSH + fail2ban](secure-ssh-with-fail2ban.md)** for key-only SSH, `sshd`
  hardening, and brute-force protection. This is where root SSH login is disabled.
- **[Kernel sysctl hardening](kernel-network-hardening-sysctl.md)** for the
  network-stack parameters (anti-spoofing, redirects, SYN-flood defense).
- **[Secure DNS](secure-dns-quad9-dot-dnssec.md)** where encrypted resolution is
  wanted.

## Step 10: Broad hardening with CIS through OpenSCAP

With the foundation in place, measure and remediate against a CIS Benchmark using
the [CIS with OpenSCAP](cis-hardening-with-openscap.md) guide. **Level 1** is the
base recommendation, meant to be adopted promptly without an extensive performance
or usability cost; start here on a general server. **Level 2** adds defense in
depth for environments where security is paramount and can reduce functionality.

The content available differs by distribution, so the hand-off is not identical:

| Distribution | OpenSCAP CIS content | How to run it |
| --- | --- | --- |
| RHEL 9 / 10 | `scap-security-guide` from AppStream (Red Hat-supported) | Exactly as the OpenSCAP guide describes |
| Ubuntu 24.04 | Upstream release zip (`ssg-ubuntu2404-ds.xml`) | As the OpenSCAP guide describes |
| Debian 12 | Upstream release zip (`ssg-debian12-ds.xml`), profiles `cis_level1_server` / `cis_level2_server` | Same flow, swap the data stream file |
| Debian 13 | **No CIS profile shipped yet** (v0.1.81 carries ANSSI only) | See below |

For **Debian 12**, `openscap-scanner` is in apt and the same upstream content zip
used in the OpenSCAP guide's Ubuntu path contains `ssg-debian12-ds.xml`, so the flow
is identical with the file name swapped. One caveat to state to a client: that content
tracks the CIS Debian 12 Benchmark **v1.1.0**, while CIS has since published
**v2.0.0**, so a scan aligns to the older recommendation set.

For **Debian 13**, the released content has no CIS profile. Until a newer content
release ships one, the accurate options are to apply the CIS Debian 13 Benchmark
v1.0.0 by hand (or with CIS-CAT Pro), or to scan with the ANSSI BP-028 profiles
that the Debian 13 data stream does carry. ANSSI is a legitimate hardening baseline
but it is not CIS; do not present one as the other.

## Where CIS, NIST, and ISO 27001 apply

Each baseline step maps to a benchmark section and to the control an auditor cites.
NIST control titles and ISO Annex A titles below are quoted from the standards.

| Baseline step | CIS Benchmark area | NIST SP 800-53 Rev 5 | ISO/IEC 27001:2022 Annex A |
| --- | --- | --- | --- |
| Identity, config settings (1) | Initial Setup | CM-2 Baseline Configuration; CM-6 Configuration Settings | A.8.9 Configuration management |
| Time synchronization (2) | Time Synchronization | SC-45 System Time Synchronization | A.8.17 Clock synchronization |
| Admin account and sudo (3) | Access, Authentication and Authorization | AC-2 Account Management; AC-6 Least Privilege; IA-5 Authenticator Management | A.8.2 Privileged access rights; A.5.15 Access control |
| Automatic security updates (4) | System Maintenance | SI-2 Flaw Remediation | A.8.8 Management of technical vulnerabilities |
| Minimize software and services (5) | Services | CM-7 Least Functionality | A.8.19 Installation of software on operational systems |
| Logging and auditing (6) | Logging and Auditing | AU-2 Event Logging; AU-3 Content of Audit Records; AU-12 Audit Record Generation | A.8.15 Logging |
| Mandatory access control (7) | Mandatory Access Control | AC-3(3) Mandatory Access Control | A.8.3 Information access restriction; A.8.7 Protection against malware |
| Filesystem and bootloader (8) | Initial Setup | CM-6 Configuration Settings; IA-5 Authenticator Management | A.8.9 Configuration management; A.8.5 Secure authentication |

CIS numbering is version-specific and differs across the five benchmarks this guide
touches, so the CIS column names the benchmark area rather than a rule number; the
exact rules are what the OpenSCAP scan in Step 10 reports. The ISO controls tie this
technical baseline to the governance side: see the
[ISMS documentation structure](../topics/isms-structure.md) and
[Shadow IT](../topics/shadow-it.md) topics.

## Step 11: Verify and maintain

A baseline is only true on the day you set it. Keep it true:

- Re-run the CIS scan on a schedule and after any significant change; a rising
  failure count is configuration drift.
- Keep the before and after OpenSCAP reports as evidence.
- Confirm the automatic-update timer is still active (`systemctl list-timers`), and
  that the journal and auditd still record after a reboot.

## What you achieved

- A named, repeatable server baseline: identity, time, an administrative account,
  automatic security updates, persistent logging and auditing, mandatory access
  control enforcing, and the build-time filesystem and bootloader decisions made.
- The focused guides applied for firewall, SSH, kernel, and DNS, with no step
  duplicated between documents.
- Broad hardening measured and remediated against a CIS Benchmark through OpenSCAP,
  at the level the environment needs, with the Debian caveats stated.
- Every step mapped to CIS, NIST SP 800-53, and ISO/IEC 27001, so one effort
  satisfies engineering and audit.

## References

- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks)
- [CIS Debian Linux Benchmarks](https://www.cisecurity.org/benchmark/debian_linux)
- [NIST SP 800-53 Rev 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)
- [ISO/IEC 27001:2022](https://www.iso.org/standard/27001)
- [Ubuntu Server: automatic updates](https://ubuntu.com/server/docs/how-to/software/automatic-updates/)
- [Red Hat: automating software updates in RHEL](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/managing_software_with_the_dnf_tool/assembly_automating-software-updates-in-rhel-9_managing-software-with-the-dnf-tool)
- [Debian Wiki: unattended upgrades](https://wiki.debian.org/UnattendedUpgrades)
