# How to Secure and Harden SSH on Linux, with fail2ban (CIS- and NIST-aligned)

> **Published:** 2026-06-27. Covers Fedora 44, RHEL 10 (and 9), Ubuntu 26.04 LTS
> (Desktop and Server), and CachyOS. Where a command differs by distribution, the
> form for each is given. OpenSSH versions at publication: Fedora 44 10.2p1,
> CachyOS 10.3p1, Ubuntu 26.04 10.2p1, RHEL 10 9.9p1, RHEL 9 8.7p1. fail2ban
> 1.1.0.

> Any machine with SSH exposed to the internet is hit constantly by automated
> bots trying to guess passwords on port 22. The most effective fixes are to turn
> off password authentication entirely (so there is no password to guess) and,
> for servers, to keep the SSH port off the public internet altogether. This
> guide does both, plus CIS- and NIST-aligned `sshd` hardening and fail2ban as
> defense-in-depth.

The order of priority matters, so be honest about what each part buys you:

- **Keeping SSH off the public internet is the single most effective control
  for a server.** A port that the public internet cannot reach cannot be
  brute-forced or hit by an internet-borne 0-day. This is covered in
  [Step 9](#step-9-keep-ssh-off-the-public-internet-overlay-networks).
- **Key-only authentication is the core host control.** With both
  `PasswordAuthentication no` and `KbdInteractiveAuthentication no` (Step 5),
  there is no password path left for brute force to guess.
- **fail2ban is noise and load reduction, not your primary defense.** Once
  passwords are off, bots cannot get in regardless. Its value shrinks further
  once SSH is no longer publicly reachable (see Step 9).
- **A non-standard port is obscurity, not security.** It reduces automated scan
  noise but stops no determined attacker. It is optional and covered last.

This guide maps its settings to the SSH-server section of the **CIS Linux
Benchmarks** (there is no standalone "CIS OpenSSH" benchmark; the SSH controls
live inside the Distribution Independent Linux and per-distro benchmarks) and to
**NIST SP 800-53 Rev 5** and **NISTIR 7966**. See
[CIS and NIST alignment](#cis-and-nist-alignment) for the mapping.

> **Lockout warning:** every step here can lock you out of a remote machine if
> done out of order. Keep your current SSH session open the entire time, test
> changes from a second terminal before closing it, and on a cloud VM know where
> your provider's recovery console is before you start. New to Linux? Read
> [Before you start](#before-you-start) first; it explains the second terminal,
> the placeholders, the recovery console, and the jargon used below.

* * *

## Distribution cheat-sheet

The procedure is the same everywhere; only a few commands differ. Substitute from
this table as you go. Fedora and RHEL share tooling (dnf, `sshd.service`,
firewalld, SELinux), so they are grouped.

| | Fedora 44 / RHEL 10 (and 9) | Ubuntu 26.04 (Desktop/Server) | CachyOS (Arch-based) |
| --- | --- | --- | --- |
| Install SSH server | `sudo dnf install openssh-server` | `sudo apt install openssh-server` | `sudo pacman -S openssh` |
| systemd unit | `sshd` (persistent daemon) | `ssh` (socket-activated; alias `sshd`) | `sshd` |
| Firewall | firewalld (on by default) | ufw (installed, **off** by default) | ufw (**on** by default) |
| Mandatory access control | SELinux, enforcing | AppArmor (no enforced `sshd` profile) | none by default |
| Cipher policy | system-wide crypto-policies | OpenSSH defaults | OpenSSH defaults |
| Install fail2ban | Fedora: `sudo dnf install fail2ban`; RHEL: enable EPEL first (see Step 8) | `sudo apt install fail2ban python3-systemd` | `sudo pacman -S fail2ban` |

A few cross-distro facts worth stating once:

- On **Ubuntu/Debian** the service is `ssh.service`, not `sshd.service` (an
  `sshd` alias exists). On **Fedora/RHEL/CachyOS** it is `sshd.service`.
- On **Ubuntu**, `sshd` is **socket-activated** (`ssh.socket`): a fresh `sshd`
  starts for each connection rather than running as a persistent daemon. This
  changes how reloads and port changes work, noted where it matters. Fedora,
  RHEL, and CachyOS run a persistent `sshd.service`.
- The OpenSSH client and server are **one package (`openssh`) on Arch/CachyOS**,
  but **split** on Fedora, RHEL, and Ubuntu (`openssh-server`).
- On **Fedora and RHEL**, SSH ciphers/MACs/key-exchange follow the **system-wide
  crypto policy**, and values hardcoded in `sshd_config` are ignored. See
  [Step 6](#step-6-cryptographic-algorithms).

* * *

## Prerequisites

- A non-root user account on the server that can use `sudo`.
- Terminal access to both the server and the machine you will connect from.
- The OpenSSH **client** on the machine you connect from (`ssh`, `ssh-keygen`).

* * *

## Before you start

If you are new to Linux, read this short orientation first; the steps assume
these conventions.

**Two machines.** You work with the computer you sit at (your **laptop**, also
called the client) and the **server** you are securing (often a rented cloud
machine, also called a VPS, a Virtual Private Server). The key generation in
Step 3 runs on your laptop. Every other step runs on the server, inside an SSH
session. If you lose track, run `hostname` to see which machine you are on.

**Replace the placeholders.** Commands contain stand-in values you must change to
your own:

- `you` is your login name on the server. Run `whoami` on the server to see it.
- `server` is the server's IP address or hostname, for example `203.0.113.10` or
  `host.example.com`, so `ssh you@server` becomes something like
  `ssh alex@203.0.113.10`.
- `100.x.y.z` (Step 9) is an address a command prints for you; use the exact
  value shown.
- `ssh-users` and `10-hardening.conf` are names this guide picks; keep them.

**Keep a second terminal open.** Several steps say to test "from a second
terminal." That means open another terminal window on your laptop (in GNOME
Terminal, **File > New Window** or **Ctrl+Shift+N**) and start a fresh
`ssh you@server` there, while leaving your first session connected and untouched.
If a change breaks login, the original window is still in, so you can undo it.
This is the safety net for the whole guide.

**Editing files.** When a step says `sudo nano <file>`, it opens the nano text
editor. Paste the block shown below the command into the editor, then press
**Ctrl+O** and Enter to save and **Ctrl+X** to exit. A block that looks like file
contents (settings, not commands) goes inside the file.

**reload vs restart.** `sudo systemctl reload sshd` makes the running SSH service
re-read its config without dropping current connections, so your session stays
up. `sudo systemctl restart sshd` fully stops and starts it (sessions can drop)
and is only needed when the listening port changes.

**If you do lock yourself out.** A cloud provider gives you an out-of-band
**recovery console** (often labelled "Serial Console", "VNC console", or
"Recovery mode" in the dashboard, for example AWS EC2 Serial Console,
DigitalOcean Recovery Console, or Hetzner/Linode LISH) that logs you in even when
SSH is broken. Find yours before you start.

**A few terms used below.**

- **daemon** is a program that runs in the background as a service (sshd is the
  SSH daemon).
- **drop-in** is a small extra config file (here under
  `/etc/ssh/sshd_config.d/`) read in addition to the main config, so your changes
  stay separate and easy to remove.
- **socket activation** (Ubuntu) means systemd holds the SSH port open and starts
  a fresh sshd per connection, so config changes apply on the next login
  automatically and a port change means restarting `ssh.socket`.
- **firewall** controls which network ports are reachable. Fedora and RHEL use
  **firewalld**; Ubuntu and CachyOS use **ufw**.
- **SELinux** (Fedora and RHEL, in "enforcing" mode) is an extra security layer
  that, among other things, only lets sshd use ports labelled as SSH ports, which
  is why changing the port needs a `semanage` command.
- **PAM** is the framework Linux programs use to check logins; it is how you add
  extras such as a one-time code or account lockout.
- **EPEL** is a trusted community package repository for RHEL (fail2ban lives
  there).
- **FIPS** is a US-government cryptography standard; it is only relevant for
  federal or regulated systems. Otherwise skip it.
- **bastion** or **jump host** is one hardened server you SSH into first, then
  hop to other machines from there.
- **file modes 700 and 600** mean only the owner can use the folder (700) or
  read and write the file (600); SSH refuses keys whose permissions are looser.

* * *

## Step 1: Install and enable the SSH server

Install the server package for your distribution (see the cheat-sheet), then
enable and start it:

```bash
# Fedora, RHEL, and CachyOS
sudo systemctl enable --now sshd

# Ubuntu
sudo systemctl enable --now ssh
```

* * *

## Step 2: Allow SSH through the firewall (before anything else)

Do this **before** changing SSH config or enabling a firewall on a remote host,
or you can cut yourself off.

```bash
# Fedora and RHEL (firewalld)
sudo firewall-cmd --add-service=ssh --permanent
sudo firewall-cmd --reload

# Ubuntu (ufw) -- ufw is OFF by default; add the rule, then enable it
sudo ufw allow OpenSSH
sudo ufw enable

# CachyOS (ufw, already enabled by default)
sudo ufw allow ssh
```

* * *

## Step 3: Set up key-based authentication

This is the core host control. Do it carefully and in order.

**1. Generate a key pair on the machine you connect FROM** (not the server). An
Ed25519 key is the right choice in 2026: fast and secure by modern standards, and
it is `ssh-keygen`'s default.

```bash
ssh-keygen -t ed25519 -C "you@your-laptop-2026"
```

The `-C` text is just a label to help you recognize this key later; it need not
be a real email and does not affect security. Accept the default file location
and **set a passphrase** when prompted. The passphrase encrypts the private key
on disk, so a stolen laptop does not equal a stolen key. NISTIR 7966 treats SSH keys as authenticators (NIST control IA-5):
protect them with a passphrase, do not copy or share private keys, and rotate
them on a defined schedule and on suspected compromise.

> Use RSA only for old systems that predate Ed25519 (OpenSSH older than 6.5):
> `ssh-keygen -t rsa -b 4096`. Avoid ECDSA for new keys. For high-value admin
> accounts, a hardware-backed FIDO2 key (`ssh-keygen -t ed25519-sk`, requires a
> security token) keeps the private key on the token where it cannot be copied,
> and is a genuine second factor.

**2. Copy the public key to the server:**

```bash
ssh-copy-id you@server
```

**3. Test the key login in a NEW terminal, before changing any server config:**

```bash
ssh you@server
```

You should be logged in **without being asked for your account password**. If it
still asks for the server password, fix the key before going further.
Permissions are the usual culprit: on the server, `~/.ssh` must be `700` and
`~/.ssh/authorized_keys` must be `600`.

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Do not continue until a key-only login works.

* * *

## Step 4: Create an SSH access group

The hardened config below restricts SSH to members of a group (CIS requires SSH
access to be limited; this maps to NIST least privilege, AC-6) and disables root
login. Set up the group and confirm your admin user is in it **before** applying
the config, or you will lock yourself out.

```bash
sudo groupadd -f ssh-users
sudo usermod -aG ssh-users you
id you            # confirm 'ssh-users' is listed
```

> Every account that needs SSH access must be in `ssh-users`. Because root login
> will be disabled, make sure this non-root user has working key login (Step 3)
> and `sudo` access. `id you` confirms the group was added to `/etc/group`; a
> brand-new SSH login is checked against the updated group immediately, but log
> out and back in so your interactive session's own credentials also carry the
> new group (for `sudo` and file access). Re-logging in now is safe because the
> lock-you-out settings are not applied until Step 5; keep that fresh session open
> as your safety net through Step 5.

* * *

## Step 5: Harden the sshd configuration (CIS / NIST)

Put your changes in a drop-in under `/etc/ssh/sshd_config.d/`, which all of these
distributions read. Files there are read in alphabetical order, and for any
setting the **first** value wins, so a low-numbered file takes precedence over
distribution defaults (on Ubuntu, over `50-cloud-init.conf` and
`60-cloudimg-settings.conf`).

```bash
sudo nano /etc/ssh/sshd_config.d/10-hardening.conf
```

```ini
# Authentication: keys only
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitEmptyPasswords no
HostbasedAuthentication no
IgnoreRhosts yes
PermitRootLogin no
UsePAM yes

# Restrict who may log in
AllowGroups ssh-users

# Brute-force and session limits
MaxAuthTries 4
LoginGraceTime 60
MaxStartups 10:30:60
MaxSessions 4
ClientAliveInterval 300
ClientAliveCountMax 3

# Reduce attack surface. DisableForwarding is the umbrella; the explicit lines
# below also cover OpenSSH < 10.0 (such as RHEL 9), where DisableForwarding alone
# does not fully block X11 and agent forwarding (CVE-2025-32728). Comment out
# this whole block if you use SSH tunnels or agent forwarding.
PermitUserEnvironment no
DisableForwarding yes
X11Forwarding no
AllowAgentForwarding no
AllowTcpForwarding no

# Logging and banner
LogLevel VERBOSE
Banner /etc/issue.net
```

What the less obvious lines do, and the caveats:

- **`PasswordAuthentication no` / `KbdInteractiveAuthentication no`** is the pair
  that actually stops password brute force. `KbdInteractiveAuthentication` is the
  current name for what older guides call `ChallengeResponseAuthentication`, now
  a deprecated alias.
- **`PermitRootLogin no`** disables direct root SSH entirely (CIS 5.2.10). Log in
  as your `ssh-users` member and use `sudo`. If root key access is genuinely
  required (some single-account VPS), use `prohibit-password` instead, which
  still blocks root password logins.
- **`AllowGroups ssh-users`** limits SSH to that group only. Confirm Step 4 first.
- **`MaxAuthTries 4`**, **`LoginGraceTime 60`**, **`MaxStartups 10:30:60`**, and
  **`MaxSessions 4`** are the CIS values. Raise `MaxSessions` toward 10 if you
  rely on SSH session multiplexing.
- **`ClientAliveInterval 300` / `ClientAliveCountMax 3`** drop unresponsive
  connections after about 15 minutes (300s x 3 = 900s; NIST AC-12, session
  termination).
  CIS editions disagree on the exact numbers: older editions use a 300-second
  interval with count 0, while v2.0.0 and later use a 15-**second** interval with
  count 3. The 300/3 here is an operational middle ground that matches neither
  exactly, so substitute the precise pair your target benchmark requires. These
  probes drop genuinely dead links only; an idle but connected session stays,
  because the client answers the probes automatically.
- **`DisableForwarding yes`** (a CIS Level 2 recommendation, available since
  OpenSSH 7.4) turns off SSH forwarding. On OpenSSH **before 10.0** this directive
  does not fully block X11 and agent forwarding (CVE-2025-32728). RHEL 9 (8.7) is
  affected; RHEL 10 ships 9.9p1 but carries the backported fix
  (`openssh-9.9p1-11.el10`), and Fedora, Ubuntu, and CachyOS are all on 10.x. The
  explicit `X11Forwarding no`, `AllowAgentForwarding no`, and `AllowTcpForwarding
  no` lines are included so forwarding is off everywhere regardless. **Comment out the whole forwarding block
  if you use SSH tunnels (`ssh -L`, `-D`), agent forwarding to a jump host, or
  tools that depend on forwarding.**
- **`LogLevel VERBOSE`** also logs the key fingerprint used to
  authenticate, which NISTIR 7966 calls for and which supports audit (NIST
  AU-3).
- **Do not add a custom `Ciphers`, `MACs`, or `KexAlgorithms` list here.** See
  [Step 6](#step-6-cryptographic-algorithms); on Fedora/RHEL such lines are
  ignored, and modern OpenSSH defaults are already strong.
- There is **no `Protocol 2` line**. SSH protocol 1 and its associated
  configuration options, including `Protocol`, were removed in OpenSSH 7.6
  (2017).

**Create the warning banner** that `Banner` points to (CIS 5.2.19). The default
text below is fine for personal use; replace it with your organization's approved
notice if you have one. The point is only that a warning is shown before login:

```bash
sudo tee /etc/issue.net >/dev/null <<'EOF'
WARNING: Authorized access only. All activity may be monitored and recorded.
Disconnect now if you are not an authorized user.
EOF
```

**Tighten config file permissions** (CIS 5.2.1):

```bash
sudo chmod 600 /etc/ssh/sshd_config /etc/ssh/sshd_config.d/10-hardening.conf
```

**Validate the configuration before applying it:**

```bash
sudo sshd -t
```

If the command shows no output and returns you to the prompt, the config is
valid. If you instead get `sshd: command not found`, run it with the full path:
`sudo /usr/sbin/sshd -t`. A syntax error here does not affect the running service
or your current session.

**Apply it:**

```bash
# Fedora, RHEL, and CachyOS (persistent daemon)
sudo systemctl reload sshd
```

On **Ubuntu**, `sshd` is socket-activated and starts fresh for each new
connection, so once `sudo sshd -t` passes, the new settings apply to the next
login automatically. If you have disabled socket activation (or are on an older
release where `ssh.service` is the listener), `sudo systemctl restart ssh` is a
harmless way to be sure.

**Test from a second terminal before closing your current session:**

```bash
ssh you@server
sudo sshd -T | grep -Ei 'passwordauthentication|permitrootlogin|allowgroups|maxauthtries'
```

The second command prints the server's effective settings for those keywords;
seeing lines such as `passwordauthentication no` and `permitrootlogin no` is
success, not an error. Only close your original session once a fresh key-only
login as an `ssh-users` member works.

* * *

## Step 6: Cryptographic algorithms

In plain terms, a *cipher* encrypts your traffic, a *MAC* checks the traffic was
not tampered with, and *key exchange* is how the two ends agree on a shared
secret. Modern OpenSSH (8.x and newer, so every distribution here) already picks
strong ones by default and removes weak algorithms each release. A hardcoded list
copied from an older guide (including older CIS algorithm lists) tends to
**remove** newer, stronger algorithms. The short version: on Ubuntu and CachyOS,
leave the defaults alone; on Fedora and RHEL, set the system crypto policy below;
touch FIPS only if a regulation requires it. The full approach by distribution:

**Fedora and RHEL: use system-wide crypto policies, not `sshd_config`.** On these
systems `sshd` follows `/etc/crypto-policies`, and `Ciphers`/`MACs`/
`KexAlgorithms` placed in `sshd_config` are **ignored**. Set a policy instead:

```bash
sudo update-crypto-policies --set FUTURE   # stricter set; DEFAULT is the baseline
sudo systemctl restart sshd
```

`FUTURE` is a forward-looking, quantum-resistant-leaning policy and can reject
older clients, so test connectivity. For federal or regulated environments that
require FIPS 140-3 validated cryptography (NIST SC-13), enable FIPS mode, which
constrains SSH (and the rest of the system) to approved algorithms. The method
depends on the version:

- **RHEL 9 and Fedora:** `sudo fips-mode-setup --enable`, then reboot. On RHEL
  this is deprecated since 9.5 but still works for a post-install switch.
- **RHEL 10:** `fips-mode-setup` has been removed and switching to FIPS mode
  after installation is **not supported**. FIPS mode must be enabled at install
  time with the `fips=1` kernel argument.

Setting the FIPS crypto policy on its own is not the same as enabling FIPS mode,
and enabling it after install does not by itself guarantee validated compliance.
Red Hat recommends installing in FIPS mode, and on RHEL 10 that is the only
supported path.

**Ubuntu and CachyOS: rely on the OpenSSH defaults.** They are current and
strong. If you must, only subtract a specific weak algorithm rather than pinning
a whole list, for example `MACs -hmac-sha1`.

* * *

## Step 7 (optional): Change the SSH port

A non-standard port only reduces automated scan noise; it is not a security
control, and an overlay network (Step 9) is a far better way to hide SSH. If you
want a port change anyway, it touches the firewall, SELinux (Fedora/RHEL), and
the SSH config, in that order. This example uses `2222`.

**1. Open the new port (leave port 22 open for now):**

```bash
# Fedora and RHEL (firewalld)
sudo firewall-cmd --add-port=2222/tcp --permanent
sudo firewall-cmd --reload

# Ubuntu and CachyOS (ufw)
sudo ufw allow 2222/tcp
```

**2. On Fedora and RHEL, label the port for SELinux** (enforcing by default), or
`sshd` cannot bind it:

```bash
sudo semanage port -a -t ssh_port_t -p tcp 2222
```

If `semanage` is missing: `sudo dnf install policycoreutils-python-utils`.
Ubuntu and CachyOS do not need this.

**3. Set the port in your drop-in and validate.** The first command appends a
`Port 2222` line to your hardening file (`tee -a` means "append"); you could
instead open the file in `nano` and add the line by hand.

```bash
echo 'Port 2222' | sudo tee -a /etc/ssh/sshd_config.d/10-hardening.conf
sudo sshd -t
```

**4. Apply the port change:**

```bash
# Fedora, RHEL, and CachyOS
sudo systemctl restart sshd

# Ubuntu (the socket owns the listening port)
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

**5. Test the new port from a second terminal, then close port 22:**

```bash
ssh -p 2222 you@server
# once confirmed:
sudo firewall-cmd --remove-service=ssh --permanent && sudo firewall-cmd --reload   # Fedora/RHEL
sudo ufw delete allow OpenSSH    # Ubuntu
sudo ufw delete allow ssh        # CachyOS
```

Use `-p 2222` from now on, and set `port = 2222` in the fail2ban jail below.

* * *

## Step 8: Install and configure fail2ban

fail2ban watches the auth log and bans IPs that fail repeatedly, which reduces
brute-force log noise and load. It implements part of NIST AC-7 (unsuccessful
logon attempts); durable account lockout is better handled by PAM `pam_faillock`.

**Install it.**

```bash
# Fedora
sudo dnf install fail2ban

# Ubuntu
sudo apt install fail2ban python3-systemd

# CachyOS
sudo pacman -S fail2ban
```

On **RHEL**, fail2ban is not in the base or AppStream repositories. It comes from
**EPEL**. On genuine RHEL 10 (use the matching version number for RHEL 9):

```bash
sudo subscription-manager repos --enable codeready-builder-for-rhel-10-$(arch)-rpms
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-10.noarch.rpm
sudo dnf install fail2ban
```

On Rocky Linux, AlmaLinux, or CentOS Stream, the EPEL step is simpler:
`sudo dnf install epel-release` (then enable CRB with
`sudo dnf config-manager --set-enabled crb`), then `sudo dnf install fail2ban`.

**Configure it.** Put settings in `/etc/fail2ban/jail.local`. Never edit
`jail.conf`, which is overwritten on upgrade.

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
# Read auth events from the systemd journal (works on all of these distros).
backend  = systemd

# Never ban these. Add the IP you connect from so a few fumbled logins never
# lock you out (find it by running 'curl ifconfig.me' on your laptop). You can
# list single IPs or whole ranges like 192.168.1.0/24. Skip your own IP if it
# changes often, or you could end up trusting someone else's address later.
ignoreip = 127.0.0.1/8 ::1

# 4 failures within 10 minutes earns a 1 hour ban (matches MaxAuthTries 4).
maxretry = 4
findtime = 10m
bantime  = 1h

[sshd]
enabled = true
# If you changed the SSH port in Step 7, set it here, e.g. port = 2222
port    = ssh
```

**Match the ban action to your firewall.** Add one line under `[DEFAULT]`:

- **Fedora and RHEL (firewalld):** `banaction = firewallcmd-rich-rules` (the
  `fail2ban-firewalld` subpackage is pulled in automatically; this is its current
  default action).
- **Ubuntu and CachyOS (ufw):** `banaction = ufw`.

**Enable and verify.**

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd
```

> **Ubuntu note:** the stock filter usually still matches Ubuntu because it also
> keys on the `sshd` process name, not just the unit. If the jail starts and
> never records failures even though the log shows failed logins, pin it to the
> right unit by adding to `[sshd]`:
> `journalmatch = _SYSTEMD_UNIT=ssh.service + _COMM=sshd`.

* * *

## Step 9: Keep SSH off the public internet (overlay networks)

This is the most effective control for a server, and it maps directly to NIST
boundary protection (SC-7), least functionality (CM-7), and routing remote
access through managed access points (AC-17(3)). A publicly reachable SSH port is
scanned and attacked continuously. If the port is not reachable from the public
internet at all, none of that traffic can touch it.

The modern way to do this without a traditional VPN appliance is an **identity-
based overlay network**: every device gets a private address on an encrypted
mesh, and only enrolled, authenticated devices can reach each other. You then
firewall SSH so it is reachable only over that overlay, and close the public
port. Two good options:

**Tailscale** creates a small private network of only your own devices, encrypted
end to end, that works even when they sit behind home or office routers.
Technically it is a mesh VPN built on WireGuard. Each device joins your private
network (a "tailnet") and gets a stable private `100.x` address that only tailnet
members can reach, with access between devices governed by simple rules (ACLs,
access control lists). It also offers **Tailscale SSH**,
where Tailscale itself handles SSH authentication and authorization within the
tailnet using device identity and policy rules, including a "check mode" that can
force re-authentication (with MFA) before high-risk connections such as
connecting as root.

To restrict a normal `sshd` to the tailnet and remove public SSH, get onto the
tailnet first and confirm a tailnet login works **before** you remove the public
rule, or you will lock yourself out.

**1. Install Tailscale and bring the node onto the tailnet.** Follow the one-line
installer for your distribution at
[tailscale.com/download](https://tailscale.com/download), then:

```bash
sudo tailscale up      # prints a URL; open it to sign in and authorize this device
tailscale ip -4        # shows this node's private 100.x address
```

**2. From a SECOND terminal, confirm SSH works over the tailnet** (replace the
address with the one printed above):

```bash
ssh you@100.x.y.z
```

Do not proceed until this succeeds.

**3. Allow the Tailscale interface and deny other inbound by default.** The public
SSH rule is still in place, so your current session stays up. `tailscale0` is the
virtual network interface Tailscale creates; allowing it permits SSH over the
tailnet only.

```bash
# Ubuntu and CachyOS (ufw)
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0
sudo ufw reload

# Fedora and RHEL (firewalld): put the overlay interface in the trusted zone
sudo firewall-cmd --permanent --zone=trusted --change-interface=tailscale0
sudo firewall-cmd --reload
```

> On Fedora and RHEL the `--permanent` change only takes effect after the
> `firewall-cmd --reload` above (the ufw rules apply immediately). Confirm with
> `sudo firewall-cmd --zone=trusted --list-interfaces`. `tailscale0` is created by
> the Tailscale daemon and is normally not managed by NetworkManager, so this
> binding survives reboots; if your NetworkManager does manage the interface, also
> pin the zone so a reconnect cannot reset it to `public`:
> `sudo nmcli connection modify <connection> connection.zone trusted`.

**4. Only now remove the public SSH rule** (after step 2's tailnet login worked):

```bash
# Ubuntu (rule named OpenSSH) or CachyOS (rule named ssh)
sudo ufw delete allow OpenSSH     # CachyOS: sudo ufw delete allow ssh
sudo ufw reload

# Fedora and RHEL (firewalld): drop ssh from the public zone
sudo firewall-cmd --permanent --zone=public --remove-service=ssh
sudo firewall-cmd --reload
```

After this, SSH to the server's public IP times out, while SSH to its `100.x`
Tailscale address still works for authenticated tailnet users.

**Defined Networking** (defined.net) is a managed control plane for **Nebula**,
an open-source overlay network created at Slack. Nebula uses a certificate-
authority model: a CA signs a host certificate for each node, nodes mutually
authenticate by validating certificates, and well-known "lighthouse" nodes let
peers discover each other and punch through NAT without opening inbound firewall
ports. Nebula also has a built-in, group-based host firewall. Defined Networking
adds a web dashboard, automatic certificate and key distribution and rotation,
and a host agent (DNClient), so you do not have to run the CA and distribute
certificates by hand. (A *lighthouse* is an always-reachable helper node that
lets your other nodes find each other through home or office routers; a
*certificate authority* is the trusted signer that issues each node its identity.)
As with Tailscale, you bind SSH to the overlay and close the public port: use the
same firewall steps as above, substituting Nebula's interface (typically
`nebula1`) for `tailscale0`, and keep the same order, confirming an overlay login
works before you remove the public rule.

**Why this is better than a publicly exposed management port.** Identity-based
overlays move SSH onto a private, mutually authenticated network and let you take
the public port to zero exposure. Unauthenticated internet hosts then have
nothing to connect to, so the public-side attack surface for SSH effectively
disappears. This is the boundary-protection and least-functionality principle:
do not publish a management interface to the whole internet.

**Is fail2ban still needed?** It depends on exposure:

- **SSH reachable only over the overlay (public port closed):** fail2ban adds
  little. With no public-facing port, there are no internet brute-force attempts
  for it to observe or ban. You can keep it as defense-in-depth, but it is no
  longer doing meaningful work.
- **SSH still publicly reachable** (a bastion you have not migrated, a partial
  rollout, or a host that must accept public SSH): keep fail2ban. It is genuinely
  useful there, and it remains reasonable on a jump host as a second layer.

In short, fail2ban matters most exactly where Step 9 has not been applied. The
overlay reduces the problem fail2ban solves rather than complementing it.

**Honest caveats.** An overlay adds a control plane you must trust and keep
available: Tailscale's coordination server (or a self-hosted Headscale), or
Defined Networking's service and your Nebula CA private key, become part of your
trust boundary. It does **not** replace host hardening: an authorized or
compromised overlay peer can still reach `sshd`, so keep the key-only auth and
the hardening from Steps 3 to 6. And Tailscale SSH is not the same as OpenSSH: it
authenticates by node identity rather than SSH keys, claims port 22 on the
Tailscale address, cannot use an alternate port, and ties SSH authorization to
Tailscale's policy engine.

* * *

## Optional: multi-factor authentication

For higher-assurance servers, add a second factor (NIST IA-2(1)):

- **TOTP via PAM:** install a PAM TOTP module (for example
  `libpam-google-authenticator` on Ubuntu, `google-authenticator` on
  Fedora/RHEL), enable it in `/etc/pam.d/sshd`, and require both factors with
  `AuthenticationMethods publickey,keyboard-interactive:pam` in your
  `10-hardening.conf`. This forces an SSH key **and** a one-time code.
  **Important:** Step 5 set `KbdInteractiveAuthentication no`; for TOTP you must
  change it to `yes` in the drop-in, or the code prompt cannot appear. Run the
  authenticator setup once per user to enroll, validate with `sudo sshd -t`, and
  test in a second terminal.
- **Hardware FIDO2 keys:** `ssh-keygen -t ed25519-sk` creates a key bound to a
  security token; the private key cannot leave the hardware and a physical touch
  is required. This is a strong, low-friction second factor. It needs OpenSSH
  8.2 or newer (all distributions here qualify) and a FIDO2 token that supports
  Ed25519; for older tokens use `ecdsa-sk` instead.

Configure MFA with a second session open, the same as any auth change.

* * *

## Verification summary

```bash
# Key-only login as an ssh-users member works (no account-password prompt)
ssh you@server

# Effective sshd settings match intent
sudo sshd -T | grep -Ei 'passwordauthentication|permitrootlogin|allowgroups|maxauthtries|loglevel'

# fail2ban is guarding sshd
sudo fail2ban-client status sshd

# (Fedora/RHEL) crypto policy in effect
update-crypto-policies --show
```

* * *

## CIS and NIST alignment

The settings above map to the SSH-server section of the CIS Linux benchmarks and
to NIST controls. Exact CIS control numbers come from the CIS Distribution
Independent Linux Benchmark; numbering and a few values vary between benchmark
versions and distributions, so verify against the specific benchmark you must
meet.

| Setting / action | CIS Benchmark | NIST SP 800-53 Rev 5 / NISTIR 7966 |
| --- | --- | --- |
| Key-only auth (`PasswordAuthentication no`, key login) | Recommended (site policy) | IA-2, IA-5; NISTIR 7966 key management |
| `PermitRootLogin no` | 5.2.10 | AC-6, CM-6 |
| `PermitEmptyPasswords no` | 5.2.11 | IA-5 |
| `HostbasedAuthentication no` | 5.2.9 | IA-2 |
| `IgnoreRhosts yes` | 5.2.8 | AC-6 |
| `AllowGroups` access limit | 5.2.18 | AC-6 (least privilege) |
| `MaxAuthTries 4` | 5.2.7 | AC-7 (with PAM/fail2ban) |
| `LoginGraceTime 60` | 5.2.17 | AC-12 |
| `MaxStartups 10:30:60` | 5.2.22 | SC-5 (resource exhaustion) |
| `MaxSessions 4` | 5.2.23 | AC-10 |
| `ClientAliveInterval`/`CountMax` | 5.2.16 | AC-12 (session termination) |
| `PermitUserEnvironment no` | 5.2.12 | CM-6 |
| Disable forwarding (`DisableForwarding` + X11/Agent/Tcp) | 5.2.6, 5.2.21 | AC-4, CM-7 |
| `LogLevel VERBOSE`, key-fingerprint logging | 5.2.5 | AU-2, AU-3, AU-12 |
| `Banner /etc/issue.net` | 5.2.19 | AC-8 (system use notification) |
| `UsePAM yes` | 5.2.20 | IA-2 |
| Strong ciphers / crypto-policies / FIPS | 5.2.13 to 5.2.15 (and crypto policy) | SC-8, SC-13; FIPS 140-3 |
| `sshd_config` mode 600, host-key perms | 5.2.1 to 5.2.3 | CM-6, AC-6 |
| Keep SSH off the public internet (overlay) | site policy / network hardening | SC-7, CM-7, AC-17(3) |
| fail2ban / brute-force response | site policy | AC-7 |
| Key passphrase, rotation, no sharing | site policy | IA-5; NISTIR 7966 |

NISTIR 7966 is NIST's dedicated SSH publication; it focuses on SSH access and key
management (provisioning, rotation, source and command restrictions on automation
keys, and logging key fingerprints) rather than line-by-line `sshd` hardening,
which it places out of scope.

* * *

## Troubleshooting

- **Locked out of a remote server.** Use the provider's serial or rescue console
  to edit `/etc/ssh/sshd_config.d/10-hardening.conf`, re-enable
  `PasswordAuthentication` temporarily, or fix the group/key, and reload `sshd`.
  This is why you keep a session open and test from a second terminal.
- **Locked out by `AllowGroups` or `PermitRootLogin no`.** You are not in
  `ssh-users`, or root was your only account. Recover via console, add your user
  to the group, and confirm before re-applying.
- **fail2ban banned your own IP.** Unban and add it to `ignoreip`:
  `sudo fail2ban-client set sshd unbanip <your-ip>`.
- **`fail2ban-client status sshd` reports a missing log file.** The jail is using
  the file backend on a journal-only system. Confirm `backend = systemd` is set,
  and on Ubuntu that `python3-systemd` is installed.
- **Cipher hardening in `sshd_config` seems ignored on Fedora/RHEL.** That is
  expected. Use `update-crypto-policies` (Step 6), not `sshd_config`.
- **Port change on Ubuntu does not take effect.** You restarted `ssh.service`
  instead of `ssh.socket`. Run
  `sudo systemctl daemon-reload && sudo systemctl restart ssh.socket`.
- **`sshd` will not start on a new port (Fedora/RHEL).** SELinux is blocking the
  bind. Add the label: `sudo semanage port -a -t ssh_port_t -p tcp <port>`.

* * *

## How to roll back

```bash
# Undo the SSH hardening
sudo rm /etc/ssh/sshd_config.d/10-hardening.conf
sudo sshd -t && sudo systemctl reload sshd     # Fedora, RHEL, CachyOS
# On Ubuntu, removing the drop-in applies on the next login (socket-activated);
# if needed: sudo systemctl daemon-reload && sudo systemctl restart ssh.socket

# Stop and disable fail2ban
sudo systemctl disable --now fail2ban
```

> **If you changed the SSH port (Step 7), reverting needs care to avoid a
> lockout.** Removing the drop-in restores `Port 22` in the config, but a running
> daemon keeps listening on the old port until it is fully restarted, not just
> reloaded. Do it in this order:
>
> 1. Remove the drop-in, then **restart** (not reload) so `sshd` rebinds to 22:
>    `sudo sshd -t && sudo systemctl restart sshd` (on Ubuntu:
>    `sudo systemctl daemon-reload && sudo systemctl restart ssh.socket`).
> 2. Re-open port 22 in the firewall and confirm a login on 22 works.
> 3. Only then remove the custom-port firewall rule and, on Fedora/RHEL, the
>    SELinux label: `sudo semanage port -d -t ssh_port_t -p tcp <port>`.

* * *

## What you achieved

- SSH accepts **keys only**, from members of a defined group, with **no root
  login**, so password brute force cannot succeed and access is least-privilege.
- The login window, attempt count, and forwarding are tightened to CIS values,
  and authentication is logged in detail for audit.
- Cryptography is left to strong, self-updating defaults (or system crypto
  policies and FIPS mode on Fedora/RHEL).
- **fail2ban** bans repeat offenders where SSH is still publicly reachable.
- The strongest option is to take SSH **off the public internet** behind an
  identity-based overlay, which removes the public attack surface entirely.

* * *

## References

- [OpenSSH `sshd_config(5)` manual](https://man.openbsd.org/sshd_config.5)
- [OpenSSH `ssh-keygen(1)` manual](https://man.openbsd.org/ssh-keygen.1)
- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks)
- [NIST SP 800-53 Rev 5](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final)
- [NISTIR 7966: Security of Interactive and Automated Access Management Using SSH](https://csrc.nist.gov/pubs/ir/7966/final)
- [NIST SP 800-123: Guide to General Server Security](https://csrc.nist.gov/pubs/sp/800/123/final)
- [FIPS 140-3](https://csrc.nist.gov/pubs/fips/140-3/final)
- [Red Hat: Using system-wide cryptographic policies](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/security_hardening/using-the-system-wide-cryptographic-policies_security-hardening)
- [EPEL (Extra Packages for Enterprise Linux)](https://docs.fedoraproject.org/en-US/epel/)
- [fail2ban documentation](https://github.com/fail2ban/fail2ban)
- [Ubuntu: socket-based sshd activation](https://discourse.ubuntu.com/t/sshd-now-uses-socket-based-activation-ubuntu-22-10-and-later/30189)
- [Tailscale SSH](https://tailscale.com/kb/1193/tailscale-ssh)
- [Defined Networking (managed Nebula)](https://www.defined.net/) and [Nebula](https://github.com/slackhq/nebula)
