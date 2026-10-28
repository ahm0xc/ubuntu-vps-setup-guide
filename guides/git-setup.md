# 🔀 Git Setup Guide

A complete guide to installing and configuring Git on your Ubuntu VPS, setting up SSH authentication with GitHub/GitLab, and establishing a solid workflow.

> Tested on **Ubuntu 22.04 LTS** and **Ubuntu 24.04 LTS**.

---

## Table of Contents

1. [Install Git](#1-install-git)
2. [Configure Git Identity](#2-configure-git-identity)
3. [Generate an SSH Key for Git Hosting](#3-generate-an-ssh-key-for-git-hosting)
4. [Add the SSH Key to GitHub / GitLab](#4-add-the-ssh-key-to-github--gitlab)
5. [Test the SSH Connection](#5-test-the-ssh-connection)
6. [Clone a Repository](#6-clone-a-repository)
7. [Essential Git Configuration](#7-essential-git-configuration)
8. [Managing Multiple Git Identities](#8-managing-multiple-git-identities)
9. [Useful Git Aliases](#9-useful-git-aliases)
10. [Git Credential Helper (HTTPS fallback)](#10-git-credential-helper-https-fallback)
11. [Git Cheatsheet](#11-git-cheatsheet)

---

## 1. Install Git

Git is available in the default Ubuntu repositories. Install it with:

```bash
sudo apt update && sudo apt install git -y
```

> - `apt update` — Refreshes the local package index before installing.
> - `apt install git -y` — Installs Git and auto-confirms the prompt.

Verify the installation and check the version:

```bash
git --version
```

> Should output something like `git version 2.43.0`. Any version ≥ 2.28 is fine for modern workflows.

---

## 2. Configure Git Identity

Git attaches your name and email to every commit you make. Set these globally so they apply to all repositories on this machine:

```bash
git config --global user.name "Your Name"
```

> Sets your display name. Use the same name as your GitHub/GitLab profile for consistency.

```bash
git config --global user.email "you@example.com"
```

> Sets your email. This should match the email linked to your GitHub/GitLab account, otherwise contributions won't be attributed to your profile correctly.

Set your preferred default branch name (modern default is `main`):

```bash
git config --global init.defaultBranch main
```

> When you run `git init`, the initial branch will be named `main` instead of the old default `master`.

Verify your configuration:

```bash
git config --global --list
```

> Prints all global Git settings. Config is stored in `~/.gitconfig`.

---

## 3. Generate an SSH Key for Git Hosting

Using SSH keys with GitHub/GitLab is more secure than HTTPS passwords and doesn't require entering credentials on every push or pull.

Generate a dedicated key pair for Git hosting:

```bash
ssh-keygen -t ed25519 -C "you@example.com" -f ~/.ssh/id_ed25519_github
```

> - `-t ed25519` — Modern elliptic-curve algorithm; faster and more secure than RSA.
> - `-C "you@example.com"` — A comment to identify this key (use your Git hosting email).
> - `-f ~/.ssh/id_ed25519_github` — Saves the key with a descriptive filename instead of the default, so it doesn't clash with your server SSH key.
> - You'll be prompted for an optional **passphrase** — strongly recommended to protect the private key.

This creates:
- `~/.ssh/id_ed25519_github` — your **private key**. Never share this.
- `~/.ssh/id_ed25519_github.pub` — your **public key**. This goes on GitHub/GitLab.

Start the SSH agent and add your new key so it's available for all Git operations:

```bash
eval "$(ssh-agent -s)"
```

> Starts the SSH agent in the background. The agent holds your decrypted private key in memory so you only need to enter your passphrase once per session.

```bash
ssh-add ~/.ssh/id_ed25519_github
```

> Adds the private key to the running SSH agent.

**Make this persistent across sessions** by adding to your shell profile:

```bash
nano ~/.bashrc
```

Add these lines at the bottom:

```bash
# Start SSH agent and add Git key if not already loaded
if [ -z "$SSH_AUTH_SOCK" ]; then
    eval "$(ssh-agent -s)" > /dev/null
    ssh-add ~/.ssh/id_ed25519_github 2>/dev/null
fi
```

> This block checks if an SSH agent is already running before starting a new one — avoids spawning duplicate agents on every new shell session.

```bash
source ~/.bashrc
```

> Reloads your shell profile so the changes take effect immediately.

---

## 4. Add the SSH Key to GitHub / GitLab

Print your public key to copy it:

```bash
cat ~/.ssh/id_ed25519_github.pub
```

> Copy the entire output — it starts with `ssh-ed25519` and ends with your email comment.

### GitHub

1. Go to **GitHub → Settings → SSH and GPG keys → New SSH key**
2. Give it a descriptive title (e.g. `ubuntu-vps`)
3. Set **Key type** to `Authentication Key`
4. Paste the public key and click **Add SSH key**

### GitLab

1. Go to **GitLab → User Settings → SSH Keys → Add new key**
2. Paste the public key in the **Key** field
3. Give it a **Title** and optionally set an **Expiration date**
4. Click **Add key**

---

## 5. Test the SSH Connection

Verify the key is accepted by your Git host:

```bash
# For GitHub
ssh -T git@github.com
```

> If successful, you'll see: `Hi username! You've successfully authenticated, but GitHub does not provide shell access.`

```bash
# For GitLab
ssh -T git@gitlab.com
```

> If successful: `Welcome to GitLab, @username!`

If the test fails, debug with verbose output:

```bash
ssh -vT git@github.com
```

> `-v` enables verbose mode and prints each step of the connection handshake. Look for lines mentioning your key file to diagnose issues.

---

## 6. Clone a Repository

With SSH configured, clone using the SSH URL (not HTTPS):

```bash
git clone git@github.com:username/repo-name.git
```

> - `git clone` — Downloads the entire repository including its history.
> - Use the SSH URL (`git@github.com:...`) not the HTTPS URL (`https://github.com/...`) so your SSH key is used for authentication.

Clone into a specific directory name:

```bash
git clone git@github.com:username/repo-name.git my-project
```

> Clones the repo into a folder called `my-project` instead of `repo-name`.

---

## 7. Essential Git Configuration

### Configure the default pull strategy

Avoid the noisy "divergent branches" warning on `git pull`:

```bash
git config --global pull.rebase false
```

> Sets pull to use **merge** strategy (the traditional behaviour). Alternatively use `pull.rebase true` if you prefer a linear history via rebasing.

### Set a global `.gitignore`

Create a global ignore file to keep system/editor files out of every repository:

```bash
nano ~/.gitignore_global
```

Add common patterns:

```gitignore
# macOS
.DS_Store
.AppleDouble
.LSOverride

# Linux
*~
.fuse_hidden*
.Trash-*

# Editor files
.vscode/
.idea/
*.swp
*.swo
*.sublime-project
*.sublime-workspace

# Environment files
.env
.env.local
.env.*.local

# Logs
*.log
npm-debug.log*

# Dependency directories
node_modules/
__pycache__/
*.pyc
```

Register it with Git:

```bash
git config --global core.excludesfile ~/.gitignore_global
```

> Tells Git to always consult this file when deciding what to ignore, in addition to any per-project `.gitignore`.

### Set your preferred editor

```bash
git config --global core.editor nano
```

> Used when Git needs you to write a message (e.g. `git commit` without `-m`, `git rebase -i`). Replace `nano` with `vim`, `nvim`, or your editor of choice.

### Enable coloured output

```bash
git config --global color.ui auto
```

> Turns on coloured terminal output for `git diff`, `git log`, `git status`, etc., making them much easier to read.

---

## 8. Managing Multiple Git Identities

If you work with multiple GitHub accounts (e.g. personal and work), use SSH config to route each host through a different key.

Edit your SSH config:

```bash
nano ~/.ssh/config
```

Add entries like this:

```
# Personal GitHub account
Host github-personal
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github

# Work GitHub account
Host github-work
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github_work
```

> - `Host` — A nickname you choose. You'll use this in your clone URL instead of `github.com`.
> - `HostName` — The real hostname to connect to.
> - `IdentityFile` — Which SSH key to use for this alias.

Clone using the alias instead of `github.com`:

```bash
# Personal repo
git clone git@github-personal:personal-username/repo.git

# Work repo
git clone git@github-work:work-org/repo.git
```

For existing repos, update the remote URL:

```bash
git remote set-url origin git@github-personal:personal-username/repo.git
```

> Changes the `origin` remote to use your SSH config alias, so the correct key is used when pushing and pulling.

---

## 9. Useful Git Aliases

Add shortcuts for common commands to your global config:

```bash
git config --global alias.st status
git config --global alias.co checkout
git config --global alias.br branch
git config --global alias.ci commit
git config --global alias.lg "log --oneline --graph --decorate --all"
git config --global alias.undo "reset HEAD~1 --mixed"
git config --global alias.unstage "reset HEAD --"
```

Usage examples:

```bash
git st          # git status
git co main     # git checkout main
git lg          # pretty log tree
git undo        # undo the last commit (keeps changes staged)
git unstage .   # unstage all staged changes
```

> Aliases are stored in `~/.gitconfig` under the `[alias]` section. You can edit them directly there too.

---

## 10. Git Credential Helper (HTTPS fallback)

If you ever need to use HTTPS (e.g. for a host that doesn't support SSH), store credentials so you're not prompted every time:

```bash
git config --global credential.helper store
```

> Saves credentials in plaintext to `~/.git-credentials`. Convenient but less secure — use SSH keys whenever possible.

For a more secure option that caches credentials in memory temporarily:

```bash
git config --global credential.helper "cache --timeout=3600"
```

> Caches credentials in memory for 3600 seconds (1 hour). After the timeout, you'll be prompted again.

---

## 11. Git Cheatsheet

### Basic Workflow

```bash
git init                        # Initialize a new local repository
git clone <url>                 # Clone a remote repository
git status                      # Show working tree status
git add <file>                  # Stage a specific file
git add .                       # Stage all changes in current directory
git commit -m "message"         # Commit staged changes with a message
git push origin main            # Push commits to remote branch
git pull                        # Fetch and merge remote changes
```

### Branching

```bash
git branch                      # List local branches
git branch feature/my-feature   # Create a new branch
git checkout feature/my-feature # Switch to the branch
git checkout -b feature/new     # Create and switch in one step
git merge feature/my-feature    # Merge branch into current branch
git branch -d feature/my-feature# Delete a merged branch
git push origin --delete feature/my-feature  # Delete remote branch
```

### Inspecting History

```bash
git log                         # Full commit history
git log --oneline               # Condensed one-line history
git log --oneline --graph       # Visual branch/merge graph
git diff                        # Unstaged changes
git diff --staged               # Staged changes (vs last commit)
git show <commit-hash>          # Show a specific commit's changes
git blame <file>                # Show who last modified each line
```

### Undoing Changes

```bash
git restore <file>              # Discard unstaged changes to a file
git restore --staged <file>     # Unstage a file (keep changes)
git revert <commit-hash>        # Create a new commit that undoes a past commit
git reset HEAD~1 --soft         # Undo last commit, keep changes staged
git reset HEAD~1 --mixed        # Undo last commit, keep changes unstaged
git reset HEAD~1 --hard         # ⚠️ Undo last commit and discard all changes
```

> ⚠️ `git reset --hard` is destructive and permanently discards uncommitted changes. Use with caution.

### Remote Management

```bash
git remote -v                   # List remotes with URLs
git remote add origin <url>     # Add a remote called "origin"
git remote set-url origin <url> # Change the URL of "origin"
git fetch origin                # Download remote changes without merging
git push -u origin main         # Push and set upstream tracking
```

### Stashing

```bash
git stash                       # Temporarily shelve uncommitted changes
git stash pop                   # Apply and remove the latest stash
git stash list                  # List all stashes
git stash apply stash@{1}       # Apply a specific stash without removing it
git stash drop stash@{0}        # Delete a specific stash
```

---

> **Tip:** Always use SSH URLs (`git@github.com:...`) instead of HTTPS for passwordless, secure authentication from your VPS.
