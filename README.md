# 🛡️ Ubuntu VPS Setup & Security Guide

A step-by-step guide to provisioning and hardening a fresh Ubuntu VPS from scratch.

> Tested on **Ubuntu 22.04 LTS** and **Ubuntu 24.04 LTS**.

---

## Table of Contents

1. [Initial Login](#1-initial-login)
2. [Update the System](#2-update-the-system)
3. [Create a Non-Root User with Sudo](#3-create-a-non-root-user-with-sudo)
4. [Configure SSH Key Authentication](#4-configure-ssh-key-authentication)
5. [Harden SSH Configuration](#5-harden-ssh-configuration)
6. [Configure the Firewall (UFW)](#6-configure-the-firewall-ufw)
7. [Secure Shared Memory](#7-secure-shared-memory)
8. [Install Fail2Ban](#8-install-fail2ban)
9. [Enable Automatic Security Updates](#9-enable-automatic-security-updates)
10. [Optional: Change the SSH Port](#10-optional-change-the-ssh-port)
11. [Optional: 2FA for SSH](#11-optional-2fa-for-ssh)
12. [Final Checklist](#12-final-checklist)

---

## 1. Initial Login

When your VPS is first provisioned, you'll receive a root password from your hosting provider. Log in using:

```bash
ssh root@YOUR_SERVER_IP
```

> `ssh` opens a secure encrypted connection to your server. `root` is the default superuser account. Replace `YOUR_SERVER_IP` with the IP address of your VPS.

> **Never leave your server in this default state.** The steps below harden it significantly.

---

## 2. Update the System

Before doing anything else, update all installed packages to their latest versions. This ensures any known vulnerabilities in the default OS image are patched immediately.

```bash
apt update && apt upgrade -y
```

> - `apt update` — Refreshes the local list of available packages from the Ubuntu repositories. It doesn't install anything yet.
> - `apt upgrade -y` — Downloads and installs all available updates. The `-y` flag auto-confirms prompts so you don't have to type "yes" manually.
> - The `&&` means the second command only runs if the first one succeeds.

```bash
apt autoremove -y
```

> Removes packages that were installed as dependencies but are no longer needed by anything. Keeps the system lean and reduces the attack surface.

If a kernel update was applied, reboot so the new kernel is loaded:

```bash
reboot
```

> Restarts the server. Your SSH session will disconnect — wait ~30 seconds, then log back in.

---

## 3. Create a Non-Root User with Sudo

Running as root is dangerous — a single mistake or exploit can compromise the entire system. Always create a regular user for day-to-day work and only use elevated privileges when necessary.

```bash
adduser admin
```

> Creates a new system user account named `admin`. Replace this with your preferred username. You'll be prompted to set a password and fill in optional info (name, phone, etc.) — you can leave the optional fields blank.

```bash
usermod -aG sudo admin
```

> - `usermod` — Modifies an existing user account.
> - `-aG sudo` — **Appends** (`-a`) the user to the **group** (`-G`) called `sudo`. Members of the `sudo` group can run commands as root by prefixing them with `sudo`.
> - Without the `-a` flag, this would *replace* all existing group memberships — a common mistake.

Test the new account by opening a **new terminal tab** and logging in:

```bash
ssh admin@YOUR_SERVER_IP
```

> Confirms the new user account can log in successfully.

```bash
sudo whoami
```

> Runs `whoami` (which prints the current user) with elevated root privileges. If sudo is working correctly, it will output `root`.

---

## 4. Configure SSH Key Authentication

Passwords can be brute-forced. SSH keys use cryptographic key pairs that are practically impossible to guess. This is the most important security step.

### 4a. Generate a Key Pair (on your local machine)

Run this on your **local machine**, not the server:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

> - `ssh-keygen` — The tool for generating SSH key pairs.
> - `-t ed25519` — Specifies the key type. `ed25519` is a modern elliptic-curve algorithm — smaller, faster, and more secure than the older `rsa`.
> - `-C "your_email@example.com"` — Adds a comment to the key, typically your email, so you can identify it later.
> - You'll be asked for a **passphrase** — this encrypts the private key file on disk. Highly recommended.

This creates two files:
- `~/.ssh/id_ed25519` — your **private key**. Never share this with anyone.
- `~/.ssh/id_ed25519.pub` — your **public key**. This is what goes on the server.

### 4b. Copy the Public Key to the Server

The easiest way:

```bash
ssh-copy-id admin@YOUR_SERVER_IP
```

> Automatically appends your local public key (`~/.ssh/id_ed25519.pub`) to `~/.ssh/authorized_keys` on the remote server and sets the correct file permissions. You'll need to enter the user's password one last time.

If `ssh-copy-id` isn't available, do it manually:

```bash
# On your local machine — prints your public key to the terminal
cat ~/.ssh/id_ed25519.pub
```

> `cat` reads and prints the file contents. Copy the entire output (starts with `ssh-ed25519 ...`).

```bash
# On the server — create the .ssh directory with strict permissions
mkdir -p ~/.ssh
```

> - `mkdir` — Makes a directory.
> - `-p` — Creates parent directories as needed and doesn't error if the directory already exists.

```bash
chmod 700 ~/.ssh
```

> Sets permissions on the `.ssh` directory so only the owner can read, write, or enter it. SSH refuses to work if this directory is too permissive.

```bash
nano ~/.ssh/authorized_keys
# Paste the public key, save with Ctrl+O, exit with Ctrl+X
```

> Opens the `authorized_keys` file in the `nano` text editor. SSH checks this file to decide which keys are allowed to log in.

```bash
chmod 600 ~/.ssh/authorized_keys
```

> Sets permissions so only the file owner can read or write it. `600` means no access for group or others. Again, SSH will reject the file if the permissions are too open.

### 4c. Verify Key-Based Login

Open a **new terminal** and confirm you can log in without a password:

```bash
ssh admin@YOUR_SERVER_IP
```

> If key authentication is working, SSH will use your private key to authenticate automatically. No password prompt should appear (unless you set a key passphrase).

> ⚠️ **Do not close your current SSH session** until you've confirmed the new session works. Locking yourself out is a very common mistake at this stage.

---

## 5. Harden SSH Configuration

The SSH daemon (`sshd`) has a configuration file where you can tighten its behaviour significantly.

```bash
sudo nano /etc/ssh/sshd_config
```

> Opens the main SSH server configuration file with root privileges. Changes here affect how SSH accepts connections.

Apply the following settings (find and update existing lines, or add them if missing):

```ini
# Prevents the root account from logging in over SSH directly.
# Even if someone knows the root password, they can't use it remotely.
PermitRootLogin no

# Disables password-based login entirely — only SSH keys are accepted.
PasswordAuthentication no

# Disables keyboard-interactive auth (used by some password-based methods).
ChallengeResponseAuthentication no

# Prevents login with an account that has no password set.
PermitEmptyPasswords no

# Limits the number of authentication attempts per connection.
# After 3 failed tries, the connection is dropped.
MaxAuthTries 3

# Sends a keepalive signal every 600 seconds (10 minutes) of inactivity.
ClientAliveInterval 600

# After 1 missed keepalive with no response, the session is terminated.
# Combined with the above, idle sessions are dropped after 10 minutes.
ClientAliveCountMax 0

# Restricts SSH access to only the listed users.
# Anyone not on this list is denied, even with a valid password or key.
AllowUsers admin

# Disables X11 (graphical) forwarding — not needed on a headless server
# and a potential vector for privilege escalation.
X11Forwarding no

# Forces SSH to only use protocol version 2.
# SSHv1 is outdated and has known vulnerabilities.
Protocol 2
```

Restart the SSH service to apply the changes:

```bash
sudo systemctl restart sshd
```

> `systemctl` manages system services. `restart sshd` stops and starts the SSH daemon, loading the updated config. The existing session stays connected — only new connections use the new config.

> ⚠️ Open a **new terminal** and confirm you can still log in before closing your current session.

---

## 6. Configure the Firewall (UFW)

UFW (Uncomplicated Firewall) provides a simple interface to control which network traffic is allowed in and out of your server.

```bash
sudo apt install ufw -y
```

> Installs UFW if it's not already present. On most Ubuntu installations it comes pre-installed but may not be active.

```bash
sudo ufw default deny incoming
```

> Sets the **default policy** for all inbound traffic to "deny". This means any connection not explicitly allowed by a rule will be blocked. This is the "default closed" approach — secure by default.

```bash
sudo ufw default allow outgoing
```

> Sets the default policy for all outbound traffic to "allow". Your server can freely initiate outgoing connections (e.g., to download packages), but unsolicited inbound connections are blocked.

```bash
# Allow SSH — do this BEFORE enabling UFW or you'll lock yourself out
sudo ufw allow ssh
```

> Creates a rule that allows inbound traffic on port 22 (the default SSH port). **This must be done before enabling the firewall**, otherwise you'll be immediately locked out of your own server.

```bash
# Allow HTTP (port 80) — needed if you're hosting a website
sudo ufw allow http

# Allow HTTPS (port 443) — needed for SSL/TLS encrypted web traffic
sudo ufw allow https
```

> These rules open the standard web server ports. Only add these if you're running a web server. Remove them if you're not.

```bash
sudo ufw enable
```

> Activates the firewall with all the rules you've defined. You'll see a warning that it may disrupt existing SSH connections — type `y` to confirm. (Your SSH session should remain active because you already allowed it.)

```bash
sudo ufw status verbose
```

> Displays all active firewall rules and their status. Useful to verify the rules look correct after enabling UFW.

---

## 7. Secure Shared Memory

`/run/shm` is a shared memory space in RAM that processes can use to communicate with each other quickly. Attackers with limited access to your system can sometimes exploit it to escalate privileges. Mounting it with restrictions closes that vector.

```bash
sudo nano /etc/fstab
```

> Opens the filesystem table — a config file that defines how and where disks and virtual filesystems are mounted at boot.

Add this line at the bottom of the file:

```
tmpfs /run/shm tmpfs defaults,noexec,nosuid,nodev 0 0
```

> - `tmpfs` — A virtual filesystem stored in RAM.
> - `/run/shm` — The mount point (location) for shared memory.
> - `defaults` — Uses standard mount options.
> - `noexec` — Prevents any binaries from being executed from this location.
> - `nosuid` — Prevents setuid/setgid bits from taking effect (blocks privilege escalation tricks).
> - `nodev` — Prevents device files from being used here.
> - The two `0`s at the end disable dump backup and filesystem check for this mount.

Apply the change immediately without rebooting:

```bash
sudo mount -o remount /run/shm
```

> `mount -o remount` re-mounts the filesystem at the given path using the new options from `/etc/fstab`, without needing a reboot.

---

## 8. Install Fail2Ban

Fail2Ban watches log files for patterns indicating brute-force or malicious activity (like repeated failed SSH logins) and automatically bans the offending IP addresses using firewall rules.

```bash
sudo apt install fail2ban -y
```

> Installs the Fail2Ban package from the Ubuntu repositories.

Create a local configuration override so your custom settings survive package upgrades:

```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

> Copies the default config to `jail.local`. Fail2Ban reads `.local` files with higher priority than `.conf` files — this way, package updates can overwrite `jail.conf` without touching your customisations in `jail.local`.

```bash
sudo nano /etc/fail2ban/jail.local
```

> Opens your local config for editing. Find the `[sshd]` section and set it up as shown below.

```ini
[sshd]
enabled  = true        # Activates this jail (monitoring rule)
port     = ssh         # Monitors port 22 (or use your custom port number)
filter   = sshd        # Uses the built-in filter that knows SSH log patterns
logpath  = /var/log/auth.log  # The log file Fail2Ban watches for failures
maxretry = 5           # Ban the IP after 5 failed login attempts
bantime  = 3600        # Keep the IP banned for 3600 seconds (1 hour)
findtime = 600         # Count failures within a 10-minute sliding window
```

Enable and start Fail2Ban:

```bash
sudo systemctl enable fail2ban
```

> Configures Fail2Ban to start automatically every time the server boots.

```bash
sudo systemctl start fail2ban
```

> Starts the Fail2Ban service right now without waiting for a reboot.

Verify it's working and watching SSH:

```bash
sudo fail2ban-client status sshd
```

> Queries the Fail2Ban daemon for the status of the `sshd` jail. Shows how many IPs are currently banned and how many failures have been detected.

---

## 9. Enable Automatic Security Updates

Keeping up with security patches manually is easy to forget. This step configures Ubuntu to automatically download and install security updates in the background.

```bash
sudo apt install unattended-upgrades -y
```

> Installs the `unattended-upgrades` package, which handles automatic updates.

```bash
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

> Launches an interactive dialog to configure the package. When asked if you want to automatically download and install stable updates, select **Yes**. The `--priority=low` flag shows all available configuration questions instead of just the essential ones.

Review or customise the upgrade config:

```bash
sudo nano /etc/apt/apt.conf.d/50unattended-upgrades
```

> Opens the main configuration file for `unattended-upgrades`. Inside, make sure this line is **uncommented** (no `//` at the start):

```
"${distro_id}:${distro_codename}-security";
```

> This tells the system to automatically apply security updates for your specific Ubuntu version. You can also uncomment `-updates` for all general package updates, though that carries a slightly higher risk of breakage.

---

## 10. Optional: Change the SSH Port

By default, every automated bot on the internet scans port 22 for SSH servers. Moving SSH to a non-standard port won't stop a determined attacker, but it dramatically reduces log noise and opportunistic attacks.

```bash
sudo nano /etc/ssh/sshd_config
```

> Opens the SSH configuration file for editing.

Find the `Port` line and change it:

```ini
Port 2222
```

> Replace `2222` with any unused port between 1024 and 65535. Avoid well-known ports like 80, 443, 3306 etc.

**Update the firewall to allow the new port first** (before restarting SSH):

```bash
sudo ufw allow 2222/tcp
```

> Creates a firewall rule to allow inbound TCP traffic on port 2222. The `/tcp` specifies the protocol explicitly. Always do this before restarting SSH, or you'll lock yourself out.

```bash
sudo ufw delete allow ssh
```

> Removes the old rule that allowed traffic on port 22. Once SSH is moved to the new port, port 22 should be blocked.

```bash
sudo ufw reload
```

> Reloads the UFW rule set so the changes take effect immediately.

```bash
sudo systemctl restart sshd
```

> Restarts the SSH daemon to load the new port setting.

From now on, connect to your server with the `-p` flag:

```bash
ssh -p 2222 admin@YOUR_SERVER_IP
```

> `-p 2222` tells the SSH client to connect on port 2222 instead of the default port 22.

**Tip:** Add this to your `~/.ssh/config` on your local machine so you don't have to type `-p 2222` every time:

```
Host myserver
    HostName YOUR_SERVER_IP
    User admin
    Port 2222
    IdentityFile ~/.ssh/id_ed25519
```

> Now you can simply run `ssh myserver` and all the options are applied automatically.

---

## 11. Optional: 2FA for SSH

Two-factor authentication adds a time-based one-time password (TOTP) as a second factor on top of your SSH key — so even if your key is somehow stolen, an attacker still can't get in.

```bash
sudo apt install libpam-google-authenticator -y
```

> Installs the Google Authenticator PAM (Pluggable Authentication Module). Despite the name, it works with any TOTP app such as Authy, 1Password, or the Google Authenticator app.

Run the setup wizard **as your regular user** (not root):

```bash
google-authenticator
```

> Walks you through TOTP setup interactively. It will:
> 1. Display a QR code — scan this with your authenticator app.
> 2. Show emergency backup codes — save these somewhere safe offline.
> 3. Ask a few configuration questions (answer `y` to all for the recommended settings).

Enable TOTP in the SSH PAM stack:

```bash
sudo nano /etc/pam.d/sshd
```

> Opens the PAM configuration for SSH. PAM controls the authentication flow for various system services.

Add this at the **top** of the file:

```
auth required pam_google_authenticator.so
```

> Tells PAM that the Google Authenticator module is required for SSH authentication. `required` means the check must pass — failing it denies login even if other methods succeed.

Update the SSH config to require both a key and a TOTP code:

```bash
sudo nano /etc/ssh/sshd_config
```

Set these two lines:

```ini
# Enables keyboard-interactive auth, which is used to prompt for the TOTP code
ChallengeResponseAuthentication yes

# Requires BOTH a valid SSH key AND a keyboard-interactive (TOTP) response
AuthenticationMethods publickey,keyboard-interactive
```

Restart SSH to apply:

```bash
sudo systemctl restart sshd
```

> Loads the updated SSH configuration. On next login you'll be prompted for your SSH key (as usual) and then immediately asked for the 6-digit TOTP code from your authenticator app.

---

## 12. Final Checklist

Use this as a quick reference to confirm your VPS is hardened:

| Task | Done |
|------|------|
| System fully updated | ✅ |
| Non-root sudo user created | ✅ |
| SSH key authentication enabled | ✅ |
| Root SSH login disabled | ✅ |
| Password authentication disabled | ✅ |
| UFW firewall configured & enabled | ✅ |
| Fail2Ban installed & monitoring SSH | ✅ |
| Automatic security updates enabled | ✅ |
| Shared memory secured | ✅ |
| SSH port changed (optional) | ⬜ |
| 2FA enabled (optional) | ⬜ |

---

## 🔐 Security Philosophy

- **Principle of least privilege**: Only give users the access they actually need.
- **Defense in depth**: Layer multiple security controls — keys + firewall + Fail2Ban + 2FA.
- **Keep it minimal**: Uninstall software and services you don't use.
- **Monitor regularly**: Check logs (`/var/log/auth.log`, `journalctl -u sshd`) for suspicious activity.
- **Backups**: Always keep snapshots or backups before making system-level changes.

---

## 📚 Further Reading

- [Ubuntu Server Security Guide](https://ubuntu.com/server/docs/security-introduction)
- [SSH Hardening Guide (Mozilla)](https://infosec.mozilla.org/guidelines/openssh)
- [UFW Documentation](https://help.ubuntu.com/community/UFW)
- [Fail2Ban Wiki](https://www.fail2ban.org/wiki/index.php/Main_Page)
