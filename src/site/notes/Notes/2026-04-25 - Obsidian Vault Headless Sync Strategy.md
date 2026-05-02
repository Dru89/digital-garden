---
{"dg-publish":true,"permalink":"/notes/2026-04-25-obsidian-vault-headless-sync-strategy/","tags":["obsidian","homelab","sync","infrastructure"],"updated":"2026-05-01T18:40:51.656-07:00"}
---

## Overview

Obsidian vaults are synced continuously to ds9 (home server) using the official `obsidian-headless` CLI, making them available to agents, scripts, and tooling running on the server without requiring the desktop app.

## Setup

- **Tool**: [`obsidian-headless`](https://github.com/obsidianmd/obsidian-headless) (npm global install)
- **Requires**: Node.js 22+
- **Sync mode**: Bidirectional (default)

## Vaults

| Vault | Local path |
|---|---|
| 2026 Personal | `/data/obsidian/2026-personal/` |
| D&D: Witchlight | `/data/obsidian/dnd-witchlight/` |

## Systemd Service

Sync runs as a persistent daemon via a **template unit** — one service file, one instance per vault.

- **Unit file**: `/etc/systemd/system/obsidian-sync@.service`
- **Instance naming**: `obsidian-sync@<vault-dir-name>` (e.g. `obsidian-sync@2026-personal`)
- **User**: `drew`
- **Working directory**: `/data/obsidian/%i` (`%i` = instance name)
- **Command**: `ob sync --continuous`
- **Restart policy**: `on-failure`, 10s delay

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

### Enabled instances

```bash
obsidian-sync@2026-personal.service
obsidian-sync@dnd-witchlight.service
```

### Adding a new vault

A new vault directory must be associated with Obsidian Sync before the service can run headlessly:

```bash
mkdir -p /data/obsidian/<name>
cd /data/obsidian/<name>
ob sync-setup   # interactive: pick the vault from a list
sudo systemctl enable --now obsidian-sync@<name>.service
```

## Rationale

- Vault lives under `/data/` alongside other persistent server data, consistent with Docker appdata layout
- Host path (not Docker volume) keeps vault accessible to any process without indirection
- Bidirectional sync means agents on ds9 can write to the vault and changes propagate to all devices
- Read-only bind mount (`:ro`) recommended when exposing vault to devbox containers or experimental agent code

## Future Work

- Evaluate RAG/vector indexing layer on top of the local vault for agent retrieval
- Integrate local vault path into sift (currently being evaluated in Claude Code)
- Expose vault to devbox containers via read-only bind mount at `/vault`
