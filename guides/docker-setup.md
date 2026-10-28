# 🐳 Docker Setup Guide

A step-by-step guide to installing Docker Engine on Ubuntu, securing the daemon, and learning essential day-to-day commands.

> Tested on **Ubuntu 22.04 LTS** and **Ubuntu 24.04 LTS**.

---

## What Is Docker?

Docker is a platform for running applications inside **containers** — lightweight, isolated environments that bundle your app and all its dependencies together. Unlike a virtual machine, containers share the host OS kernel, making them fast to start and resource-efficient.

Key concepts:
- **Image** — A read-only template (e.g., `nginx`, `postgres`, `node`). Think of it as a snapshot.
- **Container** — A running instance of an image. You can run many containers from the same image.
- **Volume** — Persistent storage that survives container restarts.
- **Network** — A virtual network that lets containers talk to each other.
- **Compose** — A tool for defining and running multi-container apps from a single `docker-compose.yml` file.

---

## Prerequisites

- A VPS running Ubuntu 22.04 or 24.04 (ideally set up with the [VPS security guide](../README.md))
- A non-root sudo user (the guide uses `admin`)
- Internet access on the server

---

## Table of Contents

1. [Remove Old Docker Versions](#1-remove-old-docker-versions)
2. [Install Docker Engine](#2-install-docker-engine)
3. [Verify the Installation](#3-verify-the-installation)
4. [Run Docker Without sudo](#4-run-docker-without-sudo)
5. [Harden the Docker Daemon](#5-harden-the-docker-daemon)
6. [Essential Docker Commands](#6-essential-docker-commands)
7. [Docker Compose](#7-docker-compose)
8. [Managing Volumes](#8-managing-volumes)
9. [Managing Networks](#9-managing-networks)
10. [Keeping Docker Clean](#10-keeping-docker-clean)
11. [Security Notes](#11-security-notes)

---

## 1. Remove Old Docker Versions

Ubuntu's default repositories include older, unofficial Docker packages (`docker.io`, `docker-compose`) that conflict with the official Docker Engine. Remove them first.

```bash
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do
  sudo apt remove $pkg -y
done
```

> Loops over a list of conflicting package names and removes each one. It's safe to run even if none of them are installed — `apt remove` simply skips packages that aren't present.

---

## 2. Install Docker Engine

Install from Docker's official APT repository so you always get the latest stable version and future updates via `apt upgrade`.

**Step 1 — Install prerequisites:**

```bash
sudo apt update
```

> Refreshes the package index before installing anything.

```bash
sudo apt install -y ca-certificates curl
```

> - `ca-certificates` — A bundle of trusted SSL certificate authorities. Required to verify the HTTPS connection to Docker's servers.
> - `curl` — A command-line tool for downloading files over HTTP/HTTPS. Used to fetch Docker's GPG signing key.

**Step 2 — Add Docker's GPG signing key:**

```bash
sudo install -m 0755 -d /etc/apt/keyrings
```

> Creates the `/etc/apt/keyrings` directory with permissions `0755` (readable by all, writable by root only). This is the standard location for APT repository signing keys on modern Ubuntu.

```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
```

> Downloads Docker's official GPG public key and saves it to the keyrings directory. APT uses this key to verify that packages you download actually came from Docker and haven't been tampered with.

```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

> Makes the key file readable by all users (`a+r` = add read permission for all). APT runs as an unprivileged user internally and needs to read the key during package verification.

**Step 3 — Add Docker's APT repository:**

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

> Constructs and writes the APT source entry for Docker's repository:
> - `arch=$(dpkg --print-architecture)` — Automatically detects your CPU architecture (e.g., `amd64` for 64-bit Intel/AMD, `arm64` for ARM).
> - `signed-by=...` — Tells APT to verify packages using Docker's key.
> - `$(. /etc/os-release && echo "$VERSION_CODENAME")` — Reads your Ubuntu version codename (e.g., `jammy` for 22.04, `noble` for 24.04) so the correct repository branch is used.
> - `stable` — Pulls from the stable release channel (as opposed to `nightly` or `test`).

**Step 4 — Install Docker Engine:**

```bash
sudo apt update
```

> Refreshes the package index again, this time including Docker's newly added repository.

```bash
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

> Installs the full Docker stack:
> - `docker-ce` — The Docker **Community Edition** daemon (the core engine that runs containers).
> - `docker-ce-cli` — The `docker` command-line client used to talk to the daemon.
> - `containerd.io` — The low-level container runtime that Docker Engine sits on top of.
> - `docker-buildx-plugin` — Extends `docker build` with advanced features like multi-platform image building.
> - `docker-compose-plugin` — Installs Docker Compose as a Docker CLI plugin (`docker compose`), replacing the older standalone `docker-compose` binary.

---

## 3. Verify the Installation

Check that the Docker daemon started successfully:

```bash
sudo systemctl status docker
```

> Displays the status of the Docker systemd service. Look for `Active: active (running)` in green. Docker is also set to start on boot automatically after installation.

Run the canonical test container:

```bash
sudo docker run hello-world
```

> - `docker run` — Creates and starts a new container.
> - `hello-world` — A tiny official image that simply prints a confirmation message and exits.
> If you see `Hello from Docker!` in the output, everything is working correctly.

Check the installed versions:

```bash
docker --version
docker compose version
```

> Prints the version numbers of the Docker CLI and Compose plugin. Useful for verifying you have the expected versions and for bug reports.

---

## 4. Run Docker Without sudo

By default, the `docker` command requires `sudo` because the Docker daemon runs as root. Adding your user to the `docker` group lets you use Docker without prefixing every command with `sudo`.

```bash
sudo usermod -aG docker admin
```

> Adds the `admin` user to the `docker` group. Members of this group can communicate with the Docker socket (`/var/run/docker.sock`) without needing root. Replace `admin` with your actual username.

Apply the group change to your current shell session without logging out:

```bash
newgrp docker
```

> Starts a new shell session with the updated group membership. After this, your current terminal session will have `docker` group access.

Test that it works without sudo:

```bash
docker run hello-world
```

> If this runs without a "permission denied" error, Docker is correctly configured for your user.

> ⚠️ **Security note:** The `docker` group grants effective root access to the host machine. Only add trusted users to it. See [Security Notes](#11-security-notes) for more detail.

---

## 5. Harden the Docker Daemon

Docker's default configuration is suitable for development but needs tightening for a production VPS. The daemon's behaviour is controlled via `/etc/docker/daemon.json`.

```bash
sudo nano /etc/docker/daemon.json
```

> Creates or opens the Docker daemon configuration file. If the file doesn't exist, Docker uses built-in defaults. Any settings you define here override those defaults globally for all containers.

Paste the following configuration:

```json
{
  "live-restore": true,
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 64000,
      "Soft": 64000
    }
  },
  "no-new-privileges": true
}
```

**Each setting explained:**

- **`live-restore: true`** — Keeps containers running even if the Docker daemon itself is restarted (e.g., during a `docker` package upgrade). Without this, every daemon restart kills all running containers.
- **`log-driver: "json-file"`** — Writes container logs to JSON files on disk. The default is also `json-file`, but explicitly setting it ensures the options below are applied.
- **`log-opts.max-size: "10m"`** — Caps each container log file at 10 MB. Without a size limit, a verbose container can fill your disk and crash the whole server.
- **`log-opts.max-file: "3"`** — Keeps a maximum of 3 rotated log files per container (so up to 30 MB total per container). Older files are deleted automatically.
- **`default-ulimits.nofile`** — Sets the maximum number of open file descriptors for all containers to 64,000. This prevents a runaway container from exhausting file handles and affecting other services on the host.
- **`no-new-privileges: true`** — Prevents any process inside a container from gaining additional privileges via `setuid` or `setgid` binaries. Blocks a common container privilege-escalation technique.

Apply the new configuration:

```bash
sudo systemctl daemon-reload
```

> Reloads `systemd`'s knowledge of service unit files. Required after changing daemon-level config so systemd picks up the changes.

```bash
sudo systemctl restart docker
```

> Restarts the Docker daemon, applying the new `daemon.json` settings. Running containers are kept alive because we set `live-restore: true`.

Confirm Docker is still healthy:

```bash
sudo systemctl status docker
```

---

## 6. Essential Docker Commands

### Images

```bash
# Download an image from Docker Hub without running it
docker pull nginx:latest
```
> `pull` fetches the image and stores it locally. `nginx` is the image name, `latest` is the tag (version). Specify a version tag like `nginx:1.27` for reproducibility in production.

```bash
# List all locally stored images
docker images
```
> Shows every image on your system — name, tag, image ID, creation date, and size.

```bash
# Remove an image you no longer need
docker rmi nginx:latest
```
> `rmi` (remove image) deletes the image from local storage. Fails if a container (even a stopped one) is still using the image — remove the container first.

---

### Containers

```bash
# Run a container in the foreground (interactive)
docker run -it ubuntu:22.04 bash
```
> - `-i` — Keeps STDIN open (interactive mode).
> - `-t` — Allocates a pseudo-TTY (terminal). Combined, `-it` gives you an interactive shell inside the container.
> - `bash` — The command to run inside the container. You get a shell prompt inside Ubuntu.

```bash
# Run a container in the background (detached)
docker run -d --name my-app nginx:latest
```
> - `-d` — Detached mode. The container runs in the background and you get your terminal back immediately.
> - `--name my-app` — Gives the container a human-readable name. Without this, Docker assigns a random name like `quirky_tesla`.

```bash
# Map a host port to a container port
docker run -d --name my-web -p 8080:80 nginx:latest
```
> - `-p 8080:80` — Maps port `8080` on your VPS to port `80` inside the container. Visit `http://YOUR_SERVER_IP:8080` to reach Nginx. Format is always `HOST_PORT:CONTAINER_PORT`.

```bash
# List running containers
docker ps
```
> Shows all currently running containers — their ID, image, name, status, and port mappings.

```bash
# List all containers, including stopped ones
docker ps -a
```
> `-a` (all) includes containers that have exited. Useful to see what ran before and to clean up.

```bash
# View a container's live log output
docker logs -f my-app
```
> Prints the stdout/stderr of the container named `my-app`. `-f` streams new lines in real time (like `tail -f`). Press `Ctrl+C` to stop following.

```bash
# Open a shell inside a running container
docker exec -it my-app bash
```
> `exec` runs a command inside an **already running** container. `-it` gives you an interactive terminal. Replace `bash` with `sh` if the container image doesn't have bash installed.

```bash
# Stop a running container gracefully
docker stop my-app
```
> Sends a `SIGTERM` signal to the container's main process, giving it time to shut down cleanly. If it doesn't stop within 10 seconds, Docker sends `SIGKILL`.

```bash
# Start a stopped container
docker start my-app
```
> Restarts a container that was previously stopped. The container keeps its name, port mappings, and volume mounts from when it was created.

```bash
# Remove a stopped container
docker rm my-app
```
> Permanently deletes the container. The image it was based on is not affected. Use `-f` to force-remove a running container (equivalent to stop + rm).

```bash
# Inspect detailed container metadata
docker inspect my-app
```
> Outputs a large JSON object with everything about the container — IP address, mounts, environment variables, network config, restart policy, and more. Useful for debugging.

```bash
# Show live CPU, memory, and network usage
docker stats
```
> A live-updating table (like `top`) showing resource consumption for all running containers. Press `Ctrl+C` to exit.

---

## 7. Docker Compose

Docker Compose lets you define an entire multi-container application in a single `docker-compose.yml` file and manage it with simple commands.

### Example: a web app with a database

Create a project directory and compose file:

```bash
mkdir my-app && cd my-app
nano docker-compose.yml
```

```yaml
version: "3.8"

services:

  web:
    image: nginx:latest
    # Map port 8080 on the host to port 80 in the container
    ports:
      - "8080:80"
    # Mount a local directory into the container for config/content
    volumes:
      - ./html:/usr/share/nginx/html:ro
    # Ensure web only starts after the db is up
    depends_on:
      - db
    # Restart the container automatically if it crashes
    restart: unless-stopped
    # Connect to the shared app network
    networks:
      - app-network
    # Limit resources so this container can't starve the host
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: 256M

  db:
    image: postgres:16
    # Inject environment variables (credentials, database name)
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: mydb
    # Persist the database files in a named volume
    volumes:
      - db-data:/var/lib/postgresql/data
    restart: unless-stopped
    networks:
      - app-network

# Named volumes are managed by Docker and persist across container restarts
volumes:
  db-data:

# A shared bridge network so the containers can reach each other by service name
networks:
  app-network:
    driver: bridge
```

**Start the entire stack:**

```bash
docker compose up -d
```

> - `compose up` — Creates and starts all services defined in `docker-compose.yml`.
> - `-d` — Detached mode: runs everything in the background.

**View logs for all services:**

```bash
docker compose logs -f
```

> Streams combined logs from all services. Prefix each line with the service name. Use `docker compose logs -f web` to follow only one service.

**View running services:**

```bash
docker compose ps
```

> Lists the containers managed by this Compose project and their current status.

**Stop everything without removing containers:**

```bash
docker compose stop
```

> Gracefully stops all running containers in the project. Data in volumes is preserved. Use `docker compose start` to bring them back up.

**Stop and remove containers (keep volumes):**

```bash
docker compose down
```

> Stops and removes all containers and networks created by `compose up`. Named volumes are preserved by default, so your data isn't lost.

**Stop and remove everything including volumes:**

```bash
docker compose down -v
```

> The `-v` flag also removes named volumes. Use with care — this **permanently deletes all data** stored in those volumes.

**Restart a single service after a config change:**

```bash
docker compose restart web
```

> Restarts only the `web` service without touching the `db` service.

**Rebuild images and recreate containers:**

```bash
docker compose up -d --build
```

> Forces Docker to rebuild any images defined with a `build:` block in the compose file, then recreates the containers. Use this after changing a `Dockerfile`.

---

## 8. Managing Volumes

Volumes are Docker-managed persistent storage. Data written to a volume survives container restarts and removals.

```bash
# Create a named volume
docker volume create my-data
```
> Registers a new named volume with Docker. The actual data is stored in `/var/lib/docker/volumes/my-data/_data` on the host.

```bash
# List all volumes
docker volume ls
```
> Shows every volume on the system — both named volumes and anonymous ones.

```bash
# Inspect a volume (find its location on disk)
docker volume inspect my-data
```
> Prints details about the volume, including the `Mountpoint` — the actual path on the host where the data lives.

```bash
# Remove a volume (only works if no container is using it)
docker volume rm my-data
```
> Permanently deletes the volume and all its data. Will error if a container (even stopped) still references it.

```bash
# Remove all unused volumes to free disk space
docker volume prune
```
> Deletes all volumes not currently referenced by any container. Prompts for confirmation before deleting.

---

## 9. Managing Networks

Docker networks let containers communicate with each other using service names as hostnames instead of hardcoded IP addresses.

```bash
# Create a custom bridge network
docker network create my-network
```
> Creates a new isolated network. Containers connected to the same network can reach each other by container name.

```bash
# Run a container attached to your network
docker run -d --name my-app --network my-network nginx:latest
```
> Connects the container to `my-network` at startup. Other containers on the same network can now reach it at `http://my-app`.

```bash
# List all networks
docker network ls
```
> Shows all networks — the built-in ones (`bridge`, `host`, `none`) and any custom ones you've created.

```bash
# Remove an unused network
docker network rm my-network
```
> Deletes the network. Fails if any container is still connected to it.

---

## 10. Keeping Docker Clean

Docker aggressively caches layers, stopped containers, unused images, and dangling volumes. Without regular cleanup, these can silently consume gigabytes of disk space.

```bash
# Remove all stopped containers, unused networks,
# dangling images, and build cache in one go
docker system prune
```
> The safest bulk cleanup command. It prompts for confirmation and tells you how much space will be freed before doing anything. Does **not** remove named volumes or images that have a tag.

```bash
# Same as above but also removes all unused images
# (not just dangling ones) — more aggressive
docker system prune -a
```
> Removes everything `prune` does, plus any image not currently used by a running or stopped container. Use this when you want to do a full clean slate.

```bash
# Check how much disk space Docker is using
docker system df
```
> Breaks down Docker's disk usage by category: images, containers, volumes, and build cache. Shows both total size and reclaimable size.

---

## 11. Security Notes

### The Docker socket is root access

The Docker socket (`/var/run/docker.sock`) gives full control over all containers on the host. Anyone who can run `docker` commands can mount the host filesystem and escape to root. This means:

- **Only add trusted users to the `docker` group.** Adding a user to `docker` is equivalent to giving them passwordless `sudo`.
- **Never expose the Docker socket over TCP** without mutual TLS authentication.
- **Never mount `/var/run/docker.sock` into a container** unless you absolutely need it (e.g., a CI runner) and understand the risk.

### Set resource limits on every container

An unconstrained container can consume all available CPU and memory on the host, starving other services or crashing the server. Always set limits:

```bash
docker run -d \
  --name my-app \
  --memory="512m" \
  --memory-swap="512m" \
  --cpus="1.0" \
  nginx:latest
```

> - `--memory="512m"` — Hard cap on RAM usage. The container is killed if it tries to exceed this.
> - `--memory-swap="512m"` — Setting swap equal to memory effectively disables swap for this container, preventing it from thrashing disk.
> - `--cpus="1.0"` — Limits the container to one CPU core's worth of processing time.

### Run containers as non-root

By default, processes inside a container run as `root` inside that container. If a container is compromised, this can make it easier to attack the host. Use the `--user` flag or set a `USER` in your `Dockerfile`:

```bash
docker run -d --user 1000:1000 nginx:latest
```

> Runs the container process as UID 1000 (a non-root user). This limits what the process can do if it escapes the container.

### Audit your Docker setup

Run Docker Bench for Security — a script based on the CIS Docker Benchmark — to scan your installation for common misconfigurations:

```bash
git clone https://github.com/docker/docker-bench-security.git
cd docker-bench-security
sudo sh docker-bench-security.sh
```

> Checks your daemon config, file permissions, network settings, and running containers against industry best practices. Items marked `WARN` should be reviewed and addressed.

---

## 📚 Further Reading

- [Docker Engine Install Docs (Official)](https://docs.docker.com/engine/install/ubuntu/)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)
- [Docker Security Best Practices](https://docs.docker.com/engine/security/)
- [Play with Docker (browser-based sandbox)](https://labs.play-with-docker.com/)
- [Docker Hub (official image registry)](https://hub.docker.com/)
