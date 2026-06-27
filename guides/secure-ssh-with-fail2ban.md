# How to Secure SSH with Key-Based Authentication and fail2ban

> **Last verified:** 2026-06-27 with OpenSSH 10.3 and fail2ban 1.1.0, on Fedora
> 44, Ubuntu 26.04 LTS, and CachyOS. Where the three distributions differ, the
> command for each is given.

> Any machine with SSH exposed to the internet is hit constantly by automated
> bots trying to guess passwords on port 22. The single most effective fix is to
> turn off password authentication entirely and log in with keys, so there is no
> password to guess. This guide does that first, then adds defense-in-depth:
> sensible `sshd` settings and **fail2ban** to drop repeat offenders at the
> firewall.

The order of priority matters, so be honest about what each part buys you:

- **Key-only authentication is the real protection.** With
  `PasswordAuthentication no`, brute-force password guessing cannot succeed,
  full stop.
- **fail2ban is noise and load reduction, not your primary defense.** Once
  passwords are off, bots cannot get in regardless. fail2ban still helps by
  banning their IPs after a few tries, which cuts log spam and connection load.
- **A non-standard port is obscurity, not security.** It reduces automated scan
  noise but stops no determined attacker. It is optional and covered last.

> **Lockout warning:** every step here can lock you out of a remote machine if
> done out of order. Keep your current SSH session open the entire time, test
> changes from a second terminal before closing it, and on a cloud VM know where
> your provider's serial or rescue console is before you start.

* * *

## Distribution cheat-sheet

The procedure is the same everywhere; only a few commands differ. Substitute from
this table as you go.

| | Fedora 44 | Ubuntu 26.04 | CachyOS (Arch-based) |
| --- | --- | --- | --- |
| Install SSH server | `sudo dnf install openssh-server` | `sudo apt install openssh-server` | `sudo pacman -S openssh` |
| systemd unit | `sshd` | `ssh` (alias `sshd` also works) | `sshd` |
| Firewall | firewalld (on by default) | ufw (installed, **off** by default) | ufw (**on** by default) |
| Mandatory access control | SELinux, enforcing | AppArmor (no enforced `sshd` profile) | none by default |
| Install fail2ban | `sudo dnf install fail2ban` | `sudo apt install fail2ban python3-systemd` | `sudo pacman -S fail2ban` |

A few cross-distro facts worth stating once:

- On **Ubuntu/Debian** the service is `ssh.service`, not `sshd.service`. A
  compatibility alias means `systemctl ... sshd` usually works too, but the real
  unit is `ssh`.
- On Ubuntu, `sshd` is **socket-activated** (`ssh.socket`): a fresh `sshd` starts
  for each connection rather than running as a persistent daemon. This changes
  how reloads and port changes work, noted where it matters.
- The OpenSSH client and server are **one package (`openssh`) on Arch/CachyOS**,
  but **split** on Fedora and Ubuntu (`openssh-server`).

* * *

## Prerequisites

- A user account on the server that can use `sudo`.
- Terminal access to both the server and the machine you will connect from.
- The OpenSSH **client** on the machine you connect from (`ssh` and `ssh-keygen`,
  preinstalled on most desktops).

* * *

## Step 1: Install and enable the SSH server

Install the server package for your distribution (see the cheat-sheet), then
enable and start it:

```bash
# Fedora and CachyOS
sudo systemctl enable --now sshd

# Ubuntu
sudo systemctl enable --now ssh
```

Confirm it is listening:

```bash
systemctl status sshd    # use 'ssh' on Ubuntu
```

* * *

## Step 2: Allow SSH through the firewall (before anything else)

Do this **before** changing SSH config or enabling a firewall on a remote host,
or you can cut yourself off.

```bash
# Fedora (firewalld)
sudo firewall-cmd --add-service=ssh --permanent
sudo firewall-cmd --reload

# Ubuntu (ufw) -- ufw is OFF by default; add the rule, then enable it
sudo ufw allow OpenSSH
sudo ufw enable

# CachyOS (ufw, already enabled by default)
sudo ufw allow ssh
```

> On Fedora, the active zone matters. `firewall-cmd --get-active-zones` shows it;
> the Workstation zone already permits SSH, while Server/minimal installs use the
> `public` zone where the rule above is needed.

* * *

## Step 3: Set up key-based authentication

This is the core of the whole guide. Do it carefully and in order.

**1. Generate a key pair on the machine you connect FROM** (not the server). An
Ed25519 key is the right choice in 2026: fast and secure by modern standards,
and it is `ssh-keygen`'s default.

```bash
ssh-keygen -t ed25519 -C "you@your-laptop-2026"
```

Accept the default file location and **set a passphrase** when prompted. The
passphrase encrypts the private key on disk, so a stolen laptop does not equal a
stolen key.

> Use RSA only for old systems that predate Ed25519 (OpenSSH older than 6.5):
> `ssh-keygen -t rsa -b 4096`. Avoid ECDSA for new keys. For high-value admin
> accounts, a hardware-backed FIDO2 key (`ssh-keygen -t ed25519-sk`, requires a
> security token) keeps the private key on the token where it cannot be copied.

**2. Copy the public key to the server:**

```bash
ssh-copy-id you@server
```

This logs in with your existing password one last time and appends your public
key to `~/.ssh/authorized_keys` on the server. If `ssh-copy-id` is unavailable,
copy `~/.ssh/id_ed25519.pub` into the server's `~/.ssh/authorized_keys` manually.

**3. Test the key login in a NEW terminal, before changing any server config:**

```bash
ssh you@server
```

You should be logged in **without being asked for your account password** (the
key passphrase prompt, if any, is on your local machine and is expected). If it
still asks for the server password, stop and fix the key before going further.
Permissions are the usual culprit: on the server, `~/.ssh` must be `700` and
`~/.ssh/authorized_keys` must be `600`.

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Do not continue to Step 4 until a key-only login works.

* * *

## Step 4: Harden the sshd configuration

Rather than edit the shipped `/etc/ssh/sshd_config`, drop your changes into a
file under `/etc/ssh/sshd_config.d/`, which all three distributions read. Files
there are read in alphabetical order, and for any setting the **first** value
wins, so a low-numbered file takes precedence over distribution defaults (for
example Ubuntu's cloud-init drop-in).

```bash
sudo nano /etc/ssh/sshd_config.d/10-hardening.conf
```

Add:

```ini
# --- Authentication: keys only ---
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password

# --- Reduce brute-force and slow-auth exposure ---
MaxAuthTries 3
LoginGraceTime 30

# --- Drop dead sessions after ~10 minutes ---
ClientAliveInterval 300
ClientAliveCountMax 2

# --- Turn off X11 forwarding (re-enable only if you need it) ---
X11Forwarding no

# --- Optional: restrict who may log in over SSH ---
#AllowUsers you
```

What each line does, and why it is written this way:

- **`PasswordAuthentication no`** is the change that actually stops password
  brute force. **`KbdInteractiveAuthentication no`** closes the keyboard-
  interactive path that can otherwise present a password prompt through PAM.
  (`KbdInteractiveAuthentication` is the current name for what older guides call
  `ChallengeResponseAuthentication`, which is now only a deprecated alias.)
  `PubkeyAuthentication yes` is already the default and is stated for clarity.
- **`PermitRootLogin prohibit-password`** is the default, and it blocks password
  and keyboard-interactive root logins while still allowing key-based root. Once
  you have a non-root user with `sudo` and a working key, change this to
  **`PermitRootLogin no`** to disable direct root SSH entirely. Do not set `no`
  while root is your only account with key access, or you will lock yourself out.
- **`MaxAuthTries 3`** (default 6) and **`LoginGraceTime 30`** (default 120s)
  shrink how long and how many attempts an unauthenticated connection gets.
- **`ClientAliveInterval`/`ClientAliveCountMax`** drop genuinely dead connections
  (defaults: interval 0, meaning probes off, and count 3). Active sessions that
  are merely idle stay connected, because the client still answers the server's
  keep-alive probes.
- **Do not add a custom `Ciphers`, `MACs`, or `KexAlgorithms` list.** Modern
  OpenSSH (10.x) already defaults to strong AEAD ciphers, ETM MACs, and
  post-quantum key exchange, and it drops weak algorithms each release. A
  hardcoded list copied from an older guide tends to *remove* newer, stronger
  algorithms. Rely on the defaults; if you must, only subtract a specific weak
  algorithm (for example `MACs -hmac-sha1`).
- There is **no `Protocol 2` line**. SSH protocol 1 and its associated
  configuration options, including `Protocol`, were removed in OpenSSH 7.6
  (2017); setting it does nothing.

**Validate the configuration before applying it:**

```bash
sudo sshd -t
```

This prints nothing and exits 0 if the config is valid. If `sshd` is not on your
`PATH`, use `sudo /usr/sbin/sshd -t`. A syntax error here does not affect the
running service or your current session, so a bad config cannot drop your
connection.

**Apply it:**

```bash
# Fedora and CachyOS (persistent daemon)
sudo systemctl reload sshd
```

On **Ubuntu**, `sshd` is socket-activated and starts fresh for each new
connection, so once `sudo sshd -t` passes, the new settings apply to the next
login automatically. There is no persistent daemon to reload. If you have
disabled socket activation (or are on an older release where `ssh.service` is the
listener), `sudo systemctl restart ssh` is a harmless way to be sure.

**Now test from a second terminal before closing your current session:**

```bash
ssh you@server
```

You can also confirm the settings actually took effect by dumping the resolved
configuration:

```bash
sudo sshd -T | grep -Ei 'passwordauthentication|kbdinteractive|permitrootlogin|maxauthtries'
```

Only close your original session once a fresh key-only login works.

* * *

## Step 5 (optional): Change the SSH port

A non-standard port only reduces automated scan noise; it is not a security
control. If you want it anyway, it touches three things in a specific order:
firewall, then SELinux (Fedora), then the SSH config. Pick a port (this example
uses `2222`).

**1. Open the new port (leave port 22 open for now):**

```bash
# Fedora (firewalld)
sudo firewall-cmd --add-port=2222/tcp --permanent
sudo firewall-cmd --reload

# Ubuntu and CachyOS (ufw)
sudo ufw allow 2222/tcp
```

**2. On Fedora, label the port for SELinux** (which is enforcing by default), or
`sshd` cannot bind it:

```bash
sudo semanage port -a -t ssh_port_t -p tcp 2222
```

If `semanage` is missing: `sudo dnf install policycoreutils-python-utils`.
Ubuntu and CachyOS do not need this (Ubuntu's AppArmor has no enforced `sshd`
profile, and CachyOS uses neither SELinux nor AppArmor by default).

**3. Set the port in your drop-in:**

```bash
echo 'Port 2222' | sudo tee -a /etc/ssh/sshd_config.d/10-hardening.conf
sudo sshd -t
```

**4. Apply the port change:**

```bash
# Fedora and CachyOS
sudo systemctl restart sshd

# Ubuntu (the socket owns the listening port)
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

> On Ubuntu 24.04 and later a systemd generator derives the socket's port from
> `sshd_config`, so setting `Port` there and restarting `ssh.socket` is enough.
> On older socket-activated Ubuntu (22.10 to 23.10) you had to override
> `ListenStream` in `ssh.socket` instead.

**5. Test the new port from a second terminal, then close port 22:**

```bash
ssh -p 2222 you@server
# once confirmed:
sudo firewall-cmd --remove-service=ssh --permanent && sudo firewall-cmd --reload   # Fedora
sudo ufw delete allow OpenSSH    # Ubuntu
sudo ufw delete allow ssh        # CachyOS
```

Remember to use `-p 2222` from now on, and to set `port = 2222` in the fail2ban
jail below.

* * *

## Step 6: Install and configure fail2ban

Install fail2ban for your distribution (see the cheat-sheet). On Ubuntu, also
install `python3-systemd` so fail2ban can read the systemd journal.

Put your settings in `/etc/fail2ban/jail.local`. Never edit `jail.conf`, which is
overwritten on upgrade.

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
# Read auth events from the systemd journal (all three distros use journald).
backend  = systemd

# Never ban these. Add your own static admin IP or LAN so a fat-fingered
# login never locks you out. Space-separated; CIDR allowed.
ignoreip = 127.0.0.1/8 ::1

# Ban policy: 3 failures within 10 minutes earns a 1 hour ban.
maxretry = 3
findtime = 10m
bantime  = 1h

[sshd]
enabled = true
# If you changed the SSH port in Step 5, set it here, e.g. port = 2222
port    = ssh
```

**Firewall integration (`banaction`).** fail2ban must ban using the same firewall
you run:

- **Fedora (firewalld):** the `fail2ban-firewalld` package, pulled in
  automatically, configures this for you. Nothing to add.
- **Ubuntu and CachyOS (ufw):** add `banaction = ufw` under `[DEFAULT]`.

```ini
# Add under [DEFAULT] on Ubuntu and CachyOS:
banaction = ufw
```

**Enable and start it:**

```bash
sudo systemctl enable --now fail2ban
```

**Verify the jail is active and watching SSH:**

```bash
sudo fail2ban-client status sshd
```

You should see the `sshd` jail with its file/journal source and a (probably zero)
ban count. Bans appear here as bots hit the server.

> **Ubuntu note:** the stock filter usually still matches Ubuntu because it also
> keys on the `sshd` process name, not just the unit. But if the jail starts and
> never records failures even though the log shows failed logins, pin it to the
> right unit: Ubuntu's unit is `ssh.service`, so add this to the `[sshd]` section
> and restart fail2ban:
> `journalmatch = _SYSTEMD_UNIT=ssh.service + _COMM=sshd`.

* * *

## Verification summary

```bash
# Key-only login works (no account-password prompt)
ssh you@server

# Effective sshd settings are what you intended
sudo sshd -T | grep -Ei 'passwordauthentication|permitrootlogin|maxauthtries'

# fail2ban is guarding sshd
sudo fail2ban-client status sshd
```

* * *

## Troubleshooting

- **Locked out of a remote server.** Use the provider's serial or rescue console
  (AWS EC2 Serial Console, GCP/Azure serial console, or a rescue boot) to edit
  `/etc/ssh/sshd_config.d/10-hardening.conf`, re-enable `PasswordAuthentication`
  temporarily, and fix the key or firewall. This is why you keep a session open
  and test from a second terminal.
- **fail2ban banned your own IP.** Unban it and add it to `ignoreip`:
  `sudo fail2ban-client set sshd unbanip <your-ip>`.
- **`fail2ban-client status sshd` errors with "Have not found any log file" or
  "No file(s) found".** The jail is using the file backend on a journal-only
  system. Confirm `backend = systemd` is set, and on Ubuntu that
  `python3-systemd` is installed.
- **Changed the port on Ubuntu but it still listens on 22 (or refuses the new
  port).** You restarted `ssh.service` instead of `ssh.socket`. Run
  `sudo systemctl daemon-reload && sudo systemctl restart ssh.socket`.
- **`sshd` will not start on the new port (Fedora).** SELinux is blocking the
  bind. Add the port label: `sudo semanage port -a -t ssh_port_t -p tcp <port>`.
- **A setting in the drop-in seems ignored.** Another file in
  `/etc/ssh/sshd_config.d/` set it first (first value wins). Check with
  `sudo sshd -T` and give your file a lower number so it is read earlier.

* * *

## How to roll back

```bash
# Undo the SSH hardening
sudo rm /etc/ssh/sshd_config.d/10-hardening.conf
sudo sshd -t && sudo systemctl reload sshd     # use 'ssh' on Ubuntu

# Stop and disable fail2ban
sudo systemctl disable --now fail2ban
```

> **If you changed the SSH port (Step 5), reverting needs care to avoid a
> lockout.** Removing the drop-in restores `Port 22` in the config, but a
> running daemon keeps listening on the old port until it is fully restarted, not
> just reloaded. Do it in this order:
>
> 1. Remove the drop-in, then **restart** (not reload) so `sshd` rebinds to 22:
>    `sudo sshd -t && sudo systemctl restart sshd` (on Ubuntu:
>    `sudo systemctl daemon-reload && sudo systemctl restart ssh.socket`).
> 2. Re-open port 22 in the firewall and confirm a login on 22 works.
> 3. Only then remove the custom-port firewall rule and, on Fedora, the SELinux
>    label: `sudo semanage port -d -t ssh_port_t -p tcp <port>`.

* * *

## What you achieved

- SSH accepts **keys only**. Password and keyboard-interactive logins are off, so
  password brute force cannot succeed.
- Root cannot log in with a password, and (optionally) cannot log in over SSH at
  all.
- The login window and attempt count are tightened, and dead sessions are
  dropped.
- Cipher choice is left to OpenSSH's strong, self-updating defaults.
- **fail2ban** bans repeat offenders at your firewall, cutting log noise and load.

* * *

## References

- [OpenSSH `sshd_config(5)` manual](https://man.openbsd.org/sshd_config.5)
- [OpenSSH `ssh-keygen(1)` manual](https://man.openbsd.org/ssh-keygen.1)
- [OpenSSH release notes](https://www.openssh.com/releasenotes.html)
- [fail2ban documentation](https://github.com/fail2ban/fail2ban)
- [ArchWiki: SSH keys](https://wiki.archlinux.org/title/SSH_keys)
- [ArchWiki: fail2ban](https://wiki.archlinux.org/title/Fail2ban)
- [Ubuntu Server docs: OpenSSH server](https://ubuntu.com/server/docs/how-to/security/openssh-server/)
- [Ubuntu: socket-based sshd activation](https://discourse.ubuntu.com/t/sshd-now-uses-socket-based-activation-ubuntu-22-10-and-later/30189)
- [Mozilla Infosec: OpenSSH guidelines](https://infosec.mozilla.org/guidelines/openssh)
