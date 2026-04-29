# Speckle-DEV Setup Notes
Date created: 2026-04-27 | Last updated: 2026-04-29 (v2.0.0 — no-op Camera converter, version strings, full docs)

> Mirror of this file lives on the dev VM at `/home/rebus/speckle-server-dev/rebus/` (git-tracked).

---

## Overview

| Property | Value |
|---|---|
| **URL** | https://speckle-dev.rebus.industries |
| **Purpose** | Pre-release / dev testing. Tracks `:latest` Speckle images (alpha/pre-release channel). |
| **VM** | RB-DA2-VM-SpeckleDev |
| **Proxmox node** | RB-DA2-SRV03, VMID 301 |
| **IP** | 10.0.200.112 (VLAN 200) |
| **OS** | Ubuntu 24.04.4 |
| **Resources** | 8 vCPU / 8 GB RAM / 128 GB disk |
| **Source** | Full clone of VM 201 (Speckle-Server), created 2026-04-27 |

---

## VM Setup History

- **2026-04-27:** Cloned from production VM 201 (Speckle-Server) via Proxmox.
  - Clone was already in progress when session started; disk was mounted with `qemu-nbd` pre-boot to patch the netplan IP directly on disk (avoiding an IP conflict with production).
  - Hostname changed from `RB-DA2-VM-SpeckleServer` → `RB-DA2-VM-SpeckleDev`.
  - IP changed from `10.0.200.11` → `10.0.200.112` (patched on disk via qemu-nbd before first boot, then applied via reboot).
  - VM resources reduced to 8 vCPU / 8 GB RAM (from 16/16) to conserve SRV03 capacity.
  - Data directories wiped on first boot (fresh Postgres, MinIO, Redis — not cloned from production).

---

## SSH Access

```
ssh -i ~/.ssh/id_ed25519_rebus rebus@10.0.200.112
```

Or via `~/.ssh/config` alias:
```
ssh speckle-dev
```

The `id_ed25519_rebus` key (stored in `D:\Documents\Claude\REBUS System\`) is authorised on `~rebus/.ssh/authorized_keys`.

---

## Docker Stack

### Location
All compose files live at `/opt/speckle/` on the VM.

### Files
| File | Purpose |
|---|---|
| `docker-compose.yml` | Main stack. `:latest` Speckle images, all URLs → `speckle-dev.rebus.industries`. Reference copy: `docker-compose.yml` in this folder. |
| `docker-compose.override.yml` | Host port bindings + custom frontend image. Reference copy: `docker-compose.override.yml` in this folder. |
| `docker-compose.yml.prod-orig` | Backup of the production compose from before the dev conversion. |

### Services & ports

| Service | Image | Host port |
|---|---|---|
| speckle-frontend-2 | `speckle-frontend-2-rebus:v2.0.0` ⚠️ custom build | 80 |
| speckle-server | `speckle/speckle-server:latest` | 3000 |
| minio | `minio/minio` | 9000 (S3), 9001 (console) |
| postgres | `postgres:16.9-alpine` | — (internal) |
| redis (valkey) | `valkey/valkey:8-alpine` | — (internal) |
| preview-service | `speckle/speckle-preview-service:latest` | — (internal) |
| webhook-service | `speckle/speckle-webhook-service:latest` | — (internal) |
| fileimport-service | `speckle/speckle-fileimport-service:latest` | — (internal) |

### Key environment differences from production

| Variable | Production | Dev |
|---|---|---|
| `CANONICAL_URL` | `https://speckle.rebus.industries` | `https://speckle-dev.rebus.industries` |
| Image tags | `2.31.2` (pinned) | `latest` (pre-release) |
| `SESSION_SECRET` | (prod secret) | Separate dev secret |
| Data | Live production data | Fresh / empty |

### Useful commands

```bash
# Start / stop / restart
cd /opt/speckle && docker compose up -d
cd /opt/speckle && docker compose down
cd /opt/speckle && docker compose restart

# Check status
docker compose -p speckle-server ps --format "table {{.Name}}\t{{.Status}}"

# Pull latest images and restart (update)
speckle-update

# Logs
cd /opt/speckle && docker compose logs -f speckle-server
cd /opt/speckle && docker compose logs -f speckle-frontend-2

# Port check
ss -ltn | awk 'NR==1 || /:80 |:3000 |:9000 |:9001 /'

# Smoke test
curl -s http://localhost:80 -o /dev/null -w "Frontend: %{http_code}\n"
curl -s http://localhost:3000/readiness -o /dev/null -w "Server: %{http_code}\n"
```

### Updating to latest images

The `speckle-update` script on this VM has been **replaced** with a simple pull-and-restart (no version pinning):

```bash
speckle-update
# equivalent to:
docker compose --project-directory /opt/speckle pull
docker compose --project-directory /opt/speckle up -d
```

---

## Reverse Proxy (Caddy)

Traffic flows: Internet → public IP → VRRP VIP `10.0.200.250` → Proxy1 (or Proxy2 on failover) → `10.0.200.112`.

### Caddyfile block (identical on Proxy1 and Proxy2)

```
speckle-dev.rebus.industries {
    @backend {
        path /graphql /api/* /auth/* /objects/* /preview/* /static/*
    }

    reverse_proxy @backend 10.0.200.112:3000
    reverse_proxy 10.0.200.112:80
}
```

### TLS certificate

Let's Encrypt via HTTP-01 challenge. Certificate issued 2026-04-27 by Proxy1 and synced to Proxy2 via the existing 6-hour rsync timer.

### DNS

`speckle-dev.rebus.industries` A record → same public IP as `speckle.rebus.industries`. Added 2026-04-27.

---

## GitHub — Fork & Branch

### Repository
`CheekiSkrub/speckle-server-dev`
https://github.com/CheekiSkrub/speckle-server-dev

Forked from `specklesystems/speckle-server` (all branches, 2026-04-27).

### Branch structure

| Branch | Purpose |
|---|---|
| `main` | Mirror of upstream `specklesystems/speckle-server:main`. Keep in sync, don't commit REBUS changes here. |
| `rebus-dev` | **Active working branch.** REBUS-specific config and patches live here. |

### REBUS config in the repo

`rebus/` directory on the `rebus-dev` branch:
- `docker-compose.yml` — the live dev compose config
- `docker-compose.override.yml` — port bindings
- `README.md` — deployment notes

### Local clone on the dev VM

```
/home/rebus/speckle-server-dev/   ← git clone of the fork, tracking rebus-dev
```

### SSH key for GitHub

Generated at `~rebus/.ssh/id_ed25519_github` on the dev VM.
- Comment: `speckle-dev@rebus.industries`
- Public key: `ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICgJuZGtNB68uMDsA48GOER8uXq8QgERSacjo+g+98Gc speckle-dev@rebus.industries`
- Added to GitHub account: `CheekiSkrub` → Settings → SSH keys → "Speckle-DEV VM (RB-DA2-VM-SpeckleDev, 10.0.200.112)"

### Syncing with upstream Speckle

```bash
cd ~/speckle-server-dev
git remote add upstream git@github.com:specklesystems/speckle-server.git  # first time only
git fetch upstream
git checkout main && git merge upstream/main && git push origin main
git checkout rebus-dev && git merge main
git push origin rebus-dev
```

### Committing config changes back to the repo

When you update `/opt/speckle/docker-compose.yml` (e.g. after a new Speckle release changes the compose format):

```bash
cd ~/speckle-server-dev
cp /opt/speckle/docker-compose.yml rebus/docker-compose.yml
git add rebus/
git commit -m "chore(rebus): update docker-compose to reflect latest changes"
git push origin rebus-dev
```

---

## Auto-start on Boot

Docker CE (`docker.service`) is enabled as a systemd service. All containers have `restart: always` and start automatically on reboot.

---

## Known Differences from Production VM

| Item | Production (VM 201) | Dev (VM 301) |
|---|---|---|
| IP | 10.0.200.11 | 10.0.200.112 |
| Hostname | RB-DA2-VM-SpeckleServer | RB-DA2-VM-SpeckleDev |
| URL | speckle.rebus.industries | speckle-dev.rebus.industries |
| Speckle version | 2.31.2 (pinned) | `:latest` (pre-release) |
| Data | Live production | Fresh/empty |
| RAM | 16 GB | 8 GB |
| `speckle-update` | Interactive, checks Docker Hub for stable/pre-release | Simple pull-and-restart |
| Backups | Scheduled (pending cron setup) | None configured |

---

## Backups

```bash
speckle-backup          # run a backup now
speckle-backup list     # list existing backups
speckle-backup status   # show l