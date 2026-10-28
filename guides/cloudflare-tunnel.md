# ☁️ Cloudflare Tunnel Setup Guide

A step-by-step guide to installing the `cloudflared` CLI and running a Cloudflare Tunnel on your Ubuntu VPS. This lets you expose services to the internet **without opening any inbound ports** on your firewall.

> Tested on **Ubuntu 22.04 LTS** and **Ubuntu 24.04 LTS**.

---

## What Is a Cloudflare Tunnel?

Normally, to make a service on your VPS accessible from the internet, you need to open ports on your firewall (e.g., port 80 for HTTP). This exposes your server's IP address and creates attack surface.

A Cloudflare Tunnel works the other way around: your VPS **dials out** to Cloudflare's network and keeps a persistent encrypted connection. All traffic flows through that outbound connection — your server never needs to accept any inbound connections at all.

Benefits:
- Your server's real IP address stays hidden
- No inbound firewall ports need to be opened
- Free DDoS protection and CDN from Cloudflare
- Works even behind NAT or without a static IP

---

## Prerequisites

- A VPS with Ubuntu 22.04 or 24.04 (ideally hardened with the [VPS setup guide](../README.md))
- A **Cloudflare account** — [sign up free at cloudflare.com](https://dash.cloudflare.com/sign-up)
- A **domain name** whose DNS is managed by Cloudflare (add your domain in the Cloudflare dashboard and point its nameservers there)
- A service running locally on your VPS to expose (e.g., a web app on `http://localhost:3000`)

---

## Table of Contents

1. [Install cloudflared](#1-install-cloudflared)
2. [Authenticate with Your Cloudflare Account](#2-authenticate-with-your-cloudflare-account)
3. [Create a Named Tunnel](#3-create-a-named-tunnel)
4. [Configure the Tunnel](#4-configure-the-tunnel)
5. [Route DNS to the Tunnel](#5-route-dns-to-the-tunnel)
6. [Run as a System Service](#6-run-as-a-system-service)
7. [Verify the Tunnel](#7-verify-the-tunnel)
8. [Managing the Tunnel](#8-managing-the-tunnel)
9. [Useful Examples](#9-useful-examples)

---

## 1. Install cloudflared

Install `cloudflared` via Cloudflare's official APT repository. This way it stays up to date automatically alongside your other packages.

**Step 1 — Add Cloudflare's package signing key:**

```bash
sudo install -m 0755 -d /usr/share/keyrings
```

> Creates the `/usr/share/keyrings` directory with permissions `0755` (readable by everyone, writable only by root). This is the standard location for APT keyring files on modern Ubuntu. The `-d` flag means "create directory" and `-m 0755` sets the permissions.

```bash
curl -fsSL https://pkg.cloudflare.com/cloudflare-main.gpg | sudo tee /usr/share/keyrings/cloudflare-main.gpg > /dev/null
```

> - `curl -fsSL` — Downloads the GPG signing key from Cloudflare's servers. `-f` fails silently on HTTP errors, `-s` suppresses progress output, `-S` still shows errors, `-L` follows redirects.
> - `sudo tee` — Writes the downloaded key into the keyrings directory with root permissions (since a regular user can't write there).
> - `> /dev/null` — Discards the duplicate output that `tee` would print to the terminal, since we only need it written to the file.

**Step 2 — Add Cloudflare's APT repository:**

```bash
echo "deb [signed-by=/usr/share/keyrings/cloudflare-main.gpg] https://pkg.cloudflare.com/cloudflared any main" | sudo tee /etc/apt/sources.list.d/cloudflared.list
```

> - `echo "deb ..."` — Constructs the APT source line. `deb` means it's a binary package source. `signed-by=...` tells APT to verify packages against the key we just downloaded. `any main` means it covers all Ubuntu/Debian releases.
> - `sudo tee /etc/apt/sources.list.d/cloudflared.list` — Writes it into APT's sources directory so the package manager knows where to look for `cloudflared`.

**Step 3 — Update the package list and install:**

```bash
sudo apt-get update
```

> Refreshes APT's knowledge of available packages, now including the newly added Cloudflare repository.

```bash
sudo apt-get install cloudflared -y
```

> Downloads and installs the `cloudflared` binary from Cloudflare's official repository. Using the repo (rather than a manual `.deb` download) means future `apt upgrade` commands will automatically keep it up to date.

**Step 4 — Verify the installation:**

```bash
cloudflared --version
```

> Prints the installed version of `cloudflared`. If this outputs a version number (e.g., `cloudflared version 2024.x.x`), the installation was successful.

---

## 2. Authenticate with Your Cloudflare Account

Link your VPS to your Cloudflare account so it can create tunnels and manage DNS records on your behalf.

```bash
cloudflared tunnel login
```

> Starts the authentication flow. The command will print a long URL — open it in your browser and log in to your Cloudflare account. You'll be asked to select which domain you want to use with the tunnel. After you approve, Cloudflare generates a certificate file and downloads it to your VPS automatically.

What happens after you approve in the browser:
- A credentials certificate (`cert.pem`) is saved to `~/.cloudflared/cert.pem`
- This file proves to Cloudflare's API that your VPS is authorised to manage tunnels for your domain

> **Note:** The `cert.pem` file is sensitive — it grants access to create and delete tunnels and DNS records for your domain. It's stored in your home directory and only readable by your user, which is fine.

---

## 3. Create a Named Tunnel

Create a tunnel with a meaningful name. This name is just a label for your own reference — it doesn't affect routing.

```bash
cloudflared tunnel create my-vps-tunnel
```

> - `tunnel create` — Tells `cloudflared` to register a new tunnel with Cloudflare's network.
> - `my-vps-tunnel` — The name you give the tunnel. You can name it anything (e.g., your domain name or server hostname). Use hyphens, not spaces.

After running this, you'll see output like:

```
Created tunnel my-vps-tunnel with id a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

Two things are created:
- A **Tunnel UUID** (the long string of letters and numbers) — this is the tunnel's permanent identifier. **Copy it**, you'll need it in the next step.
- A **credentials JSON file** at `~/.cloudflared/<TUNNEL-UUID>.json` — this allows `cloudflared` to authenticate as this specific tunnel.

To list your tunnels at any time:

```bash
cloudflared tunnel list
```

> Displays all tunnels registered under your Cloudflare account, along with their names, UUIDs, and status.

---

## 4. Configure the Tunnel

The configuration file tells `cloudflared` which tunnel to run and how to route incoming traffic to your local services.

```bash
sudo mkdir -p /etc/cloudflared
```

> Creates the `/etc/cloudflared` directory for system-wide configuration. Using `/etc/` (rather than `~/.cloudflared`) is the right location when the tunnel will run as a system service.

```bash
sudo nano /etc/cloudflared/config.yml
```

> Creates and opens the tunnel configuration file. YAML format is used (indentation matters — use spaces, not tabs).

Paste and customise the following template:

```yaml
# The UUID of the tunnel you created in Step 3
tunnel: YOUR-TUNNEL-UUID

# Path to the credentials file generated when you ran "tunnel create"
# Adjust the username if you're not running as root
credentials-file: /home/admin/.cloudflared/YOUR-TUNNEL-UUID.json

ingress:
  # Route traffic from a public subdomain to a local service
  # Replace "app.yourdomain.com" with your actual domain/subdomain
  # Replace "http://localhost:3000" with the address of your local service
  - hostname: app.yourdomain.com
    service: http://localhost:3000

  # Optional: expose SSH securely through the tunnel (no open port needed)
  - hostname: ssh.yourdomain.com
    service: ssh://localhost:22

  # Required catch-all rule — must always be the last entry
  # Any traffic that doesn't match a hostname above gets a 404 response
  - service: http_status:404
```

**Key fields explained:**

- `tunnel` — The UUID from Step 3. This tells `cloudflared` which registered tunnel to connect as.
- `credentials-file` — Path to the JSON file generated in Step 3. Must be the full absolute path.
- `ingress` — A list of routing rules processed top-to-bottom. Each rule maps a public hostname to a local service address.
  - `hostname` — The public-facing domain/subdomain that Cloudflare will receive traffic for.
  - `service` — Where to forward that traffic on your VPS (use `http://`, `https://`, or `ssh://`).
- The final `- service: http_status:404` is **required** — `cloudflared` will refuse to start without a catch-all rule at the end.

> **Multiple services:** Add as many `hostname`/`service` pairs as you need. Each subdomain can point to a different local port or service.

---

## 5. Route DNS to the Tunnel

Create CNAME records in Cloudflare DNS that point your subdomains to the tunnel. `cloudflared` can do this for you automatically.

```bash
cloudflared tunnel route dns my-vps-tunnel app.yourdomain.com
```

> - `tunnel route dns` — Tells `cloudflared` to create a DNS record in your Cloudflare account.
> - `my-vps-tunnel` — The tunnel name (or UUID) to point the record at.
> - `app.yourdomain.com` — The subdomain you want to route. This creates a CNAME record pointing to `<TUNNEL-UUID>.cfargotunnel.com`.

If you have multiple subdomains, run the command once for each:

```bash
cloudflared tunnel route dns my-vps-tunnel ssh.yourdomain.com
```

> You can verify the DNS records were created by logging into the Cloudflare dashboard → your domain → DNS → Records. You should see CNAME entries pointing to `<uuid>.cfargotunnel.com`.

---

## 6. Run as a System Service

Running `cloudflared` as a background system service ensures the tunnel starts automatically on boot and restarts if it crashes.

```bash
sudo cloudflared service install
```

> Registers `cloudflared` as a `systemd` service on your VPS. It reads the configuration from `/etc/cloudflared/config.yml` and creates a unit file at `/etc/systemd/system/cloudflared.service`. From this point on, the OS manages the tunnel process.

```bash
sudo systemctl enable cloudflared
```

> Configures the service to start automatically every time the VPS boots. Without this, the tunnel would stop if the server reboots.

```bash
sudo systemctl start cloudflared
```

> Starts the tunnel service right now (without needing a reboot). The tunnel connects to Cloudflare's network immediately.

Check that the service started successfully:

```bash
sudo systemctl status cloudflared
```

> Displays the current state of the `cloudflared` service. Look for `Active: active (running)` in green. If it shows `failed` or `error`, scroll down in the output to see the error message.

View live logs if you need to debug:

```bash
sudo journalctl -u cloudflared -f
```

> - `journalctl -u cloudflared` — Shows logs from the `cloudflared` systemd unit.
> - `-f` — "Follow" mode — streams new log lines in real time, like `tail -f`. Press `Ctrl+C` to exit.

---

## 7. Verify the Tunnel

Open your browser and navigate to `https://app.yourdomain.com`. If everything is working:

- The page loads your local service
- The connection is HTTPS (certificate provided by Cloudflare automatically)
- Your server's real IP is **not** exposed

You can also confirm the tunnel is active from the Cloudflare dashboard:

1. Log in to [dash.cloudflare.com](https://dash.cloudflare.com)
2. Go to **Zero Trust** → **Networks** → **Tunnels**
3. Your tunnel should show as **Healthy**

---

## 8. Managing the Tunnel

### Stop the tunnel

```bash
sudo systemctl stop cloudflared
```

> Stops the tunnel service immediately. The tunnel disconnects from Cloudflare's network. Your subdomains will stop resolving until the service is started again.

### Restart after config changes

```bash
sudo systemctl restart cloudflared
```

> Stops and restarts the service, reloading the config file. Use this any time you edit `/etc/cloudflared/config.yml`.

### Disable the service (stop auto-start on boot)

```bash
sudo systemctl disable cloudflared
```

> Removes the service from the boot sequence. The tunnel won't start automatically on reboot. The config and credentials are not deleted.

### Delete a tunnel entirely

```bash
cloudflared tunnel delete my-vps-tunnel
```

> Permanently deletes the tunnel registration from your Cloudflare account. The DNS CNAME records and the local credentials JSON file are not automatically removed — delete those manually if needed.

---

## 9. Useful Examples

### Expose a web app running on port 3000

```yaml
ingress:
  - hostname: app.yourdomain.com
    service: http://localhost:3000
  - service: http_status:404
```

### Expose multiple services on different subdomains

```yaml
ingress:
  - hostname: app.yourdomain.com
    service: http://localhost:3000
  - hostname: api.yourdomain.com
    service: http://localhost:8080
  - hostname: dashboard.yourdomain.com
    service: http://localhost:4000
  - service: http_status:404
```

### Expose SSH without opening port 22

```yaml
ingress:
  - hostname: ssh.yourdomain.com
    service: ssh://localhost:22
  - service: http_status:404
```

> To connect via SSH through the tunnel, you also need to install `cloudflared` on your **local machine** and add this to your local `~/.ssh/config`:
>
> ```
> Host ssh.yourdomain.com
>     ProxyCommand cloudflared access ssh --hostname %h
> ```
>
> Then connect normally with `ssh admin@ssh.yourdomain.com` — the connection is routed through the tunnel.

### Expose an HTTPS backend (e.g., with a self-signed cert)

```yaml
ingress:
  - hostname: app.yourdomain.com
    service: https://localhost:8443
    originRequest:
      noTLSVerify: true   # Skip cert verification for self-signed certs on origin
  - service: http_status:404
```

---

## 🔐 Security Notes

- **Close your firewall ports** — Since traffic flows through the tunnel, you can remove UFW rules for port 80 and 443. Your VPS doesn't need to accept inbound HTTP/HTTPS traffic directly. Only keep port 22 (or your custom SSH port) open for direct SSH access.
- **The credentials JSON is sensitive** — The file at `~/.cloudflared/<UUID>.json` allows anyone who has it to authenticate as your tunnel. Don't share it or commit it to version control.
- **Use Cloudflare Access** — For sensitive services (admin panels, dashboards), consider adding Cloudflare Access in front of the tunnel. It adds an authentication layer (SSO, email OTP, etc.) before users can reach your service.
- **Keep `cloudflared` updated** — Since you installed via APT, run `sudo apt upgrade cloudflared` periodically to stay current with security patches.

---

## 📚 Further Reading

- [Cloudflare Tunnel Docs](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [cloudflared GitHub Repository](https://github.com/cloudflare/cloudflared)
- [Cloudflare Access (Zero Trust)](https://developers.cloudflare.com/cloudflare-one/policies/access/)
- [Cloudflare Tunnel with SSH](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/use-cases/ssh/)
