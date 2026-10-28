# 🔍 Meilisearch Setup Guide

A step-by-step guide to installing, configuring, securing, and maintaining Meilisearch on Ubuntu — with both native Systemd and Docker deployment options.

> Tested on **Ubuntu 22.04 LTS** and **Ubuntu 24.04 LTS**.

---

## What Is Meilisearch?

Meilisearch is an open-source, lightning-fast, and typo-tolerant search engine written in Rust. It is engineered to deliver an instant search-as-you-type experience out of the box with zero complex configuration.

Key characteristics:
- **Instant search (< 50ms)** — Delivers sub-50ms search responses as users type.
- **Typo-tolerant** — Automatically handles typos, spelling mistakes, and phonetic variations.
- **Relevance-first** — Comes with battle-tested ranking rules out of the box (typo, words, proximity, attribute, sort, exactness).
- **Lightweight & fast** — Written in Rust; uses memory-mapped files (LMDB) with low resource overhead compared to JVM-based search engines like Elasticsearch.
- **Faceted & filterable** — Built-in support for dynamic filtering, facet distribution, sorting, and geo-search.
- **Hybrid & vector search** — Native support for vector embeddings and semantic search alongside keyword search.
- **RESTful API** — Simple JSON-over-HTTP API with official SDKs for Python, JavaScript, PHP, Go, Ruby, Rust, and more.

> Unlike Elasticsearch, which typically requires 4 GB+ RAM for its JVM heap alone, Meilisearch can comfortably run small-to-medium search workloads on modest VPS instances with 1–2 GB RAM.

---

## Prerequisites

- A VPS running Ubuntu 22.04 or 24.04 (ideally hardened using the [VPS security guide](../README.md))
- A non-root sudo user (the guide uses `admin`)
- At least 1–2 GB RAM (2 GB+ recommended for medium datasets; configure a swap file if on a 1 GB VPS)
- A domain or subdomain (e.g., `search.yourdomain.com`) pointing to your server's IP if exposing publicly over HTTPS

---

## Table of Contents

1. [Hardware & Swap Preparation](#1-hardware--swap-preparation)
2. [Method 1: Native Installation (Recommended)](#2-method-1-native-installation-recommended)
3. [Configuration File Setup](#3-configuration-file-setup)
4. [Run Meilisearch as a Systemd Service](#4-run-meilisearch-as-a-systemd-service)
5. [Method 2: Docker Installation](#5-method-2-docker-installation)
6. [Configure Reverse Proxy & SSL (Nginx)](#6-configure-reverse-proxy--ssl-nginx)
7. [API Keys & Access Control](#7-api-keys--access-control)
8. [Testing & Essential Operations](#8-testing--essential-operations)
9. [Backups, Dumps & Upgrades](#9-backups-dumps--upgrades)
10. [Performance & Production Hardening](#10-performance--production-hardening)
11. [Troubleshooting & Maintenance](#11-troubleshooting--maintenance)

---

## 1. Hardware & Swap Preparation

Meilisearch relies on **LMDB** (Lightning Memory-Mapped Database). LMDB maps database files directly into virtual memory. While read operations are exceptionally fast, batch indexing operations require sufficient physical memory. If memory runs out during heavy indexing, the Linux kernel OOM (Out Of Memory) killer may terminate the process.

If your VPS has 2 GB RAM or less, verify or create a swap file.

**Check current swap status:**

```bash
free -h
```

> Displays available physical memory and active swap space.

**If swap is 0 or less than 2 GB, create a 2 GB swap file:**

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
```

> - `fallocate -l 2G /swapfile` — Pre-allocates a 2-gigabyte file at `/swapfile`.
> - `chmod 600` — Restricts read and write permissions to the root user only.
> - `mkswap` — Formats the file as Linux swap space.
> - `swapon` — Activates the swap file immediately.

**Make swap persistent across reboots:**

```bash
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

> Appends the swap entry to `/etc/fstab` so systemd mounts it automatically on boot.

---

## 2. Method 1: Native Installation (Recommended)

Running Meilisearch directly as a native binary managed by `systemd` gives you maximum performance, minimal overhead, and native integration with system logs (`journalctl`).

### Step 1 — Download the official Meilisearch binary

You can install via the official one-liner script or download the binary manually.

#### Option A: Official install script

```bash
curl -fsSL https://install.meilisearch.com | sh
```

> - `-f` — Fails silently on server HTTP errors.
> - `-s` — Runs silently without progress bar spam.
> - `-S` — Shows error messages if the request fails.
> - `-L` — Follows redirects.
> - Downloads the latest release binary into your current working directory as `./meilisearch`.

#### Option B: Manual GitHub download

```bash
ARCH=$(dpkg --print-architecture)
TAG=$(curl -fsSL https://api.github.com/repos/meilisearch/meilisearch/releases/latest | grep '"tag_name":' | sed -E 's/.*"([^"]+)".*/\1/')
curl -fsSL "https://github.com/meilisearch/meilisearch/releases/download/${TAG}/meilisearch-linux-${ARCH}" -o meilisearch
chmod +x meilisearch
```

> Detects your architecture (`amd64` or `arm64`), grabs the latest release tag from GitHub, downloads the precompiled binary, and marks it executable (`+x`).

### Step 2 — Move binary to system path

```bash
sudo mv meilisearch /usr/local/bin/meilisearch
sudo chown root:root /usr/local/bin/meilisearch
sudo chmod 755 /usr/local/bin/meilisearch
```

> Moves the binary to `/usr/local/bin` so it is globally accessible, owned by `root`, and executable by all system users (`755`).

Verify the binary works:

```bash
meilisearch --version
```

> Prints the installed Meilisearch version.

---

## 3. Configuration File Setup

### Step 1 — Create a dedicated system user

For security, never run Meilisearch as `root` or your primary administrative user.

```bash
sudo useradd -r -s /bin/false -d /var/lib/meilisearch meilisearch
```

> - `-r` — Creates a system user (UID < 1000).
> - `-s /bin/false` — Prevents interactive shell login for this service account.
> - `-d /var/lib/meilisearch` — Assigns its home/working directory to `/var/lib/meilisearch`.

### Step 2 — Create data and dump directories

```bash
sudo mkdir -p /var/lib/meilisearch/data
sudo mkdir -p /var/lib/meilisearch/dumps
sudo mkdir -p /var/lib/meilisearch/snapshots
sudo chown -R meilisearch:meilisearch /var/lib/meilisearch
sudo chmod 750 /var/lib/meilisearch
```

> - Creates storage directories for indices (`data`), exports (`dumps`), and periodic database snapshots (`snapshots`).
> - Grants ownership to the `meilisearch` user and group.
> - Sets directory permissions to `750` (read/write/traverse for owner, read/traverse for group, no access for others).

### Step 3 — Generate a secure Master Key

In production mode, Meilisearch requires a master key of at least **16 bytes**.

Generate a strong cryptographic key:

```bash
openssl rand -base64 32
```

> Generates a secure, 32-byte pseudo-random string encoded in Base64 (e.g., `4qT+f7kL5vN9...`). Keep this safe!

### Step 4 — Create `/etc/meilisearch.toml`

Meilisearch supports configuration via TOML, YAML, or environment variables. A central config file in `/etc/` is standard for system services.

Create and open the configuration file:

```bash
sudo nano /etc/meilisearch.toml
```

Paste the following configuration (replace `PASTE_YOUR_GENERATED_MASTER_KEY_HERE` with the key generated in the previous step):

```toml
# Meilisearch Production Configuration
# /etc/meilisearch.toml

# Server environment ('production' disables the built-in web search preview)
env = "production"

# Network binding
# CRITICAL: Always bind to 127.0.0.1 on a VPS so Meilisearch is not exposed directly to the public internet
http_addr = "127.0.0.1:7700"

# Master key for authentication (must be at least 16 bytes in production)
master_key = "PASTE_YOUR_GENERATED_MASTER_KEY_HERE"

# Data storage paths
db_path = "/var/lib/meilisearch/data"
dump_dir = "/var/lib/meilisearch/dumps"
snapshot_dir = "/var/lib/meilisearch/snapshots"

# Snapshot interval (uncomment to enable automated periodic snapshots in seconds)
# schedule_snapshot = 86400

# Resource limits (adjust based on your VPS RAM)
# Limits the maximum amount of RAM used during indexation
max_indexing_memory = "1GiB"

# Number of worker threads for indexing (leave commented to use all available cores, or limit on smaller VPS)
# max_indexing_threads = 2

# Maximum payload size for document batches (default: 100MB)
http_payload_size_limit = "100MB"

# Logging level: 'ERROR', 'WARN', 'INFO', or 'DEBUG'
log_level = "INFO"
```

Lock down file permissions so only `root` and the `meilisearch` user can read the master key:

```bash
sudo chown root:meilisearch /etc/meilisearch.toml
sudo chmod 640 /etc/meilisearch.toml
```

> - Owner `root` can read/write (`6`).
> - Group `meilisearch` can read (`4`).
> - Other users have zero access (`0`).

---

## 4. Run Meilisearch as a Systemd Service

Create a systemd unit file to manage Meilisearch as a continuous background daemon with automatic restart on failure.

### Step 1 — Create the service unit

```bash
sudo nano /etc/systemd/system/meilisearch.service
```

Add the following definition:

```ini
[Unit]
Description=Meilisearch Full-Text Search Engine
Documentation=https://www.meilisearch.com/docs
After=network.target

[Service]
Type=simple
User=meilisearch
Group=meilisearch
ExecStart=/usr/local/bin/meilisearch --config-file-path /etc/meilisearch.toml
Restart=always
RestartSec=5
WorkingDirectory=/var/lib/meilisearch

# Security sandboxing
NoNewPrivileges=true
PrivateTmp=true
ProtectSystem=full
ProtectHome=true
ReadWritePaths=/var/lib/meilisearch

# Resource limits
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

Key directives explained:
- `ExecStart` — Runs the binary with the config file we created.
- `Restart=always` & `RestartSec=5` — Automatically restarts the process if it terminates or crashes unexpectedly.
- `NoNewPrivileges=true`, `ProtectSystem=full`, `ProtectHome=true` — Hardens the service against privilege escalation and restricts unauthorized filesystem tampering.
- `LimitNOFILE=65535` — Increases file descriptor limits so high-concurrency requests or large index files don't trigger "Too many open files" errors.

### Step 2 — Enable and start the service

```bash
sudo systemctl daemon-reload
sudo systemctl enable meilisearch
sudo systemctl start meilisearch
```

> - `daemon-reload` — Notifies systemd of the new unit file.
> - `enable` — Configures the service to launch automatically upon server reboot.
> - `start` — Starts Meilisearch immediately.

### Step 3 — Verify service status

```bash
sudo systemctl status meilisearch
```

> Look for `Active: active (running)`.

Test local connectivity with `curl`:

```bash
curl http://127.0.0.1:7700/health
```

Expected response:
```json
{"status":"available"}
```

---

## 5. Method 2: Docker Installation

If you manage your VPS infrastructure with Docker and Docker Compose (as covered in the [Docker Setup Guide](docker-setup.md)), Meilisearch provides an official, well-maintained container image.

### Step 1 — Create a project directory

```bash
mkdir -p ~/meilisearch-docker
cd ~/meilisearch-docker
```

### Step 2 — Create `.env` file

Generate a master key and store it in an environment file:

```bash
cat <<EOF > .env
MEILI_MASTER_KEY=$(openssl rand -base64 32)
MEILI_ENV=production
EOF
chmod 600 .env
```

> Stores your master key securely with permissions restricted to your current user.

### Step 3 — Create `docker-compose.yml`

```bash
nano docker-compose.yml
```

Add the following configuration:

```yaml
services:
  meilisearch:
    image: getmeili/meilisearch:v1.12
    container_name: meilisearch
    restart: unless-stopped
    ports:
      # Bind exclusively to 127.0.0.1 so the container port is NOT publicly exposed
      - "127.0.0.1:7700:7700"
    environment:
      - MEILI_ENV=${MEILI_ENV}
      - MEILI_MASTER_KEY=${MEILI_MASTER_KEY}
      - MEILI_NO_ANALYTICS=true
      - MEILI_MAX_INDEXING_MEMORY=1GiB
    volumes:
      - ./meili_data:/meilisearch/data.ms
      - ./meili_dumps:/dumps
      - ./meili_snapshots:/snapshots
    deploy:
      resources:
        limits:
          memory: 2G
```

> - `127.0.0.1:7700:7700` — Binds only to localhost, preventing Docker's internal iptables rules from bypassing your UFW firewall and exposing port 7700 to the public web.
> - `MEILI_MAX_INDEXING_MEMORY=1GiB` — Throttles memory during heavy index operations to prevent RAM exhaustion.
> - `MEILI_NO_ANALYTICS=true` — Disables telemetry reporting for privacy.
> - `volumes:` — Preserves indices (`./meili_data`), exports (`./meili_dumps`), and periodic snapshots (`./meili_snapshots`) in your project directory on the host VPS.
> - `deploy.resources.limits.memory: 2G` — Hard ceiling that protects host stability from runaway container processes.
> - `restart: unless-stopped` — Automatically restarts Meilisearch on system reboots or crashes.

### Step 4 — Start the container

```bash
docker compose up -d
```

> - `up -d` — Pulls the image (if not already cached) and starts the container in the background.

Verify container health:

```bash
docker compose ps
curl http://127.0.0.1:7700/health
```

---

## 6. Configure Reverse Proxy & SSL (Nginx)

Never expose raw Meilisearch HTTP port `7700` directly to the open internet. Instead, place it behind a hardened reverse proxy like **Nginx** with TLS/SSL encryption via Let's Encrypt (Certbot).

### Step 1 — Install Nginx and Certbot

```bash
sudo apt update
sudo apt install -y nginx certbot python3-certbot-nginx
```

### Step 2 — Create Nginx configuration

Create a virtual host configuration file (replace `search.yourdomain.com` with your domain):

```bash
sudo nano /etc/nginx/sites-available/meilisearch.conf
```

Add the following server block:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name search.yourdomain.com;

    # Important: Large document batch uploads require a higher client body size limit
    client_max_body_size 100M;

    location / {
        proxy_pass http://127.0.0.1:7700;
        proxy_http_version 1.1;

        # Standard proxy headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Disable buffering for streaming responses
        proxy_buffering off;
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
    }
}
```

> **Why `client_max_body_size 100M;`?** By default, Nginx limits request bodies to 1 MB. When batch-indexing thousands of JSON records into Meilisearch, Nginx will reject the payload with `413 Request Entity Too Large` unless this limit is raised.

### Step 3 — Enable site and test configuration

```bash
sudo ln -s /etc/nginx/sites-available/meilisearch.conf /etc/nginx/sites-enabled/
sudo nginx -t
```

> - `ln -s` — Symlinks the configuration into `sites-enabled`.
> - `nginx -t` — Tests the Nginx configuration syntax for errors.

Reload Nginx:

```bash
sudo systemctl reload nginx
```

### Step 4 — Obtain a free SSL certificate with Certbot

```bash
sudo certbot --nginx -d search.yourdomain.com
```

> Certbot prompts for your email address, agrees to terms, validates your domain, automatically provisions a free Let's Encrypt certificate, and updates the Nginx file with HTTPS redirects.

Verify firewall allows HTTP and HTTPS traffic:

```bash
sudo ufw allow 'Nginx Full'
```

> Opens ports `80` (HTTP) and `443` (HTTPS) in UFW while keeping Meilisearch's internal port `7700` protected.

---

## 7. API Keys & Access Control

Meilisearch provides a tiered authentication architecture designed to keep your data secure while permitting safe public client-side searches.

### The 3 Key Levels

1. **Master Key**:
   - Configured in `/etc/meilisearch.toml`.
   - Has absolute authority. Can create, rotate, and delete API keys, delete indices, and trigger backups.
   - **Never share or expose in client-side code** (e.g., React, Vue, mobile apps).

2. **Default Admin API Key**:
   - Automatically generated upon startup when the master key is set.
   - Has full permissions over documents and indices, but cannot manage other API keys.
   - Use in your **backend servers** (e.g., Node.js, Django, Laravel, Rails) to sync records.

3. **Default Search API Key**:
   - Automatically generated upon startup.
   - Strictly read-only: permitted only for the `search` action.
   - Safe to embed in frontend web apps and mobile clients.

### Viewing your generated keys

Query the `/keys` endpoint using your Master Key:

```bash
curl -s http://127.0.0.1:7700/keys \
  -H "Authorization: Bearer YOUR_MASTER_KEY" | jq .
```

> Replace `YOUR_MASTER_KEY` with your actual master key. (Tip: run `sudo apt install -y jq` for clean JSON formatting).

You will see output containing two auto-generated keys:
- A key with `name: "Default Search API Key"` and `actions: ["search"]`.
- A key with `name: "Default Admin API Key"` and `actions: ["*"]`.

### Creating a custom scoped API key

For fine-grained control, you can create custom keys restricted to specific indices, specific actions, or with an expiration timestamp:

```bash
curl -X POST 'http://127.0.0.1:7700/keys' \
  -H 'Authorization: Bearer YOUR_MASTER_KEY' \
  -H 'Content-Type: application/json' \
  --data-binary '{
    "description": "Public storefront search key",
    "actions": ["search"],
    "indexes": ["products"],
    "expiresAt": "2028-01-01T00:00:00Z"
  }'
```

> - `"actions": ["search"]` — Key can only execute search queries.
> - `"indexes": ["products"]` — Key can only access the `products` index; other indices are inaccessible.
> - `"expiresAt"` — Automatically invalidates the key after the specified timestamp.

### Tenant Tokens for Multi-Tenant Apps

If multiple customers or organizations share the same index, do not issue generic keys. Use **Tenant Tokens** generated on your backend server. Tenant tokens are digitally signed JWTs (JSON Web Tokens) specifying search rules (e.g., `filter: "tenant_id = 42"`). The client cannot tamper with or bypass the filter rule.

---

## 8. Testing & Essential Operations

All examples below use `127.0.0.1:7700` locally with `YOUR_MASTER_KEY` (or your Admin Key). If running remotely, replace with `https://search.yourdomain.com`.

### 1. Check Server Version & Health

```bash
curl -s http://127.0.0.1:7700/version \
  -H "Authorization: Bearer YOUR_MASTER_KEY"
```

### 2. Create an Index and Add Documents

Let's index a small batch of movie documents:

```bash
curl -X POST 'http://127.0.0.1:7700/indexes/movies/documents' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_MASTER_KEY' \
  --data-binary '[
    {
      "id": 1,
      "title": "Interstellar",
      "genre": "Sci-Fi",
      "release_year": 2014,
      "rating": 8.7
    },
    {
      "id": 2,
      "title": "Inception",
      "genre": "Sci-Fi",
      "release_year": 2010,
      "rating": 8.8
    },
    {
      "id": 3,
      "title": "The Dark Knight",
      "genre": "Action",
      "release_year": 2008,
      "rating": 9.0
    }
  ]'
```

> Meilisearch responds with a task object: `{"taskUid": 0, "status": "enqueued", ...}`.
>
> In Meilisearch, document ingestion and index mutations are **asynchronous**. The endpoint returns immediately with a `taskUid`.

### 3. Check Task Status

To confirm the documents finished processing:

```bash
curl -s http://127.0.0.1:7700/tasks/0 \
  -H "Authorization: Bearer YOUR_MASTER_KEY"
```

> Look for `"status": "succeeded"`. If there was an error (e.g. malformed JSON or invalid primary key), `"status": "failed"` with details will appear here.

### 4. Execute a Search Query

Test search with intentional typos:

```bash
curl -X POST 'http://127.0.0.1:7700/indexes/movies/search' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_MASTER_KEY' \
  --data-binary '{
    "q": "interstelar"
  }'
```

> Notice that `interstelar` (missing an 'l') instantly matches `Interstellar` due to Meilisearch's automatic typo tolerance.

### 5. Configure Filterable and Sortable Attributes

Before you can filter or sort by fields like `genre` or `rating`, you must declare them in the index settings:

```bash
curl -X PATCH 'http://127.0.0.1:7700/indexes/movies/settings' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_MASTER_KEY' \
  --data-binary '{
    "filterableAttributes": ["genre", "release_year"],
    "sortableAttributes": ["rating", "release_year"]
  }'
```

### 6. Perform a Filtered & Sorted Search

```bash
curl -X POST 'http://127.0.0.1:7700/indexes/movies/search' \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_MASTER_KEY' \
  --data-binary '{
    "q": "",
    "filter": "genre = Sci-Fi AND release_year >= 2010",
    "sort": ["rating:desc"]
  }'
```

> Returns Sci-Fi movies released from 2010 onwards, sorted by rating from highest to lowest.

---

## 9. Backups, Dumps & Upgrades

Meilisearch offers two backup mechanisms: **Dumps** and **Snapshots**. Understanding the difference is crucial for operations.

| Feature | Dump | Snapshot |
|---------|------|----------|
| **Format** | Raw uncompressed JSON metadata (`.dump` gzipped archive) | Exact raw binary copy of the active LMDB database |
| **Portability** | **Cross-version compatible**. Required for major/minor version upgrades | Compatible only with the identical Meilisearch version and OS architecture |
| **Speed** | Slower to generate and import (re-indexes documents) | Instantaneous copy |
| **Use case** | Version upgrades, data migration across servers | Fast disaster recovery |

### Creating a Dump

Trigger a dump generation via API:

```bash
curl -X POST 'http://127.0.0.1:7700/dumps' \
  -H 'Authorization: Bearer YOUR_MASTER_KEY'
```

> Returns a `taskUid`. Monitor task completion:

```bash
curl -s http://127.0.0.1:7700/tasks/TASK_UID \
  -H "Authorization: Bearer YOUR_MASTER_KEY"
```

Once succeeded, the dump archive will be saved in `/var/lib/meilisearch/dumps/<dump-uid>.dump`.

### Upgrading Meilisearch safely

When upgrading between minor or major versions (e.g. from `v1.11` to `v1.12`), LMDB binary formats may change. Follow this strict procedure:

**1. Create a dump before touching anything:**
```bash
curl -X POST 'http://127.0.0.1:7700/dumps' -H 'Authorization: Bearer YOUR_MASTER_KEY'
```

**2. Stop the service:**
```bash
sudo systemctl stop meilisearch
```

**3. Download the new binary:**
```bash
curl -fsSL https://install.meilisearch.com | sh
sudo mv meilisearch /usr/local/bin/meilisearch
```

**4. Move the old database folder out of the way:**
```bash
sudo mv /var/lib/meilisearch/data /var/lib/meilisearch/data.old
sudo mkdir /var/lib/meilisearch/data
sudo chown -R meilisearch:meilisearch /var/lib/meilisearch/data
```

**5. Import the dump:**
```bash
sudo -u meilisearch /usr/local/bin/meilisearch \
  --config-file-path /etc/meilisearch.toml \
  --import-dump /var/lib/meilisearch/dumps/<YOUR_DUMP_FILE>.dump
```

> Once the import completes, press `Ctrl+C` and restart the systemd service:

```bash
sudo systemctl start meilisearch
```

---

## 10. Performance & Production Hardening

### Tune Indexing Memory

By default, Meilisearch attempts to use up to 2/3 of total available memory during indexing. On a multi-tenant VPS or a 2 GB server where other services (like PostgreSQL or Node.js) run simultaneously, this can cause out-of-memory errors.

Cap indexing memory in `/etc/meilisearch.toml`:

```toml
max_indexing_memory = "1GiB"
```

### Limit Indexing CPU Threads

If batch indexing saturates all CPU cores and degrades search latency, throttle the worker threads:

```toml
max_indexing_threads = 2
```

> Ensures background indexing leaves CPU cores free to handle incoming search traffic.

### Open File Limits (ulimit)

Meilisearch opens multiple database and socket descriptors. Verify the system limit for the `meilisearch` user:

```bash
sudo su -s /bin/bash meilisearch -c "ulimit -n"
```

If the value is low (e.g. `1024`), ensure `/etc/systemd/system/meilisearch.service` includes:

```ini
[Service]
LimitNOFILE=65535
```

### Keep Web Dashboard Disabled in Production

In development mode (`env = "development"`), Meilisearch serves an interactive web dashboard at `/` where anyone with network access can test queries. 

In production mode (`env = "production"`), the web dashboard is automatically disabled for security, and requests to `/` return `{"status": "available"}` or require authorization. Keep `env = "production"` in your configuration.

---

## 11. Troubleshooting & Maintenance

### Check Service Status and Logs

If Meilisearch fails to start or crashes, inspect the systemd journal:

```bash
sudo journalctl -u meilisearch.service -n 50 --no-pager
```

> - `-u meilisearch.service` — Filters logs specifically for Meilisearch.
> - `-n 50` — Displays the last 50 log lines.
> - `--no-pager` — Outputs directly to the terminal without paging.

To follow logs in real time during testing:

```bash
sudo journalctl -u meilisearch.service -f
```

### Common Issues & Fixes

#### 1. Service fails to start with "Missing master key"
- **Cause**: In `production` mode, Meilisearch refuses to launch without a master key of at least 16 bytes.
- **Fix**: Check `/etc/meilisearch.toml` and ensure `master_key = "..."` is set with a string of 16 characters or longer.

#### 2. Nginx returns `413 Request Entity Too Large`
- **Cause**: Uploading a document batch larger than Nginx's default 1 MB limit.
- **Fix**: Add `client_max_body_size 100M;` inside the `server` or `location /` block in `/etc/nginx/sites-available/meilisearch.conf`, then run `sudo nginx -t && sudo systemctl reload nginx`.

#### 3. Search queries return `401 Unauthorized` / `Invalid API Key`
- **Cause**: Missing or incorrect `Authorization: Bearer <KEY>` header.
- **Fix**: Ensure your request includes `-H "Authorization: Bearer YOUR_KEY"`. For client applications, use the `Default Search API Key` retrieved from `GET /keys`.

#### 4. Server killed during heavy document indexing (Out of Memory)
- **Cause**: Linux kernel OOM killer terminated the process due to RAM exhaustion.
- **Fix**: Check `dmesg -T | grep -i oom` to verify. Lower `max_indexing_memory` in `/etc/meilisearch.toml` (e.g. `500MiB` or `1GiB`), enable a swap file (see [Section 1](#1-hardware--swap-preparation)), and reduce document batch sizes when sending updates from your application.

---

## 📚 Further Reading

- [Official Meilisearch Documentation](https://www.meilisearch.com/docs)
- [Meilisearch Configuration Guide](https://www.meilisearch.com/docs/learn/configuration/instance_options)
- [Security & API Keys Reference](https://www.meilisearch.com/docs/learn/security/master_api_keys)
- [Meilisearch SDKs & Integrations](https://www.meilisearch.com/docs/learn/what_is_meilisearch/sdks)
- [Meilisearch GitHub Repository](https://github.com/meilisearch/meilisearch)
