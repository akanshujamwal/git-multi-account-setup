# Git Multi-Account Setup

> Manage multiple GitHub accounts on a single machine using SSH key pairs and host aliases — no credential conflicts, no accidental cross-account commits.

<p align="left">
  <img src="https://img.shields.io/badge/Git-F05032?style=for-the-badge&logo=git&logoColor=white" />
  <img src="https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white" />
  <img src="https://img.shields.io/badge/SSH-ED25519-4D4D4D?style=for-the-badge" />
  <img src="https://img.shields.io/badge/macOS%20%2F%20Linux-supported-green?style=for-the-badge" />
</p>

---

## 📋 Table of Contents

- [Overview](#-overview)
- [Account Setup](#-account-setup)
- [Step 1 — Generate SSH Keys](#step-1--generate-ssh-keys)
- [Step 2 — Add Public Keys to GitHub](#step-2--add-public-keys-to-github)
- [Step 3 — Configure SSH](#step-3--configure-ssh)
- [Step 4 — Test Connections](#step-4--test-connections)
- [Step 5 — Clone a Repository](#step-5--clone-a-repository)
- [Step 6 — Push a New Local Project](#step-6--push-a-new-local-project)
- [Step 7 — Feature Branch Workflow](#step-7--feature-branch-workflow)
- [Daily Workflow Checklist](#-daily-workflow-checklist)
- [Common Mistakes & Fixes](#-common-mistakes--fixes)
- [Troubleshooting](#-troubleshooting)
- [Live Guide](#-live-guide)

---

## 🧭 Overview

When two GitHub accounts share one machine, SSH sends the **wrong key** by default — causing authentication failures or commits attributed to the wrong account.

**The solution:** Create two named SSH host aliases in `~/.ssh/config`. Each alias points to one key. Every `git clone` and `git remote` URL uses the alias instead of `github.com`.

```
git push  →  SSH alias (github-personal)  →  ~/.ssh/config  →  correct key sent
```

This guide uses **Personal** and **Work** as the two account types. The exact same approach works for any two (or more) GitHub accounts.

---

## 👤 Account Setup

Replace the placeholders below with your own values before running any commands.

| Account  | Username | Email | SSH Host Alias | Key File |
|----------|----------|-------|----------------|----------|
| Personal | `your-personal-username` | `personal@email.com` | `github-personal` | `~/.ssh/id_ed25519_personal` |
| Work     | `your-work-username`     | `you@company.com`    | `github-work`     | `~/.ssh/id_ed25519_work`     |

> You can rename the aliases to anything you like — `github-home`, `github-freelance`, etc. Just be consistent throughout.

---

## Step 1 — Generate SSH Keys

Run these commands in your terminal. Each account needs its own key file with a unique name.

```bash
# Personal account — replace with your email
ssh-keygen -t ed25519 -C "personal@email.com" -f ~/.ssh/id_ed25519_personal

# Work account — replace with your work email
ssh-keygen -t ed25519 -C "you@company.com" -f ~/.ssh/id_ed25519_work
```

> Press `Enter` twice to skip the passphrase, or set one for extra security.

Then add both keys to the SSH agent:

```bash
# Start the agent
eval "$(ssh-agent -s)"

# Add both keys
ssh-add ~/.ssh/id_ed25519_personal
ssh-add ~/.ssh/id_ed25519_work

# Confirm both are loaded
ssh-add -l
```

---

## Step 2 — Add Public Keys to GitHub

Copy each public key and paste it into the matching GitHub account. **Log into the correct account before adding each key.**

```bash
# macOS — copies straight to clipboard
pbcopy < ~/.ssh/id_ed25519_personal.pub   # → paste into personal GitHub
pbcopy < ~/.ssh/id_ed25519_work.pub       # → paste into work GitHub

# Linux — print to terminal, then copy manually
cat ~/.ssh/id_ed25519_personal.pub
cat ~/.ssh/id_ed25519_work.pub
```

**Where to paste:** GitHub → **Settings** → **SSH and GPG keys** → **New SSH key**

Use descriptive titles like `MacBook-Personal` and `MacBook-Work` so you can identify them later.

---

## Step 3 — Configure SSH

Create or edit `~/.ssh/config`:

```bash
# Create the file if it doesn't exist
touch ~/.ssh/config

# Set required permissions (SSH will ignore the file otherwise)
chmod 600 ~/.ssh/config

# Open with your preferred editor
nano ~/.ssh/config
# or
code ~/.ssh/config
```

Paste this configuration — replacing the alias names if you chose different ones:

```ssh-config
# ── Personal GitHub ──────────────────────────────────
Host github-personal
  HostName       github.com
  User           git
  IdentityFile   ~/.ssh/id_ed25519_personal
  IdentitiesOnly yes

# ── Work GitHub ──────────────────────────────────────
Host github-work
  HostName       github.com
  User           git
  IdentityFile   ~/.ssh/id_ed25519_work
  IdentitiesOnly yes
```

> **Why `IdentitiesOnly yes`?**
> Without it, SSH may try other keys already loaded in the agent and silently authenticate as the wrong account.

---

## Step 4 — Test Connections

Verify both accounts authenticate correctly **before** doing anything else.

```bash
ssh -T git@github-personal
# ✅ Hi your-personal-username! You've successfully authenticated...

ssh -T git@github-work
# ✅ Hi your-work-username! You've successfully authenticated...
```

If you see `Permission denied (publickey)` — the public key was not added to that GitHub account. Re-check Step 2, then run `ssh -vT git@github-personal` for verbose debug output.

---

## Step 5 — Clone a Repository

**Never use the default `git@github.com:...` URL.** Always replace `github.com` with your alias.

```bash
# Clone a personal repo
git clone git@github-personal:your-personal-username/repo-name.git
cd repo-name
git config user.name  "Your Name"
git config user.email "personal@email.com"

# Clone a work repo
git clone git@github-work:your-work-username/repo-name.git
cd repo-name
git config user.name  "Your Name"
git config user.email "you@company.com"
```

> **Rule of thumb:** Take any GitHub clone URL → replace `github.com` with `github-personal` or `github-work`. That's the only change.

---

## Step 6 — Push a New Local Project

First create an **empty** repository on GitHub.com (no README, no .gitignore), then:

```bash
cd your-project-folder

# 1. Initialise Git
git init

# 2. Set local identity BEFORE the first commit
git config user.name  "Your Name"
git config user.email "personal@email.com"   # or work email

# 3. Stage and commit
git add .
git commit -m "Initial commit"

# 4. Add remote — use the SSH alias, NOT github.com
git remote add origin git@github-personal:your-personal-username/repo-name.git
# Work:
# git remote add origin git@github-work:your-work-username/repo-name.git

# 5. Push
git branch -M main
git push -u origin main
```

---

## Step 7 — Feature Branch Workflow

Never push directly to `main`. One branch per feature or bug fix, then open a pull request.

```bash
# 1. Start from an up-to-date main
git checkout main
git pull origin main

# 2. Create a feature branch
git checkout -b feature/your-feature-name

# 3. Write code, then review changes
git status
git diff

# 4. Stage and commit
git add .
git commit -m "feat: describe what you built"

# 5. Push branch to GitHub
git push -u origin feature/your-feature-name
# GitHub will show a banner → click "Compare & pull request"

# 6. After PR is merged, clean up
git checkout main
git pull origin main
git branch -d feature/your-feature-name
```

### Conventional Commit Prefixes

| Prefix | Use for | Example |
|--------|---------|---------|
| `feat:` | New feature | `feat: add dark mode toggle` |
| `fix:` | Bug fix | `fix: broken login redirect` |
| `chore:` | Config, deps, tooling | `chore: update package.json` |
| `docs:` | Documentation | `docs: update README steps` |
| `refactor:` | Code restructure, no logic change | `refactor: split auth into service` |
| `style:` | CSS / formatting only | `style: fix button padding` |

---

## ✅ Daily Workflow Checklist

Run this block at the start of every session:

```bash
git remote -v              # verify alias — must NOT say github.com
git config user.email      # verify correct identity for this repo
git checkout main
git pull origin main
git checkout -b feature/todays-task
```

---

## ⚠️ Common Mistakes & Fixes

### ❌ Using `github.com` directly

```bash
# Wrong — SSH picks a random key, wrong account gets the push
git clone git@github.com:username/repo.git

# Correct — always use the alias
git clone git@github-personal:username/repo.git
```

### ❌ Forgot to set `user.email`

```bash
# Check recent commit authors
git log --oneline -5

# Fix identity for this repo
git config user.email "correct@email.com"

# Amend the last commit (before pushing)
git commit --amend --reset-author --no-edit
```

### ❌ Wrong remote URL on existing repo

```bash
# Switch to personal
git remote set-url origin git@github-personal:username/repo-name.git

# Switch to work
git remote set-url origin git@github-work:username/repo-name.git
```

### ⚠️ SSH keys disappear after reboot

The SSH agent resets on every restart. Add this to your `~/.zshrc` or `~/.bashrc`:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_personal
ssh-add ~/.ssh/id_ed25519_work
```

---

## 🔧 Troubleshooting

### Debug an SSH connection

```bash
# Verbose output — shows exactly which key is being tried
ssh -vT git@github-personal

# List keys currently loaded in the agent
ssh-add -l

# Re-add keys if the agent is empty
ssh-add ~/.ssh/id_ed25519_personal
ssh-add ~/.ssh/id_ed25519_work
```

### Fix file permissions

SSH silently fails when permissions are too open.

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519_personal
chmod 600 ~/.ssh/id_ed25519_work
chmod 600 ~/.ssh/config

# Verify
ls -la ~/.ssh/
```

### Inspect full repo config

```bash
git config --list --local   # all local settings
git remote -v               # remote URL
git log --oneline -5        # recent commits with author info
```

---

## 🌐 Live Guide

An interactive HTML version of this guide (with a Personal / Work toggle and copy buttons on every command) is included in this repo as `index.html`.

Open it in any browser — it works fully offline.

---

## 🤝 Contributing

Found a mistake or want to add a tip? PRs are welcome.

1. Fork the repo
2. Create a branch: `git checkout -b fix/your-fix`
3. Commit your change: `git commit -m "fix: describe the fix"`
4. Push and open a PR

---

## 📄 License

MIT — free to use, share, and adapt with attribution.

---

<p align="center">
  If this guide saved you time, consider giving it a ⭐
</p>
