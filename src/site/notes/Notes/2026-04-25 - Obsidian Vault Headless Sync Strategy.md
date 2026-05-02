---
{"dg-publish":true,"permalink":"/notes/2026-04-25-obsidian-vault-headless-sync-strategy/","tags":["obsidian","homelab","sync","infrastructure"],"updated":"2026-05-01T18:38:50.672-07:00"}
---

## Overview

The Obsidian vault is synced continuously to ds9 (home server) using the official `obsidian-headless` CLI, making it available to agents, scripts, and tooling running on the server without requiring the desktop app.

## Setup

- **Tool**: [`obsidian-headless`](https://github.com/obsidianmd/obsidian-headless) (npm global install)
- **Requires**: Node.js 22+
- **Vault**: "2026 Personal"
- **Local path**: `/data/obsidian/2026-personal/`
- **Sync mode**: Bidirectional (default)

## Systemd Service

The sync runs as a persistent daemon via systemd:

- **Unit file**: `/etc/systemd/system/obsidian-sync.service`
- **User**: `drew`
- **Working directory**: `/data/obsidian/2026-personal/`
- **Command**: `ob sync --continuous`
- **Restart policy**: `on-failure`, 10s delay
- **Enabled**: yes (starts on boot)

```ini
[Unit]
Description=Obsidian Headless Sync
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=drew
WorkingDirectory=/data/obsidian/2026-personal
ExecStart=/usr/bin/ob sync --continuous
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
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

