# REBUS Deployment Config

This directory contains the Docker Compose configuration for the REBUS Industries
Speckle deployment, maintained on the `rebus-dev` branch.

## Files

| File | Purpose |
|------|---------|
| `docker-compose.yml` | Main stack config. Speckle images pinned to `:latest` (pre-release). All URLs point to `speckle-dev.rebus.industries`. |
| `docker-compose.override.yml` | Host port bindings (80→frontend, 3000→server, 9000-9001→minio). Applied automatically by `docker compose` when both files are present. |

## Deployed at

- **URL:** https://speckle-dev.rebus.industries
- **VM:** RB-DA2-VM-SpeckleDev (Proxmox VMID 301, IP 10.0.200.112)
- **Stack location on VM:** `/opt/speckle/`
- **Image channel:** `:latest` (Speckle pre-release/dev builds)

## Updating

SSH to the dev VM and run:

```bash
speckle-update
```

This pulls the latest images and restarts the stack.

## Upstream

This branch (`rebus-dev`) is forked from `specklesystems/speckle-server` main.
To sync with upstream:

```bash
git fetch upstream
git merge upstream/main
git push origin rebus-dev
```
