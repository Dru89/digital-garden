---
{"dg-publish":true,"permalink":"/notes/2026-04-26-obsidian-vault-git-backup/","tags":["obsidian","homelab","backup","git","ds9"],"updated":"2026-05-01T18:38:41.318-07:00"}
---

## Overview

Obsidian vaults on ds9 are backed up hourly via git, with commits pushed to private GitHub repositories. This is independent of Obsidian Sync — it protects against sync-propagated deletions (e.g. the OneDrive incident where mass deletions were synced to the remote before being caught).

Both vaults use **systemd template units** (`obsidian-sync@.service`, `obsidian-git-backup@.service`, `obsidian-git-backup@.timer`) so adding a new vault is just enabling a new instance.

## Vaults

| Vault | Local path | GitHub repo | SSH host alias | Deploy key |
|---|---|---|---|---|
| 2026 Personal | `/data/obsidian/2026-personal/` | `dru89/2026-personal-obsidian-vault-backup` | `github-obsidian-backup` | `~/.ssh/id_ed25519_github_backup` |
| D&D: Witchlight | `/data/obsidian/dnd-witchlight/` | `dru89/dnd-witchlight-obsidian-vault-backup` | `github-dnd-witchlight-backup` | `~/.ssh/id_ed25519_github_dnd_backup` |

## .gitignore

Each vault has its own `.gitignore` at the vault root (not synced via Obsidian Sync — dotfiles are excluded by default):

```
.obsidian/workspace.json
.obsidian/workspace-mobile.json
.obsidian/cache
.trash/
```

Also add `.git` to **Obsidian Sync → Excluded files** to prevent the `.git` folder from syncing across devices.

## Systemd Setup

All units use template syntax — `%i` is the vault name (the directory under `/data/obsidian/`).

**Script**: `/usr/local/bin/obsidian-git-backup.sh`

```bash
#!/bin/bash
VAULT="$1"
cd /data/obsidian/$VAULT
git add -A
if ! git diff --cached --quiet; then
  git commit -m "auto: $(date '+%Y-%m-%d %H:%M')"
  git push origin main
fi
```

**Service**: `/etc/systemd/system/obsidian-sync@.service`

```ini
[Unit]
Description=Obsidian Headless Sync (%i)
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=drew
WorkingDirectory=/data/obsidian/%i
ExecStart=/usr/bin/ob sync --continuous
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

**Service**: `/etc/systemd/system/obsidian-git-backup@.service`

```ini
[Unit]
Description=Obsidian vault git backup (%i)

[Service]
Type=oneshot
User=drew
ExecStart=/usr/local/bin/obsidian-git-backup.sh %i
```

**Timer**: `/etc/systemd/system/obsidian-git-backup@.timer`

```ini
[Unit]
Description=Obsidian vault git backup (%i, hourly)

[Timer]
OnCalendar=hourly
Persistent=true

[Install]
WantedBy=timers.target
```

### Enabled instances

```bash
obsidian-sync@2026-personal.service
obsidian-sync@dnd-witchlight.service
obsidian-git-backup@2026-personal.timer
obsidian-git-backup@dnd-witchlight.timer
```

### Adding a new vault

```bash
# 1. Create directory and set up git
mkdir -p /data/obsidian/<name>
# add .gitignore, git init, git remote add origin <remote>

# 2. Run interactive sync setup (associates directory with Obsidian Sync vault)
cd /data/obsidian/<name>
ob sync-setup

# 3. Enable services
sudo systemctl enable --now obsidian-sync@<name>.service

# 4. Wait for vault to sync, then do initial push
git add -A && git commit -m "initial: vault snapshot $(date '+%Y-%m-%d')"
git push -u origin main

# 5. Enable backup timer
sudo systemctl enable --now obsidian-git-backup@<name>.timer
```

## SSH Deploy Keys

Each vault uses a dedicated passphrase-free SSH key scoped to its backup repo, so systemd can push without an agent.

**SSH config** (`~/.ssh/config`):

```
Host github-obsidian-backup
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_github_backup
  IdentitiesOnly yes

Host github-dnd-witchlight-backup
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_github_dnd_backup
  IdentitiesOnly yes
```

Each key is registered as a deploy key (with write access) on its respective GitHub repo. Git remotes use the SSH host alias so each key stays scoped to one repo.

To generate a new deploy key for a future vault:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_github_<name>_backup -N ""
cat ~/.ssh/id_ed25519_github_<name>_backup.pub
# Add to GitHub repo → Settings → Deploy keys (with write access)
```

## Restoring from Backup

**Find the last known good commit:**

```bash
cd /data/obsidian/<vault>
git log --oneline
git show --stat <commit>   # check what changed
```

**Restore everything to a known good state:**

```bash
git checkout <commit> -- .
git commit -m "restore: roll back to last known good state"
```

**Restore specific files or folders only:**

```bash
git checkout <commit> -- "Sessions/2026-04-26.md"
git checkout <commit> -- "Projects/"
git commit -m "restore: recover deleted notes"
```

Use `git checkout` + a new commit rather than `git reset --hard` to keep history intact and avoid conflicts with Obsidian Sync.
