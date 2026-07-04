# How to Scan and Harden a Linux Server Against CIS Benchmarks with OpenSCAP (Fedora, RHEL, Ubuntu)

> **Published:** 2026-07-04. **Last verified:** 2026-07-04 against ComplianceAsCode
> content v0.1.81 (released 2026-06-01, the latest at the time of writing), the
> `oscap(8)` and `oscap-xccdf(8)` man pages, the OpenSCAP user manual, and Red Hat's
> security hardening documentation. Package versions checked: Fedora 44
> (`scap-security-guide` 0.1.81, `openscap-scanner` 1.4.4), RHEL 9 and 10 family
> (`scap-security-guide` shipped in AppStream and pinned per minor release, so its
> version trails upstream; checked against CentOS Stream 9 and 10), Ubuntu 24.04 LTS
> (`openscap-scanner`
> 1.3.9; its `ssg-*` packages are 0.1.71 and do not cover 24.04 itself, so the
> upstream content archive is used instead). Covers Fedora 44, RHEL 9 and 10, and
> Ubuntu Server 24.04 LTS.

> The CIS Benchmarks are consensus configuration baselines for hardening an
> operating system. OpenSCAP is the scanner that can measure a running system
> against such a baseline, and the SCAP Security Guide (the built content of the
> [ComplianceAsCode project](https://github.com/ComplianceAsCode/content)) supplies
> the machine-readable CIS profiles it measures against. This guide walks through
> scanning a server, reading the report, generating a remediation script for
> exactly the rules that failed, applying it safely, and proving the improvement
> with a re-scan.

## What you are actually running

Three pieces work together, and naming them once avoids confusion later:

- **OpenSCAP** (`oscap`) is the scanner. It evaluates a system against SCAP
  content and produces results and reports. It ships in your distribution's
  repositories as `openscap-scanner`.
- **ComplianceAsCode/content** is the open source project that writes the rules.
  Its built release archive is still named **scap-security-guide** (SSG) for
  historical reasons. Each release ships one **SCAP source data stream** per
  operating system, named `ssg-<product>-ds.xml` (for example
  `ssg-ubuntu2404-ds.xml`), containing every profile for that OS.
- **A CIS profile** inside the data stream selects and configures the rules that
  correspond to one CIS Benchmark level, for example "Level 1 - Server".

One accuracy point worth getting right in client conversations: these profiles
are community-maintained content that **aligns to** a specific published CIS
Benchmark version and carries CIS trademark attribution. The project does not
claim CIS certification, and neither does Red Hat; CIS's own certified assessor
is CIS-CAT Pro. A clean SSG scan is strong evidence of alignment, not a CIS
certificate.

## Distribution cheat-sheet

| | Fedora 44 | RHEL 9 / 10 | Ubuntu Server 24.04 |
| --- | --- | --- | --- |
| Scanner install | `sudo dnf install openscap-scanner` | `sudo dnf install openscap-scanner` | `sudo apt install openscap-scanner` |
| Content source | `sudo dnf install scap-security-guide` (carries current 0.1.81) | `sudo dnf install scap-security-guide` (Red Hat-supported content) | Download the upstream release zip (see Step 1) |
| Data stream | `/usr/share/xml/scap/ssg/content/ssg-fedora-ds.xml` | `/usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml` or `ssg-rhel10-ds.xml` | `scap-security-guide-0.1.81/ssg-ubuntu2404-ds.xml` |
| CIS server profiles | `cis_server_l1`, `cis` (L2, **DRAFT** status) | `cis_server_l1`, `cis` (= Level 2 Server) | `cis_level1_server`, `cis_level2_server` |
| Aligned benchmark | CIS Fedora 40 Branch Benchmark (draft, best effort) | CIS RHEL 9 Benchmark v2.0.0 / RHEL 10 Benchmark v1.0.1 | CIS Ubuntu Linux 24.04 LTS Benchmark v1.0.0 |

Scope limits, stated up front:

- **CachyOS and Arch are not covered.** ComplianceAsCode v0.1.81 ships 38 product
  data streams and none of them target Arch Linux; the scanner itself is only in
  the AUR. There is no supported CIS SCAP content to scan against.
- **Ubuntu 26.04 LTS has no OS-matched content yet.** No `ubuntu2604` data stream
  exists in v0.1.81 (or anywhere else). The newest Ubuntu content is
  `ubuntu2404`, so this guide scans 24.04. Scanning 26.04 with 24.04 content is
  not a supported combination and produces misleading results.
- **On RHEL, use the dnf-shipped content, not the upstream zip.** Red Hat
  officially supports the SCAP Security Guide versions shipped with each minor
  release (Red Hat KB 6337261) and warns that content is not always backward
  compatible. The upstream zip is for distributions whose packaged content is
  missing or stale, which is exactly Ubuntu's situation below.

## Prerequisites

- A test system or snapshot first. Remediation rewrites system configuration and
  there is no automated undo (more on this in Step 6).
- Root access via `sudo`, and a working **non-root administrative account**. The
  CIS profiles disable SSH root login, so if you administer the box as root over
  SSH you will lose that path when `sshd` reloads.
- A few hundred megabytes of free space on Ubuntu for the content archive and its extracted contents (the download alone is about 175 MB).

## Step 1: Install the scanner and the content

**Fedora 44** ships the current upstream content in its own repositories:

```bash
sudo dnf install openscap-scanner scap-security-guide
ls /usr/share/xml/scap/ssg/content/ | grep fedora
```

**RHEL 9 and 10** (same packages, Red Hat-supported content):

```bash
sudo dnf install openscap-scanner scap-security-guide
ls /usr/share/xml/scap/ssg/content/ | grep rhel
```

**Ubuntu 24.04** needs the scanner from the archive but the content from
upstream. The archive's `ssg-*` packages are stuck at 0.1.71 (December 2023) and,
worse, that build predates the 24.04 product entirely: it contains no
`ssg-ubuntu2404-ds.xml`, so the packaged content cannot scan the OS it ships on.
Download the release archive instead, and verify its checksum:

```bash
sudo apt update && sudo apt install openscap-scanner unzip
wget https://github.com/ComplianceAsCode/content/releases/download/v0.1.81/scap-security-guide-0.1.81.zip
wget https://github.com/ComplianceAsCode/content/releases/download/v0.1.81/scap-security-guide-0.1.81.zip.sha512
sha512sum -c scap-security-guide-0.1.81.zip.sha512
unzip scap-security-guide-0.1.81.zip
cd scap-security-guide-0.1.81
```

The zip extracts to a single `scap-security-guide-0.1.81/` directory with every
`ssg-<product>-ds.xml` at the top level, plus prebuilt `bash/` and `ansible/`
remediation for whole profiles and human-readable HTML `guides/`. Check the
[releases page](https://github.com/ComplianceAsCode/content/releases) for a newer
version before you start; the project releases several times a year, and the
commands below only need the version string updated.

## Step 2: List the profiles and pick one

```bash
# Ubuntu (inside the extracted directory)
oscap info ssg-ubuntu2404-ds.xml

# Fedora / RHEL
oscap info /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

The output lists every profile with its title and full ID. The CIS profile IDs
follow the pattern `xccdf_org.ssgproject.content_profile_<name>`:

| Distribution | Profile ID suffix | CIS Benchmark target |
| --- | --- | --- |
| Ubuntu 24.04 | `cis_level1_server` | Level 1 - Server |
| Ubuntu 24.04 | `cis_level2_server` | Level 2 - Server |
| Ubuntu 24.04 | `cis_level1_workstation` / `cis_level2_workstation` | Workstation levels |
| RHEL 9 / 10 | `cis_server_l1` | Level 1 - Server |
| RHEL 9 / 10 | `cis` | Level 2 - Server (the plain `cis` ID is L2 Server) |
| RHEL 9 / 10 | `cis_workstation_l1` / `cis_workstation_l2` | Workstation levels |
| Fedora | `cis_server_l1`, `cis`, `cis_workstation_l1`, `cis_workstation_l2` | **DRAFT** profiles |

Which one to pick: **Level 1** is the baseline CIS considers broadly applicable
with limited breakage risk; **Level 2** adds defense-in-depth for
higher-security environments and can reduce functionality. Start with Level 1 - Server
on a server. The Fedora profiles describe themselves as draft, experimental, and
maintained best-effort against the CIS Fedora 40 Branch Benchmark; they are fine
for a personal baseline but do not present a Fedora scan to a client as a CIS
Benchmark result. The same data streams also carry non-CIS profiles (DISA STIG,
ANSSI, PCI DSS, depending on product), which is useful when a client's framework
is not CIS.

## Step 3: Scan the system

Run the evaluation as root so every check can read the files it needs; an
unprivileged scan degrades into `error`/`unknown` results for root-only checks.
One command produces both machine-readable results and an HTML report:

```bash
# Ubuntu 24.04, CIS Level 1 Server
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_level1_server \
  --results results.xml \
  --report report.html \
  ssg-ubuntu2404-ds.xml

# RHEL 9 example (the suffix shorthand also works: --profile cis_server_l1)
sudo oscap xccdf eval \
  --profile cis_server_l1 \
  --results results.xml \
  --report report.html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
```

Do not be alarmed by the exit code: `oscap` returns **0** only when every rule
passes, **2** when at least one rule failed or came back unknown, and **1** on an
actual evaluation error. On a first scan, 2 is the expected outcome, and in
scripts or CI you should treat it as "scan completed with findings", not as a
command failure.

## Step 4: Read the report

Open `report.html` in a browser (`scp` it off a headless server). It shows a
compliance score, a rule-by-rule pass/fail table, and for each rule the
rationale, the exact check, and the remediation it would apply. Three result
types deserve attention beyond pass/fail:

- **notapplicable**: the rule does not apply to this system (wrong platform,
  package absent). Normal.
- **notchecked**: the rule has no automated check and needs manual review.
- **error**: the check could not run, most often a scan executed without root.

Skim the failed rules before remediating anything. This is where you learn what
the profile is about to change, and it is the moment to decide which rules you
do not want (Step 6 shows how to skip them).

## Step 5: Generate remediation for what failed

You can generate fixes two ways, and the difference matters:

- **From the profile** (`--profile` against the data stream): the script tries to
  fix **every rule in the profile**, whether or not your system already passes
  it. This is the "bring a fresh build to baseline" option, and it is what the
  prebuilt scripts in the release's `bash/` directory are.
- **From your scan results** (`--result-id` against `results.xml`): the script
  fixes **only the rules that failed on this system**. Smaller, more reviewable,
  and the right default on an existing server.

Generate the results-based script:

```bash
# Find the TestResult ID inside your results file
oscap info results.xml
# It follows the pattern xccdf_org.open-scap_testresult_<profile-id>

oscap xccdf generate fix \
  --fix-type bash \
  --result-id xccdf_org.open-scap_testresult_xccdf_org.ssgproject.content_profile_cis_level1_server \
  results.xml > cis-remediation.sh
```

For the whole-profile variant, swap in
`--profile cis_level1_server ssg-ubuntu2404-ds.xml` in place of the
`--result-id`/results pair.

Expect lines reading `FIX FOR THIS RULE '...' IS MISSING!` when you run the
script. Not every rule has an automated fix: repartitioning (`partition_for_tmp`
and friends) and the GRUB bootloader password (`grub2_password`) are the classic
examples, since no sane script repartitions a live disk or invents a password
for you. Those rules stay failed until you address them by hand, and on a
cloud image with a single root volume, the partition rules may be a documented
accepted risk rather than a fix.

## Step 6: Review before you apply

Read the generated script before running it. It is plain bash, one commented
block per rule. There is no automated way back: Red Hat states outright that it
provides no method to revert security-hardening remediations (KB 7032665), so
your rollback is the snapshot you take now, not a flag later. The sharp edges to
check for, all confirmed against the v0.1.81 Ubuntu CIS Level 1 Server profile:

- **The firewall can change out from under you.** CIS requires a single firewall
  utility. The profile's firewall variable defaults to **nftables**, and the
  generated fix for `package_ufw_removed` runs `apt-get remove -y ufw` when the
  variable is not set to ufw. If your ruleset lives in ufw (as on a stock
  hardened Ubuntu or the [host firewall guide](host-firewall-hardening.md) Track
  B), either re-express it in nftables first or skip those rules.
- **SSH root login goes away and weak crypto is disabled.**
  `sshd_disable_root_login` plus the strong cipher, KEX, and MAC lists take
  effect when `sshd` reloads. Key-based auth for normal users is untouched (no
  CIS rule disables `PubkeyAuthentication`), but log in with your non-root
  account and keep one working session open until you have verified a fresh
  login.
- **PAM is rewritten through `pam-auth-update` on Ubuntu.** The fixes install
  `pam-configs` profiles (password quality, faillock and related) and apply them
  noninteractively. If conflicting profiles are selected you can see
  `Incompatible PAM profiles selected` messages from `pam-auth-update` itself.
  After remediation, verify you can still `sudo` and log in before closing your
  root session.
- **Skipping a rule is legitimate.** Delete the rule's block from the script, or
  build a tailored profile: `autotailor` (in `openscap-utils`) writes a tailoring
  file you pass to `oscap` with `--tailoring-file`, which is the auditable way to
  document "we deviate from 4.3.2 because...". The SCAP Workbench GUI does the
  same job, though it is currently not packaged for Fedora 44.

Then protect yourself:

```bash
# 1. Snapshot or image the machine (cloud snapshot, VM snapshot, or backup)
# 2. Keep a root shell open in a second terminal until you are done
# 3. Apply
chmod +x cis-remediation.sh
sudo bash ./cis-remediation.sh |& tee remediation.log
```

The script must run as root and only on the OS it was generated for; every rule
carries a platform gate that prints `Remediation is not applicable, nothing was
done` on a mismatch. There is also `sudo oscap xccdf eval --remediate ...`, which
fixes failures during the scan itself in one shot. Red Hat's documentation warns
that used carelessly it can leave a system non-functional, and on a live server
the generate-review-apply loop above is the defensible process; keep
`--remediate` for disposable builds.

## Step 7: Reboot and re-scan

Some fixes only bite after a reboot: kernel module blacklists
(`kernel_module_*_disabled` rules are flagged `Reboot: true`), GRUB command line
changes such as enabling AppArmor, and anything read once at boot. The score
that counts is the one from a fresh boot:

```bash
sudo reboot
# after logging back in (with your non-root account):
sudo oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_level1_server \
  --results results-after.xml \
  --report report-after.html \
  ssg-ubuntu2404-ds.xml
```

Compare `report-after.html` with the first report. Remaining failures fall into
three buckets: rules with no automated fix (Step 5), rules you deliberately
skipped or tailored out (document them), and genuine leftovers to fix by hand.
Keep both reports; a before/after pair is exactly the evidence an auditor or a
client wants to see.

## Optional: the Ansible route

For fleets, drift correction, or anywhere you already run configuration
management, generate a playbook instead of a shell script:

```bash
oscap xccdf generate fix \
  --fix-type ansible \
  --profile cis_level1_server \
  ssg-ubuntu2404-ds.xml > cis-playbook.yml

# Dry run first, then apply
ansible-playbook -i "localhost," -c local --check cis-playbook.yml
sudo ansible-playbook -i "localhost," -c local cis-playbook.yml
```

The playbooks are idempotent, which makes them safer to re-run than the bash
script. Two requirements: Ansible 2.9 or newer, and the `community.general` and
`ansible.posix` collections, so install the full `ansible` package (or add those
two collections to `ansible-core` with `ansible-galaxy`). Ansible coverage of
rules can lag bash coverage, so a bash-remediated system and an
Ansible-remediated system may not land on identical scores.

## Troubleshooting

- **`oscap` exited with code 2.** That means the scan worked and found failures.
  Only exit code 1 is an error.
- **Many `error` results.** You ran the scan without `sudo`. Re-run as root.
- **Ubuntu's packaged `ssg-*` content has no 24.04 data stream.** Correct, and
  why Step 1 downloads the upstream zip. The archive packages are at 0.1.71,
  which predates the `ubuntu2404` product.
- **The report shows `notchecked` for rules I care about.** Those rules ship no
  automated check. The HTML guide for your profile (in the release `guides/`
  directory) describes the manual verification.
- **A remediated rule still fails after re-scan.** Check whether it is flagged
  `Reboot: true` in the report and whether you rebooted; then check
  `remediation.log` for its block, which may have printed a missing-fix or
  not-applicable message.
- **I locked myself out of SSH.** Console access (or your still-open root
  session) is the way back in: re-enable what you need in
  `/etc/ssh/sshd_config.d/`, or restore the snapshot. This is why Step 6 says to
  keep a session open and verify a fresh login before disconnecting.

## How to roll back

There is no automated revert. The supported path back is the snapshot or image
you took in Step 6. For a single unwanted change, the remediation script itself
tells you what it did: find the rule's block in `cis-remediation.sh` or
`remediation.log`, see which file it edited or package it removed, and reverse
that one change by hand (for example `sudo apt install ufw` and re-enable it if
you decide to keep ufw as your firewall after all, then set the profile's
firewall variable accordingly in a tailoring file so the next scan agrees).

## What you achieved

- Your server is measured against a named, versioned CIS Benchmark
  (Ubuntu 24.04 v1.0.0, RHEL 9 v2.0.0, or RHEL 10 v1.0.1) with a scanner and
  content you can pin and re-run.
- Failures were remediated by generated, reviewed code rather than by hand, with
  the exceptions consciously skipped and documented via tailoring.
- You hold before and after reports as audit evidence, and a repeatable loop
  (scan, remediate, reboot, re-scan) for drift checks.

## References

- [ComplianceAsCode/content on GitHub](https://github.com/ComplianceAsCode/content)
- [ComplianceAsCode release v0.1.81](https://github.com/ComplianceAsCode/content/releases/tag/v0.1.81)
- [OpenSCAP project](https://www.open-scap.org/)
- [OpenSCAP user manual](https://static.open-scap.org/openscap-1.3/oscap_user_manual.html)
- [`oscap(8)` man page](https://www.mankier.com/8/oscap)
- [Red Hat: Supported versions of the SCAP Security Guide in RHEL (KB 6337261)](https://access.redhat.com/articles/6337261)
- [Red Hat Enterprise Linux 9: Security hardening guide](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/security_hardening/index) (scanning for compliance; note that remediations cannot be automatically reverted, Red Hat KB 7032665)
- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks)
