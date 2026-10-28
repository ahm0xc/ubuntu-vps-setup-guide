# 🔒 Tailscale Setup Guide

A step-by-step guide to installing Tailscale on your Ubuntu VPS, building a secure private mesh network, and using it for zero-config remote access.

> Tested on **Ubuntu 22.04 LTS** and **Ubuntu 24.04 LTS**.

---

## What Is Tailscale?

Tailscale is a **mesh VPN** built on top of WireGuard. It connects all your devices — servers, laptops, phones — into a private network called a **tailnet**, where every device gets a stable private IP address (in the `100.x.x.x` range).

Unlike a traditional VPN that routes all traffic through a central server, Tailscale connects devices **directly to each other** (peer-to-peer) when possible, falling back to relay servers only when NAT traversal isn't possible.

Key benefits:
- **Zero open ports** — your VPS doesn't need to accept any inbound connections from the public internet
- **Works through NAT** — no static IP or port forwarding required
- **End-to-end encrypted** — all traffic is encrypted with WireGuard, regardless of the underlying network
- **Exit node support** — route all your traffic through a VPS as a personal VPN
- **Subnet routing** — expose an entire LAN, not just a single device
- **Generous free tier** — up to 100 devices, 3 users

---

## Prerequisites

- A VPS running Ubuntu 22.04 or 24.04 (ideally hardened with the [VPS setup guide](../README.md))
- A non-root sudo user (this guide uses `admin`)
- A **Tailscale account** — [sign up free at tailscale.com](https://tailscale.com/) (log in with Google, GitHub, or Microsoft)
- At least one other device to connect (laptop, phone, etc.) — install Tailscale on it from [tailscale.com/download](https://tailscale.com/download)

---

## Table of Contents

1. [Install Tailscale](#1-install-tailscale)
2. [Authenticate and Join Your Tailnet](#2-authenticate-and-join-your-tailnet)
3. [Verify the Connection](#3-verify-the-connection)
4. [SSH Over Tailscale](#4-ssh-over-tailscale)
5. [Run as a System Service](#5-run-as-a-system-service)
6. [Configure an Exit Node](#6-configure-an-exit-node)
7. [Configure Subnet Routing](#7-configure-subnet-routing)
8. [Use Tailscale SSH (Optional)](#8-use-tailscale-ssh-optional)
9. [Lock Down the Firewall](#9-lock-down-the-firewall)
10. [Useful Commands Reference](#10-useful-commands-reference)
11. [Security Notes](#11-security-notes)

---

## 1. Install Tailscale

Tailscale provides an official install script that adds their APT repository and installs the package. This keeps Tailscale up to date automatically.

**Option A — One-line install script (recommended):**

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

> - `curl -fsSL` — Downloads the install script. `-f` fails on HTTP errors, `-s` suppresses progress output, `-S` still shows errors, `-L` follows redirects.
> - `| sh` — Pipes the downloaded script directly into the shell for execution. The script detects your OS, adds Tailscale's APT repository, adds the signing key, and installs the `tailscale` package.

**Option B — Manual APT install (more transparent):**

If you prefer not to pipe a script directly into your shell, you can add the repository manually:

```bash
sudo mkdir -p /usr/share/keyrings
```

> Creates the standard directory for APT keyring files, with `-p` to avoid errors if it already exists.

```bash
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/jammy.noarmor.gpg | sudo tee /usr/share/keyrings/tailscale-archive-keyring.gpg > /dev/null
```

> Downloads Tailscale's package signing key and saves it to the keyrings directory. `sudo tee` writes it with root permissions. `> /dev/null` discards the duplicate terminal output.
>
> **Note:** Replace `jammy` with `noble` if you're on Ubuntu 24.04.

```bash
curl -fsSL https://pkgs.tailscale.com/stable/ubuntu/jammy.tailscale-keyring.list | sudo tee /etc/apt/sources.list.d/tailscale.list
```

> Downloads the APT source list file that points to Tailscale's repository and saves it in APT's sources directory.

```bash
sudo apt-get update
```

> Refreshes the package index so APT knows about the packages available in the newly added Tailscale repository.

```bash
sudo apt-get install tailscale -y
```

> Installs the `tailscale` package. This also installs and starts the `tailscaled` background daemon.

**Verify the installation:**

```bash
tailscale version
```

> Prints the installed version of Tailscale (e.g., `1.xx.x`). If this outputs a version number, the installation was successful.

---

## 2. Authenticate and Join Your Tailnet

Bring your VPS up on your tailnet by authenticating it with your Tailscale account.

```bash
sudo tailscale up
```

> Starts the Tailscale connection process. The command will print a URL — open it in your browser, log in to your Tailscale account, and click **Authorize**. Once you do, the terminal will confirm the device is connected.

After authorising, the terminal output looks like this:

```
To authenticate, visit:

    https://login.tailscale.com/a/xxxxxxxxxxxx

Success.
```

**Verify your Tailscale IP address:**

```bash
tailscale ip -4
```

> Prints the IPv4 address assigned to this device on your tailnet (always in the `100.x.x.x` range). This address is stable — it stays the same even if your public IP changes.

You can also see all connected devices from the [Tailscale admin console](https://login.tailscale.com/admin/machines).

---

## 3. Verify the Connection

From another device that's already on your tailnet (e.g., your laptop), ping the VPS to confirm connectivity:

```bash
ping 100.x.x.x
```

> Replace `100.x.x.x` with the Tailscale IP printed in Step 2. A successful ping confirms the two devices are directly connected through your tailnet.

You can also check the status of your tailnet from the VPS itself:

```bash
tailscale status
```

> Lists all devices currently connected to your tailnet, along with their hostnames, Tailscale IPs, and connection status. Devices that are online show a recent timestamp; offline devices show `offline`.

---

## 4. SSH Over Tailscale

Once connected to your tailnet, you can SSH into the VPS using its Tailscale IP from any device on your tailnet — no public IP or open SSH port required.

From your laptop (or any tailnet device):

```bash
ssh admin@100.x.x.x
```

> Replace `100.x.x.x` with the Tailscale IP of your VPS. This SSH session travels entirely over the encrypted Tailscale tunnel — it never touches the public internet directly.

**Tip:** Add an entry to `~/.ssh/config` on your local machine for convenience:

```
Host myvps
    HostName 100.x.x.x
    User admin
    IdentityFile ~/.ssh/id_ed25519
```

> Replace `100.x.x.x` with your VPS's Tailscale IP. After adding this, you can simply run `ssh myvps` to connect.

---

## 5. Run as a System Service

The `tailscaled` daemon is automatically installed and configured to start on boot by the install script. Verify it's running:

```bash
sudo systemctl status tailscaled
```

> Displays the current state of the Tailscale daemon. Look for `Active: active (running)` in green.

If it's not enabled to start on boot, enable it:

```bash
sudo systemctl enable tailscaled
```

> Configures `tailscaled` to start automatically every time the VPS boots. Without this, your VPS would drop off your tailnet after a reboot until you manually started the service.

```bash
sudo systemctl start tailscaled
```

> Starts the daemon right now without needing a reboot.

**After a reboot**, Tailscale reconnects automatically without requiring re-authentication, because the auth credentials are stored locally.

---

## 6. Configure an Exit Node

An exit node routes **all internet traffic** from other tailnet devices through your VPS. This turns your VPS into a personal VPN — useful for securing traffic on untrusted networks (coffee shops, hotels).

**On the VPS — advertise as an exit node:**

```bash
sudo tailscale up --advertise-exit-node
```

> The `--advertise-exit-node` flag tells Tailscale that this device is willing to act as an exit node for other devices on the tailnet. The VPS starts advertising this capability to your tailnet.

**Enable IP forwarding** (required for the VPS to forward packets):

```bash
echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
```

> Appends the IP forwarding setting to a new sysctl config file. IP forwarding allows the kernel to forward packets from one network interface to another — necessary for the exit node to work.

```bash
echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
```

> Same as above but for IPv6 forwarding.

```bash
sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
```

> Applies the sysctl settings immediately without rebooting. The settings are also automatically applied on future boots because they're stored in the `/etc/sysctl.d/` directory.

**Approve the exit node in the admin console:**

1. Go to [login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines)
2. Find your VPS and click the three-dot menu → **Edit route settings**
3. Toggle **Use as exit node** to enabled

> Exit nodes must be approved in the admin console (or via policy) as a security measure — a device can't silently become an exit node without authorization.

**On another device — use the exit node:**

```bash
sudo tailscale up --exit-node=100.x.x.x
```

> Replace `100.x.x.x` with the Tailscale IP of your VPS. After running this, all internet traffic from that device is routed through your VPS. To stop using the exit node, run `sudo tailscale up --exit-node=` (leave the value blank).

---

## 7. Configure Subnet Routing

Subnet routing lets you expose an entire private network (e.g., `192.168.1.0/24`) through your VPS to your tailnet — without installing Tailscale on every device on that network.

**On the VPS — advertise the subnet:**

```bash
sudo tailscale up --advertise-routes=192.168.1.0/24
```

> Replace `192.168.1.0/24` with the actual subnet you want to expose. This tells Tailscale that traffic destined for that IP range can be routed through this device.

Ensure IP forwarding is enabled (see Step 6 if you haven't done this yet).

**Approve the subnet in the admin console:**

1. Go to [login.tailscale.com/admin/machines](https://login.tailscale.com/admin/machines)
2. Find your VPS and click the three-dot menu → **Edit route settings**
3. Enable the subnet route you advertised

> Like exit nodes, subnet routes must be explicitly approved in the admin console before other devices can use them.

**On another tailnet device — use the subnet route:**

Subnet routes are used automatically once approved — no configuration needed on the connecting device. Traffic to `192.168.1.x` addresses is transparently routed through the VPS.

---

## 8. Use Tailscale SSH (Optional)

Tailscale has a built-in SSH feature that manages SSH access using Tailscale identity, eliminating the need to manage SSH keys manually across devices.

**Enable Tailscale SSH on the VPS:**

```bash
sudo tailscale up --ssh
```

> Enables Tailscale's SSH server on this device. When another tailnet device tries to SSH into this machine using its Tailscale IP, authentication is handled by Tailscale identity (based on which Tailscale account the connecting device is logged in as) — no SSH keys needed.

**Connect using Tailscale SSH (from another tailnet device):**

```bash
ssh admin@100.x.x.x
```

> Same command as normal SSH. If Tailscale SSH is enabled and the connecting device is on the same tailnet, Tailscale handles authentication. Permissions are controlled by ACL policies in your admin console.

**Manage access via ACL policies in the admin console:**

Tailscale SSH access is controlled by your tailnet's ACL (Access Control List) policy. In the [Access Controls](https://login.tailscale.com/admin/acls) section, you can define rules like:

```json
{
  "ssh": [
    {
      "action": "accept",
      "src": ["autogroup:member"],
      "dst": ["tag:server"],
      "users": ["admin"]
    }
  ]
}
```

> This example allows all tailnet members to SSH into any device tagged as `server`, logging in as `admin`. Tailscale's ACL language is flexible — you can restrict access to specific users, devices, or tags.

---

## 9. Lock Down the Firewall

Once your VPS is on your tailnet, you can significantly tighten the UFW firewall. Since all access goes over Tailscale, you no longer need to expose SSH (or any other port) to the public internet.

**Allow the Tailscale interface through UFW:**

```bash
sudo ufw allow in on tailscale0
```

> Allows all incoming traffic on the `tailscale0` network interface (the virtual interface Tailscale creates). This is necessary so UFW doesn't block Tailscale traffic. The `tailscale0` interface only carries encrypted private tailnet traffic — no public traffic reaches it.

**Block direct SSH from the public internet (optional but recommended):**

```bash
sudo ufw delete allow ssh
```

> Removes the rule that allowed inbound SSH on port 22 from anywhere. After doing this, SSH is only accessible over your tailnet (via the `tailscale0` interface). **Only do this after confirming you can SSH via Tailscale**, or you will lock yourself out.

**Reload UFW to apply changes:**

```bash
sudo ufw reload
```

> Applies all pending rule changes without disrupting existing connections.

**Final UFW status after locking down:**

```bash
sudo ufw status verbose
```

> Verify the rules look correct. You should see traffic allowed on `tailscale0` but no public rule for port 22.

> ⚠️ **Before removing the public SSH rule**, always confirm you can successfully SSH into the VPS via its Tailscale IP from another device. If Tailscale is down and you've removed port 22, you'll need to use your hosting provider's emergency console (e.g., KVM/VNC access) to recover.

---

## 10. Useful Commands Reference

### Check Tailscale status

```bash
tailscale status
```

> Lists all devices on your tailnet, their IPs, hostnames, and whether they're online or offline.

### Show your own Tailscale IP

```bash
tailscale ip -4
```

> Prints just the IPv4 address assigned to this device on your tailnet.

### Ping a tailnet device

```bash
tailscale ping 100.x.x.x
```

> Sends a Tailscale-layer ping (not a regular ICMP ping) to another device. Unlike a regular `ping`, this also shows whether the connection is **direct** (peer-to-peer) or going through a **relay (DERP server)**, and the latency for each.

### Check connection details

```bash
tailscale netcheck
```

> Runs a network diagnostic and reports UDP availability, DERP latency to various regions, and whether NAT traversal is likely to work. Useful for diagnosing slow or relay-only connections.

### Disconnect from the tailnet

```bash
sudo tailscale down
```

> Takes this device offline from the tailnet. The device disappears from the admin console and can no longer communicate with other tailnet devices. The auth credentials are preserved — run `sudo tailscale up` to reconnect.

### Log out and remove credentials

```bash
sudo tailscale logout
```

> Disconnects from the tailnet **and** removes the authentication credentials. The device is deregistered from your Tailscale account. You'll need to re-authenticate with `sudo tailscale up` and authorize the device again.

### Restart the Tailscale daemon

```bash
sudo systemctl restart tailscaled
```

> Restarts the background Tailscale daemon. Useful if Tailscale is behaving unexpectedly or after making system-level network changes.

### Update Tailscale

```bash
sudo apt update && sudo apt upgrade tailscale
```

> Since Tailscale was installed via APT, this updates it to the latest stable version alongside your other system packages.

---

## 11. Security Notes

- **Direct vs. relay connections** — Run `tailscale ping 100.x.x.x` to check if your connections are direct. Direct connections are ideal; relay (DERP) connections still work but add latency. If stuck on relay, check that UDP port 41641 is allowed outbound from your VPS.
- **Remove the port 22 rule** — Once you're consistently SSH-ing over Tailscale, remove the public-facing SSH rule from UFW (see [Step 9](#9-lock-down-the-firewall)). This eliminates the entire category of SSH brute-force attacks.
- **Use ACL policies** — By default, all tailnet devices can communicate with each other. For a multi-user tailnet, configure [ACL policies](https://login.tailscale.com/admin/acls) to restrict which devices can talk to which.
- **Tag your servers** — In the admin console, apply tags (e.g., `tag:server`) to your VPS. Tags let you write ACL rules that apply to a role rather than a specific device or user.
- **Enable MagicDNS** — Tailscale's [MagicDNS](https://tailscale.com/kb/1081/magicdns/) feature lets you refer to tailnet devices by hostname (e.g., `ssh admin@my-vps`) instead of IP address. Enable it in the admin console under DNS settings.
- **Keep Tailscale updated** — Since you installed via APT, run `sudo apt upgrade tailscale` regularly to get security fixes and new features.
- **Review connected devices** — Periodically check the [admin console](https://login.tailscale.com/admin/machines) and remove devices you no longer use.

---

## 📚 Further Reading

- [Tailscale Documentation](https://tailscale.com/kb/)
- [Tailscale on Ubuntu (official guide)](https://tailscale.com/kb/1036/install-ubuntu/)
- [Exit Nodes](https://tailscale.com/kb/1103/exit-nodes/)
- [Subnet Routers](https://tailscale.com/kb/1019/subnets/)
- [Tailscale SSH](https://tailscale.com/kb/1193/tailscale-ssh/)
- [ACL Policy Syntax](https://tailscale.com/kb/1018/acls/)
- [MagicDNS](https://tailscale.com/kb/1081/magicdns/)
