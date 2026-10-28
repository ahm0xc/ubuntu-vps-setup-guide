# 🟩 Node.js Setup Guide (via NVM)

A step-by-step guide to installing Node.js using NVM (Node Version Manager) on your Ubuntu VPS, plus how to run Node apps in production with PM2.

> Tested on **Ubuntu 22.04 LTS** and **Ubuntu 24.04 LTS**.

---

## Why NVM Instead of apt?

Ubuntu's `apt` package manager ships an outdated version of Node.js that quickly falls behind. NVM solves this by:

- Letting you install **any Node.js version** with a single command
- Letting you **switch between versions** per project (e.g., v18 for one app, v22 for another)
- Keeping Node **entirely in your home directory** — no `sudo` needed for global packages
- Making it easy to upgrade Node without reinstalling everything

---

## Prerequisites

- A VPS running Ubuntu 22.04 or 24.04 (ideally set up with the [VPS security guide](../README.md))
- A non-root sudo user (the guide uses `admin`)
- `curl` installed (`sudo apt install curl -y`)

---

## Table of Contents

1. [Remove Existing Node.js](#1-remove-existing-nodejs)
2. [Install NVM](#2-install-nvm)
3. [Install Node.js](#3-install-nodejs)
4. [Managing Node Versions](#4-managing-node-versions)
5. [Pin a Version Per Project (.nvmrc)](#5-pin-a-version-per-project-nvmrc)
6. [Install PM2 Process Manager](#6-install-pm2-process-manager)
7. [Run Your App with PM2](#7-run-your-app-with-pm2)
8. [PM2 Ecosystem Config File](#8-pm2-ecosystem-config-file)
9. [Survive Reboots with systemd](#9-survive-reboots-with-systemd)
10. [PM2 Log Rotation](#10-pm2-log-rotation)
11. [Essential PM2 Commands](#11-essential-pm2-commands)
12. [Updating Node.js](#12-updating-nodejs)

---

## 1. Remove Existing Node.js

If you previously installed Node.js via `apt`, remove it first to avoid conflicts between two Node installations on the same system.

```bash
sudo apt remove --purge nodejs npm -y
```

> - `remove --purge` — Uninstalls the packages and also deletes their configuration files (unlike plain `remove`, which leaves configs behind).
> - `nodejs npm` — Removes both the Node.js runtime and the npm package manager installed by `apt`.

```bash
sudo apt autoremove -y
```

> Cleans up any leftover dependency packages that were installed alongside Node.js but are no longer needed by anything else.

Confirm Node.js is gone:

```bash
node --version
```

> Should print `command not found`. If a version is still shown, there may be another Node installation (e.g., from a manual binary or a third-party PPA) that needs to be removed too.

---

## 2. Install NVM

NVM is installed as a shell script directly from its GitHub repository. It lives entirely in your home directory (`~/.nvm`) — no root access needed.

```bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
```

> - `curl -o-` — Downloads the install script and prints it to stdout (the `-o-` flag means "output to stdout" instead of a file).
> - `| bash` — Pipes the downloaded script directly into bash to run it.
> - The script clones the NVM repository into `~/.nvm` and appends a few lines to your shell profile (`~/.bashrc`) that load NVM every time you open a terminal.

> **Security note:** Piping a script from the internet directly into bash is convenient but means you're trusting the source. Always verify you're using the official NVM repository at `github.com/nvm-sh/nvm` and check the latest version number at [github.com/nvm-sh/nvm/releases](https://github.com/nvm-sh/nvm/releases).

Load NVM into your **current** shell session without logging out:

```bash
source ~/.bashrc
```

> Re-executes your `.bashrc` file in the current terminal. This applies the NVM lines the installer just added, making the `nvm` command available immediately without needing to log out and back in.

Verify NVM is installed:

```bash
nvm --version
```

> Prints the installed NVM version (e.g., `0.40.1`). If this outputs `command not found`, the `source ~/.bashrc` step didn't work — try closing and reopening your terminal.

---

## 3. Install Node.js

### Install the LTS version (recommended for production)

```bash
nvm install --lts
```

> - `nvm install` — Downloads and installs a Node.js version.
> - `--lts` — Installs the latest **Long Term Support** release. LTS versions receive security and bug fixes for 3+ years, making them the right choice for servers and production workloads. The "Current" release is for developers who want the newest features and is not recommended for production.

Set it as your default Node version:

```bash
nvm alias default lts/*
```

> Creates an alias called `default` pointing to the latest LTS version. NVM uses this alias to know which Node version to activate in every new terminal session. Without this, NVM might not activate any version automatically.

Verify the installation:

```bash
node --version
npm --version
```

> Prints the active Node.js and npm version numbers. Both commands should respond — if either says `command not found`, re-run `source ~/.bashrc`.

### Install a specific version (optional)

```bash
nvm install 20.18.0
```

> Installs an exact Node.js version. Useful when a project requires a specific version. You can find all available versions with `nvm ls-remote`.

---

## 4. Managing Node Versions

```bash
# List all locally installed versions
nvm ls
```

> Shows every Node.js version installed on your system. An arrow (`->`) marks the currently active version, and `default` is highlighted.

```bash
# List all available versions you can install
nvm ls-remote
```

> Fetches and displays every Node.js release available from the official registry. The list is long — pipe it through `grep lts` to see only LTS releases: `nvm ls-remote --lts`.

```bash
# Switch to a specific installed version for this session
nvm use 20
```

> Activates Node.js v20 in your current terminal session only. Other terminals continue using whatever version they had active. Accepts a full version number (`20.18.0`) or just the major version (`20`), which picks the latest installed patch.

```bash
# Check which version is currently active
nvm current
```

> Prints the version NVM currently has active (e.g., `v20.18.0`).

```bash
# Set a specific version as the new default
nvm alias default 20
```

> Updates the `default` alias so all new terminal sessions automatically use Node 20.

```bash
# Uninstall a version you no longer need
nvm uninstall 18
```

> Removes a locally installed Node.js version and its associated global packages. Frees up disk space.

---

## 5. Pin a Version Per Project (.nvmrc)

The `.nvmrc` file tells NVM which Node.js version a specific project requires. This prevents version drift between your local machine, staging server, and production server.

In your project's root directory:

```bash
node --version > .nvmrc
```

> Writes the currently active Node version (e.g., `v20.18.0`) into a file called `.nvmrc`. Commit this file to your git repository so every developer and every server uses the same Node version.

Whenever you `cd` into the project directory and want to switch to the pinned version:

```bash
nvm use
```

> Reads the `.nvmrc` file in the current directory (or any parent directory) and activates the matching Node version. If that version isn't installed yet, NVM will tell you — run `nvm install` to install the pinned version.

> **Tip:** Add this to the end of your `~/.bashrc` to have NVM automatically switch versions when you `cd` into a directory with a `.nvmrc`:
>
> ```bash
> # Auto-switch Node version when entering a directory with .nvmrc
> cdnvm() {
>     command cd "$@" || return $?
>     if [[ -f .nvmrc ]]; then
>         nvm use
>     fi
> }
> alias cd='cdnvm'
> ```

---

## 6. Install PM2 Process Manager

Running `node server.js` directly in a terminal is fine for development but not for production — the process dies the moment you close the terminal or the app crashes. PM2 is a production-grade process manager that keeps your app alive.

PM2 features:
- Automatically restarts your app if it crashes
- Starts your app on server boot
- Manages logs with rotation
- Runs your app across multiple CPU cores (cluster mode)
- Zero-downtime reloads for deployments

Install PM2 globally with npm:

```bash
npm install -g pm2
```

> - `npm install` — Installs a package.
> - `-g` — Global flag. Installs PM2 as a system-wide CLI tool rather than a local project dependency. Because NVM manages npm, this installs PM2 inside `~/.nvm/versions/node/<version>/bin/` — no `sudo` required.

Verify the installation:

```bash
pm2 --version
```

> Prints the PM2 version. Also confirms the global npm install path is correctly set up by NVM.

---

## 7. Run Your App with PM2

### Start a single script

```bash
pm2 start server.js --name "my-app"
```

> - `pm2 start` — Tells PM2 to launch a new managed process.
> - `server.js` — The entry point of your Node.js application. Replace with your actual file (e.g., `index.js`, `dist/main.js`).
> - `--name "my-app"` — Assigns a human-readable name to the process so you can reference it by name in other PM2 commands instead of by its numeric ID.

### Start with environment variables

```bash
pm2 start server.js --name "my-app" --env production
```

> Passes the `production` environment to the app. PM2 reads environment-specific variables from the ecosystem config file (see next section).

### View all running processes

```bash
pm2 list
```

> Displays a table of all PM2-managed processes — their name, ID, status (online/stopped/errored), CPU usage, memory usage, and uptime.

### View live logs

```bash
pm2 logs my-app
```

> Streams the stdout and stderr logs for `my-app` in real time. Press `Ctrl+C` to stop following. Use `pm2 logs` (without a name) to see logs from all processes combined.

```bash
pm2 logs my-app --lines 200
```

> Shows the last 200 lines of historical log output before streaming new ones. Useful when diagnosing a crash that already happened.

### Restart and stop

```bash
pm2 restart my-app
```

> Stops and restarts the process. There is a brief moment of downtime. Use `pm2 reload` instead for zero-downtime restarts in cluster mode.

```bash
pm2 stop my-app
```

> Stops the process but keeps it in PM2's list. You can bring it back with `pm2 start my-app`.

```bash
pm2 delete my-app
```

> Stops the process and removes it from PM2's list entirely.

---

## 8. PM2 Ecosystem Config File

For production, use an ecosystem config file instead of long command-line flags. It keeps all your app configuration in one place and supports multiple apps.

Create the file in your project root:

```bash
nano ecosystem.config.js
```

```javascript
module.exports = {
  apps: [
    {
      // Human-readable name for this process in PM2
      name: "my-app",

      // Path to your app's entry point (adjust for your framework)
      script: "./server.js",

      // Number of instances to run:
      // "max" uses all available CPU cores (best for production throughput)
      // Use 1 for single-instance apps or those that don't support clustering
      instances: "max",

      // "cluster" enables Node.js cluster mode — PM2 load-balances
      // incoming requests across all instances automatically
      exec_mode: "cluster",

      // Restart the app automatically if it crashes unexpectedly
      autorestart: true,

      // IMPORTANT: Keep this false in production.
      // File watching causes infinite restart loops when log files or
      // build artifacts change.
      watch: false,

      // Restart the process if it exceeds this memory threshold.
      // Catches slow memory leaks before they crash the server.
      max_memory_restart: "500M",

      // Environment variables for development (used when --env is not set)
      env: {
        NODE_ENV: "development",
        PORT: 3000,
      },

      // Environment variables for production (used with: pm2 start --env production)
      env_production: {
        NODE_ENV: "production",
        PORT: 3000,
      },

      // Log file paths (relative to your project directory)
      out_file: "./logs/out.log",    // stdout goes here
      error_file: "./logs/err.log",  // stderr goes here
      merge_logs: true,              // combine logs from all cluster instances into one file
      log_date_format: "YYYY-MM-DD HH:mm:ss Z", // timestamp format in log files
    },
  ],
};
```

Start your app using the ecosystem file:

```bash
pm2 start ecosystem.config.js --env production
```

> Reads the `ecosystem.config.js` file, launches the app with the `env_production` variables, and registers it with PM2 using all the settings defined in the file.

After updating your code and rebuilding:

```bash
pm2 reload ecosystem.config.js --env production --update-env
```

> - `reload` — In cluster mode, PM2 restarts instances **one at a time**, so at least one instance is always running. This gives you **zero downtime** during deployments — unlike `restart`, which kills all instances at once.
> - `--update-env` — Tells PM2 to re-read environment variables from the ecosystem file, picking up any new or changed values.

---

## 9. Survive Reboots with systemd

PM2 processes don't automatically come back after a server reboot unless you configure a systemd startup script. Do this once right after setting up PM2.

**Step 1 — Generate the startup script:**

```bash
pm2 startup
```

> Detects your operating system and prints a `sudo env PATH=...` command tailored to your system. The command hardcodes the **absolute path** to your Node.js binary (resolving NVM's symlinks) so systemd can find it without loading `.bashrc`.

**Step 2 — Copy and run the command PM2 prints.** It looks something like this:

```bash
sudo env PATH=$PATH:/home/admin/.nvm/versions/node/v20.18.0/bin pm2 startup systemd -u admin --hp /home/admin
```

> This command installs a systemd service unit that starts PM2 (and therefore all your saved processes) when the server boots. Never type this manually — always copy the exact output from `pm2 startup` since the Node.js path is specific to your installation.

**Step 3 — Save the current process list:**

```bash
pm2 save
```

> Saves a snapshot of all currently running PM2 processes to `~/.pm2/dump.pm2`. When the systemd service starts PM2 on boot, it restores this saved list automatically — reviving all your apps without any manual intervention.

Test it by rebooting and checking that your app comes back:

```bash
sudo reboot
# Wait ~30 seconds, then SSH back in
pm2 list
```

> After logging back in, `pm2 list` should show your app with status `online`. If it shows `stopped` or isn't listed, check `pm2 logs` for errors.

---

## 10. PM2 Log Rotation

By default, PM2 writes logs indefinitely. On a busy server, log files can grow to gigabytes and fill your disk. Install PM2's official log rotation module immediately.

```bash
pm2 install pm2-logrotate
```

> Installs the `pm2-logrotate` module — a PM2 plugin that automatically rotates log files on a schedule.

Configure the rotation settings:

```bash
pm2 set pm2-logrotate:max_size 50M
```

> Rotates a log file when it reaches 50 MB. A new file is started and the old one is archived.

```bash
pm2 set pm2-logrotate:retain 7
```

> Keeps 7 rotated log archives per app. Older archives are deleted automatically, capping how much disk space logs can consume.

```bash
pm2 set pm2-logrotate:compress true
```

> Compresses rotated log archives with gzip, significantly reducing their disk footprint.

```bash
pm2 set pm2-logrotate:dateFormat YYYY-MM-DD
```

> Appends the rotation date to archived log filenames (e.g., `out-2025-10-28.log`) so you can easily find logs from a specific day.

Confirm the settings were applied:

```bash
pm2 get pm2-logrotate
```

> Prints all current `pm2-logrotate` configuration values.

---

## 11. Essential PM2 Commands

### Monitoring

```bash
# Interactive real-time dashboard (CPU, memory, logs)
pm2 monit
```

> Opens a terminal UI dashboard showing CPU and memory usage for all processes, plus a live log stream. Press `q` to exit.

```bash
# Show detailed info about a specific app
pm2 show my-app
```

> Displays everything about an app — its config, environment variables, restart count, uptime, log file paths, and more. Useful for debugging.

```bash
# View logs for all apps
pm2 logs
```

> Streams combined real-time logs from every PM2-managed process, prefixed by app name.

```bash
# Flush (clear) all log files
pm2 flush
```

> Empties all log files for all apps. Useful after a messy debug session to start with clean logs.

### Process control

```bash
# Reload all apps with zero downtime (cluster mode only)
pm2 reload all
```

> Performs a rolling restart across all cluster instances of all apps simultaneously. At least one instance stays alive throughout the process.

```bash
# Stop all apps
pm2 stop all
```

> Gracefully stops every PM2-managed process. Processes remain in the list and can be restarted.

```bash
# Delete all apps from PM2
pm2 delete all
```

> Removes all processes from PM2's management. Does not affect your application files.

```bash
# Resurrect previously saved processes
pm2 resurrect
```

> Restores the process list from the last `pm2 save` snapshot. Useful if PM2 loses its process list.

---

## 12. Updating Node.js

### Update to a new LTS version

```bash
# Install the new LTS version
nvm install --lts
```

> Downloads and installs the latest LTS release alongside your existing version. NVM keeps all versions independently — your old version and its global packages are untouched.

```bash
# Reinstall global packages (like pm2) from the old version into the new one
nvm install --lts --reinstall-packages-from=default
```

> The `--reinstall-packages-from=default` flag reads the list of globally installed npm packages from the `default` alias (your old version) and installs them into the new version automatically. This saves you from manually reinstalling `pm2` and other global tools.

```bash
# Set the new version as default
nvm alias default lts/*
```

> Updates the `default` alias to point to the new LTS. All new terminal sessions will use it.

```bash
# Verify the active version
node --version
```

> Confirm you're now running the new version.

```bash
# Remove the old version once confirmed working
nvm uninstall 20
```

> Frees up disk space by deleting the old Node version. Only do this after you've verified your apps work correctly on the new version.

### Update PM2 itself

```bash
npm install -g pm2@latest
pm2 update
```

> - `npm install -g pm2@latest` — Installs the newest PM2 version globally.
> - `pm2 update` — Signals the running PM2 daemon to reload itself with the new binary, without killing your apps.

---

## Production Checklist

| Task | Done |
|------|------|
| Old `apt` Node.js installation removed | ✅ |
| NVM installed and loaded in `.bashrc` | ✅ |
| LTS Node.js installed and set as default | ✅ |
| `.nvmrc` file committed to the project repo | ✅ |
| PM2 installed globally | ✅ |
| App running via `ecosystem.config.js` | ✅ |
| Resource limits set (`max_memory_restart`) | ✅ |
| `pm2 startup` systemd service configured | ✅ |
| `pm2 save` run to persist process list | ✅ |
| `pm2-logrotate` installed and configured | ✅ |

---

## 📚 Further Reading

- [NVM GitHub Repository](https://github.com/nvm-sh/nvm)
- [Node.js LTS Release Schedule](https://nodejs.org/en/about/previous-releases)
- [PM2 Documentation](https://pm2.keymetrics.io/docs/usage/quick-start/)
- [PM2 Ecosystem File Reference](https://pm2.keymetrics.io/docs/usage/application-declaration/)
- [PM2 Log Rotate](https://github.com/keymetrics/pm2-logrotate)
