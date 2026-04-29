# Speckle-DEV Setup Notes
Date created: 2026-04-27 | Last updated: 2026-04-29 (v2.1.0 — unified version across all three custom images)

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
| speckle-frontend-2 | `speckle-frontend-2-rebus:v2.1.0` ⚠️ custom build | 80 |
| speckle-server | `speckle-server-rebus:v2.1.0` ⚠️ custom build | 3000 |
| preview-service | `speckle-preview-service-rebus:v2.1.0` ⚠️ custom build | — (internal) |
| minio | `minio/minio` | 9000 (S3), 9001 (console) |
| postgres | `postgres:16.9-alpine` | — (internal) |
| redis (valkey) | `valkey/valkey:8-alpine` | — (internal) |
| webhook-service | `speckle/speckle-webhook-service:latest` | — (internal) |
| fileimport-service | `speckle/speckle-fileimport-service:latest` | — (internal) |

### Key environment differences from production

| Variable | Production | Dev |
|---|---|---|
| `CANONICAL_URL` | `https://speckle.rebus.industries` | `https://speckle-dev.rebus.industries` |
| Image tags | `v2.1.0` all three custom images | `v2.1.0` all three custom images + `:latest` upstream |
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
| Speckle version | v2.1.0 custom images | v2.1.0 custom images + :latest upstream |
| Data | Live production | Fresh/empty |
| RAM | 16 GB | 8 GB |
| `speckle-update` | Interactive menu — lists local `speckle-frontend-2-rebus:v*` images, no Docker Hub | Simple pull-and-restart |
| Backups | TUI available (option 5 = schedule); schedule not yet configured | TUI available (option 5 = schedule); schedule not yet configured |

---

## Backups

```bash
speckle-backup          # run a backup now
speckle-backup list     # list existing backups
speckle-backup status   # show log of last backup
```

Backups stored in `/home/rebus/backups/` as timestamped directories. Retention: 14 (set in `/opt/speckle/backup.conf`).

Desktop shortcut: **Speckle DEV Backup** — opens the interactive TUI.

### Implementation note — sudo not available

The `speckle-backup` and `speckle-backup-tui` scripts at `/usr/local/bin/` were copied from the production VM and still contain `sudo bash` calls. The `rebus` user on this VM does not have passwordless sudo. These are shadowed by corrected versions at `/home/rebus/bin/` which call `bash` directly (the `rebus` user is in the `docker` group so Docker exec works without sudo).

Additionally, `backup.sh` was originally hardcoded to `/home/ubuntu/backups` (from the original EC2/Ubuntu build). This was corrected on 2026-04-29 to `/home/rebus/backups`.

**When `/usr/local/bin/speckle-backup*` get updated**, re-apply the same fix:
```bash
# Re-shadow with corrected versions if the system scripts are ever replaced:
# ~/bin/speckle-backup  — uses bash instead of sudo bash, BACKUP_DIR=/home/rebus/backups
# ~/bin/speckle-backup-tui — same, gnome-terminal call uses /home/rebus/bin/speckle-backup
```

---

## Custom Frontend Build (`speckle-frontend-2-rebus`)

### Background

The upstream Speckle frontend (`speckle/speckle-frontend-2:latest`) has limitations for REBUS use:

1. **iFrame embedding blocked** — `X-Frame-Options` prevents embedding in `app.rebus.industries`.
2. **V3 camera views not working** — `Objects.Other.Camera` objects from the Rhino V3 connector are not natively supported in the viewer version shipping with `:latest`. Named views (EXTERIOR 1, SeatingTop, etc.) simply don't appear in the panel.
3. **Camera geometry pollutes the scene** — `Objects.Other.Camera` sub-objects (Point, Vector) are rendered as visible 3D geometry and inflate the scene bounding box, breaking canonical views and FIT/zoom.

Rather than waiting for upstream fixes, we build a patched version of the frontend locally on the dev VM and run it instead of the upstream image.

The patched image is named **`speckle-frontend-2-rebus`** and the current version is **`v2.1.0`** (git tag `REBUS-v2.1.0`).

### Files changed

| File | Change | Version |
|---|---|---|
| `packages/frontend-2/lib/viewer/composables/setup.ts` | V3 named views, camera geometry fix | v1.6.0+ |
| `packages/frontend-2/components/settings/server/general/Version.vue` | Hardcoded version, removed update check | v1.10.0+ |
| `packages/frontend-2/components/header/nav/UserMenu.vue` | Hardcoded version in user dropdown | v2.0.0 |
| `packages/viewer/src/modules/loaders/Speckle/SpeckleConverter.ts` | No-op Camera converter (root fix) | v1.17.0 |

All changes on branch `rebus-dev`.

---

### Patch 1 — iFrame embedding (v1.5.0, commit `650257fbb55c`)

Strips `X-Frame-Options` from responses and sets a permissive CSP `frame-ancestors` to allow embedding in `app.rebus.industries`. See commit history for details.

---

### Patch 2 — V3 named camera views (v1.6.0–v1.8.0)

**Problem**: The Rhino V3 connector sends named views as `Objects.Other.Camera` nodes rather than the older `Objects.View3D` format. The `:latest` viewer's `getViews()` only understands `View3D`, so no named views appear in the panel.

**Fix — pre-fetch exact camera positions** (added v1.6.0, commit `850cf8d41ade`)

The objectloader2 strips coordinate values from inline `Objects.Geometry.Point` sub-objects when loading. To recover the exact position, the code fetches the root commit object via `/objects/{streamId}/{objectId}/single` (returns full inline JSON), extracts the `position` from each camera in the `views` array, and builds a `Map<cameraId, {x,y,z}>`.

**Fix — named view target field** (fixed v1.8.0, commit `c1e3f7d38419`)

**Critical gotcha**: In `Objects.Other.Camera`, the `forward` field is the **absolute look-at target world position**, NOT a direction vector. Despite the name, use it directly as `target`:

```typescript
// WRONG — treats forward as direction:
tx = px + fwd.x;  ty = py + fwd.y;  tz = pz + fwd.z

// CORRECT — forward IS the absolute target position:
tx = fwd.x;  ty = fwd.y;  tz = fwd.z
```

`SpeckleView` objects are pushed to `views.value` in `{origin, target}` format (Rhino Z-up world space). The viewer's `CameraController.setViewSpeckle` → `SmoothOrbitControls.fromPositionAndTarget` handles the Y-up conversion internally.

---

### Patch 3 — Camera geometry and bounding box fix

#### Symptoms

When loading a model from the Rhino V3 connector on the `:latest` build, you may observe:

- Small **point markers** visible in the 3D scene near camera positions (often at architectural scale heights like z=300+)
- **Vector arrows** visible in the scene (look-at and up directions)
- The **FIT / zoom-extents button** zooms out to an enormous bounding box that includes the camera positions, not just the building geometry
- **Canonical views** (Front, Top, Left, Right, Section box) are broken — they frame an oversized world that includes the camera positions
- On the **cloud** viewer (`app.speckle.systems`) the same model looks fine — no stray points, correct bounding box

#### Why it happens — data flow

The Rhino V3 connector sends each named view as an `Objects.Other.Camera` node. That node has three sub-object fields, each with an explicit `speckle_type`:

| Field | `speckle_type` | What it is |
|---|---|---|
| `position` | `Objects.Geometry.Point` | Camera eye position in world space |
| `forward` | `Objects.Geometry.Vector` | Absolute look-at target (NOT a direction vector — see Patch 2) |
| `up` | `Objects.Geometry.Vector` | Camera up direction |

On the cloud build these sub-objects are plain untyped JSON objects, so the loader ignores them. On the `:latest` self-hosted build they carry `speckle_type`, making them indistinguishable from real renderable geometry.

The viewer processes incoming objects through `SpeckleConverter.ts`, which maintains a `delegates` map:

```
speckle_type string → converter function
```

The `traverse()` method works like this:

```
for each node in the tree:
    if node.type is in delegates:
        call delegates[node.type](node)   ← returns early, children NOT traversed
    else:
        traverse all children recursively ← children ARE converted to geometry
```

If `Objects.Other.Camera` is **absent** from the delegates map, the Camera node falls through to default traversal. Its children (`Objects.Geometry.Point`, `Objects.Geometry.Vector`) each get picked up by their own delegates, converted to render batches, and included in `worldBox`.

**Why hiding doesn't fix the bounding box**: `viewer.World.worldBox` is assembled during load by calling `World.expandWorld(batch.bounds)` for each new batch. By the time `ViewerEvent.LoadComplete` fires and any post-load hiding runs, the camera geometry is already inside `worldBox`. `FilteringExtension.hideObjects()` only affects `renderer.visibleSceneBox` (computed on-the-fly from visible batches), not `worldBox` (which is never recomputed after load). So the scene looks clean but canonical views and the Section box still use the inflated bounds.

#### The fix — no-op Camera converter

**File**: `packages/viewer/src/modules/loaders/Speckle/SpeckleConverter.ts`
**Commit**: `c90320c44`

Add a no-op entry to the `delegates` map so `traverse()` finds Camera and returns immediately, never descending into children:

```typescript
View3D: this.View3DToNode.bind(this),
// REBUS: no-op Camera converter — keeps Camera node in world tree (needed for named
// view walk in setup.ts) but returns before traversing children, preventing
// Objects.Geometry.Point and Objects.Geometry.Vector sub-objects from being
// converted to renderable geometry and inflating worldBox.
Camera: async (_obj: SpeckleObject, _node: TreeNode) => { return },
BlockInstance: this.BlockInstanceToNode.bind(this),
```

This is inserted between `View3D` and `BlockInstance` in the delegates map. The exact position doesn't matter functionally, but it keeps Camera next to View3D (both are view-related types).

**Why the Camera node itself is kept**: The named view walk in `setup.ts` calls `viewer.getWorldTree().walk()` looking for nodes where `raw.speckle_type === 'Objects.Other.Camera'`. The no-op converter still adds the Camera node to the world tree — it just prevents child traversal. Remove the Camera entry entirely and named views stop working.

**Why the sub-objects are never created**: `traverse()` calls the no-op and returns. The Point and Vector children are never visited, never converted, never batched. They simply don't exist as far as the renderer is concerned. `World.expandWorld()` is never called with camera bounds, so `worldBox` reflects only real model geometry.

#### Safety nets in setup.ts

Even with the SpeckleConverter fix, `setup.ts` retains two functions as defensive fallbacks in case the upstream build changes behaviour:

```typescript
// Hides any Camera nodes and their children that somehow made it through
const hideCameraGeometry = () => {
  const nodeIds: string[] = []
  viewer.getWorldTree().walk((node: TreeNode) => {
    const raw = node.model?.raw
    if (!node.model?.id || !raw) return true
    const isCameraNode = raw.speckle_type === 'Objects.Other.Camera'
    const isCameraChild = node.parent?.model?.raw?.speckle_type === 'Objects.Other.Camera'
    if (isCameraNode || isCameraChild) nodeIds.push(node.model.id)
    return true
  })
  if (nodeIds.length === 0) return
  viewer.getExtension(FilteringExtension).hideObjects(nodeIds, 'rebus-camera-geometry', false, false)
}

// Overwrites worldBox with visibleSceneBox (which respects hidden objects)
// Must run AFTER hideCameraGeometry so visibleSceneBox is already clean
const fixWorldBox = () => {
  requestAnimationFrame(() => {
    const visibleBox = viewer.getRenderer().visibleSceneBox
    if (!visibleBox.isEmpty()) {
      viewer.World.worldBox.copy(visibleBox)
    }
  })
}
```

**Important**: `fixWorldBox()` uses `requestAnimationFrame` to ensure it runs after the renderer has processed the hide state from `hideCameraGeometry()`. The call order must always be `hideCameraGeometry()` → `fixWorldBox()`.

**Why `.copy()` on worldBox persists**: `worldBox` is only rebuilt when `World.expandWorld()` or `World.reduceWorld()` are called — both happen only during model load/unload. Overwriting it post-load with `.copy()` is safe and persists for the entire viewing session.

#### How to verify the fix is working

If you suspect the fix has regressed (e.g. after an upstream merge), check for these signs:

1. **In the browser console** after loading a model: `viewer.World.worldBox` should have bounds consistent with your building, not enormous z-values. Run in console: `window.__viewer?.World.worldBox` (if viewer is exposed) or add a temporary `console.log(viewer.World.worldBox)` after `fixWorldBox()` fires.
2. **Visually**: Load a model with named camera views. The FIT button should zoom to the building, not a huge empty space. No point markers or vector arrows should be visible.
3. **In the world tree**: Open browser devtools → confirm no `Objects.Geometry.Point` or `Objects.Geometry.Vector` nodes are in the tree as children of any `Objects.Other.Camera` node.

#### If the problem returns

The most likely cause is an upstream merge into `rebus-dev` that overwrites `SpeckleConverter.ts`. Check that the no-op Camera entry is still present in the `delegates` map:

```bash
cd /home/rebus/speckle-server-dev
grep -n 'Camera' packages/viewer/src/modules/loaders/Speckle/SpeckleConverter.ts
```

Expected output should include the no-op line. If it's missing, re-apply the patch from commit `c90320c44`.

---

### Patch 4 — Version strings and update check removal

**v1.10.0 (commit `34bded1ec`)**: `packages/frontend-2/components/settings/server/general/Version.vue` — replaced entire component with a static hardcoded version display, removing the GitHub releases API poll and "Update to x.x.x" button.

**v2.0.0 (commit `f3a08981b`)**: `packages/frontend-2/components/header/nav/UserMenu.vue` — the user dropdown showed "Version custom" (from `serverInfo.value?.version` which returns "custom" for `:latest` builds). Patched to return a hardcoded string:

```typescript
// Before:
const version = computed(() => serverInfo.value?.version)

// After (REBUS):
const version = computed(() => 'REBUS v2.0.0') // REBUS: hardcoded, replaces serverInfo.value?.version which shows 'custom'
```

Both files need their version string updated when building a new release.

---

### Version history

| Version | Commit | What changed |
|---------|--------|--------------|
| v1.1.0–v1.4.0 | `10c4700981b6` | Early iFrame/feature-flag iterations |
| v1.5.0 | `650257fbb55c` | iFrame fix (`X-Frame-Options` stripped, CSP `frame-ancestors`) |
| v1.6.0 | `850cf8d41ade` | V3 camera views: root object pre-fetch for exact positions |
| v1.7.0 | `706cd38a736b` | worldBox fix (early attempt, later superseded) |
| v1.8.0 | `c1e3f7d38419` | Named view target fix (`target = fwd` not `pos + fwd`) |
| v1.9.0 | `87dbaff28` | Hide camera Position Points from scene; version string baked in |
| v1.10.0 | `34bded1ec` | Hardcoded version "REBUS v1.10.0", removed update check (Version.vue) |
| v1.11.0 | `2ed5185fc` | Extended hide to Point+Vector; added fixWorldBox |
| v1.12.0 | `c3f5227b9` | Attempt: batch-level worldBox fix via renderData.id matching |
| v1.13.0 | — | Fix: renderData.id is full URL, not raw.id hash |
| v1.14.0 | `98b536f45` | Attempt: in-place batch bounds recomputation (didn't clean up) |
| v1.15.0 | `046d09a9f` | Hide Camera+all children by type; copy visibleSceneBox → worldBox |
| v1.16.0 | — | requestAnimationFrame timing fix for fixWorldBox |
| v1.17.0 | `c90320c44` | **Definitive fix**: no-op Camera converter in SpeckleConverter.ts |
| v2.0.0 | `f3a08981b` | Version strings → "REBUS v2.0.0" in UserMenu.vue + Version.vue; tagged `REBUS-v2.0.0` |
| v2.0.1 | (no source change) | Custom `speckle-preview-service-rebus:v2.0.1` built from patched monorepo — applies no-op Camera converter fix to thumbnail generation. Tagged `REBUS-v2.0.1`. |
| v2.1.0 | `ed54ee58a` | **Unified build version.** All three custom images share one version number: `speckle-frontend-2-rebus:v2.1.0` (rebuilt), `speckle-server-rebus:v2.1.0` (retagged from v1.0.0), `speckle-preview-service-rebus:v2.1.0` (retagged from v2.0.1). Going forward all images are bumped together. Tagged `REBUS-v2.1.0`. |

The **original upstream base** was `speckle/speckle-frontend-2:latest` as of 2026-04-27. The REBUS image uses the same Dockerfile with patches applied and `SPECKLE_SERVER_VERSION='REBUS v{n}'` baked in at build time.

---

### How to rebuild the frontend

When you modify source files, rebuild like this on the dev VM:

```bash
# SSH in
ssh -i ~/.ssh/id_ed25519_rebus rebus@10.0.200.112

# 1. Update version strings in source before building
#    - packages/frontend-2/lib/viewer/composables/setup.ts (if changed)
#    - packages/frontend-2/components/settings/server/general/Version.vue
#      → change 'REBUS v2.1.0' to new version
#    - packages/frontend-2/components/header/nav/UserMenu.vue
#      → change 'REBUS v2.1.0' to new version

# 2. Build — always bump the version tag and pass SPECKLE_SERVER_VERSION
cd /home/rebus/speckle-server-dev
NEW_VER=v2.2.0
nohup docker build \
  --build-arg SPECKLE_SERVER_VERSION="REBUS ${NEW_VER}" \
  -f packages/frontend-2/Dockerfile \
  -t speckle-frontend-2-rebus:${NEW_VER} . \
  > /tmp/docker-build-${NEW_VER}.log 2>&1 &

# 3. Watch progress (~3–4 min: client ~50s, server ~30s, nitro ~30s)
tail -f /tmp/docker-build-${NEW_VER}.log

# 4. Update the override and restart (use python3 — sed -i fails in /opt/speckle/)
python3 -c "
import re
with open('/opt/speckle/docker-compose.override.yml','r') as f: c=f.read()
c = re.sub(r'speckle-frontend-2-rebus:v[\d.]+', 'speckle-frontend-2-rebus:${NEW_VER}', c)
with open('/opt/speckle/docker-compose.override.yml','w') as f: f.write(c)
print('Updated')
"
cd /opt/speckle && docker compose up -d speckle-frontend-2

# 5. Commit source changes
cd /home/rebus/speckle-server-dev
git add packages/frontend-2/lib/viewer/composables/setup.ts \
        packages/frontend-2/components/settings/server/general/Version.vue \
        packages/frontend-2/components/header/nav/UserMenu.vue \
        packages/viewer/src/modules/loaders/Speckle/SpeckleConverter.ts  # if changed
git commit -m "fix/feat: describe change (${NEW_VER})"
git push origin rebus-dev

# 6. Tag the release
git tag -a REBUS-${NEW_VER} -m "REBUS ${NEW_VER}: describe what changed"
git push origin REBUS-${NEW_VER}
```

---

### How to rebuild the preview-service

The preview thumbnail renderer (`speckle-preview-service`) is also a custom build. It uses `packages/preview-frontend` — a small Vite app that imports `@speckle/viewer` directly, including `SpeckleConverter.ts`. Any changes to the viewer package (e.g. the no-op Camera converter) are therefore included automatically when you build from the `rebus-dev` branch.

The preview-service image is large (~1.4 GB, includes Chromium) and takes ~8–10 minutes to build.

```bash
# Build from monorepo root
cd /home/rebus/speckle-server-dev
NEW_VER=v2.2.0
nohup docker build \
  -f packages/preview-service/Dockerfile \
  -t speckle-preview-service-rebus:${NEW_VER} . \
  > /tmp/build-preview-${NEW_VER}.log 2>&1 &

# Watch progress
tail -f /tmp/build-preview-${NEW_VER}.log

# Update dev override and restart
python3 -c "
import re
with open('/opt/speckle/docker-compose.override.yml','r') as f: c=f.read()
c = re.sub(r'speckle-preview-service-rebus:v[\d.]+', 'speckle-preview-service-rebus:${NEW_VER}', c)
with open('/opt/speckle/docker-compose.override.yml','w') as f: f.write(c)
print('Updated')
"
cd /opt/speckle && docker compose up -d preview-service

# Transfer to production (360 MB compressed)
docker save speckle-preview-service-rebus:${NEW_VER} | gzip > /tmp/preview-${NEW_VER}.tar.gz
scp -i /tmp/id_ed25519_rebus -o StrictHostKeyChecking=no \
  /tmp/preview-${NEW_VER}.tar.gz rebus@10.0.200.11:/tmp/
ssh -i /tmp/id_ed25519_rebus -o StrictHostKeyChecking=no rebus@10.0.200.11 \
  "gunzip -c /tmp/preview-${NEW_VER}.tar.gz | docker load"
# Then patch /opt/speckle/docker-compose.override.yml on prod and restart preview-service

# Tag in git
git tag -a REBUS-${NEW_VER} -m "REBUS ${NEW_VER}: describe what changed"
git push origin REBUS-${NEW_VER}
```

**Note**: the preview-service Dockerfile has no `SPECKLE_SERVER_VERSION` build-arg — the version is only tracked via the Docker image tag and git tag, not embedded in the binary.

---

### How named views work — full technical summary

**On load**, `ViewerEvent.LoadComplete` fires → `refreshWorldTreeAndFilters()` runs:

1. **Pre-fetch root object** — calls `/objects/{streamId}/{objectId}/single` (the root commit object URL from `viewer.getWorldTree().root.children[0].model.id`). This returns the full inline JSON including camera `position` coordinates, which objectloader2 otherwise strips. Builds `Map<cameraId, {x,y,z}>`.

2. **Walk world tree** — finds nodes where `raw.speckle_type === 'Objects.Other.Camera'`. For each, constructs a `SpeckleView`:
   ```typescript
   { id, name, speckle_type, origin: position, target: forward }
   ```
   where `position` comes from the pre-fetch map and `forward` is used directly as the absolute look-at target (NOT added to position).

3. **Merge views** — appends V3 camera views to V2 `View3D` views from `viewer.getViews()`, stores in `views.value` → appears in Saved Views panel.

4. **Hide camera geometry** — `hideCameraGeometry()` hides all `Objects.Other.Camera` nodes and their children using `FilteringExtension`. Since v1.17.0 this is a safety net only — the no-op converter in SpeckleConverter.ts means there is no geometry to hide.

5. **Fix worldBox** — `fixWorldBox()` copies `renderer.visibleSceneBox` (which now excludes hidden camera geometry) into `viewer.World.worldBox`. Since v1.17.0 also a safety net — worldBox is never inflated by camera geometry in the first place.

**Camera field semantics** (`Objects.Other.Camera` from Rhino V3 connector):
- `position` — camera eye world position (Rhino Z-up coords). objectloader2 strips the x/y/z values; use pre-fetch to recover them.
- `forward` — **absolute look-at target world position** (Rhino Z-up). The name is misleading — it is NOT a direction vector.
- `up` — camera up direction unit vector.

**Viewer internals relevant to this patch**:
- `viewer.World.worldBox` — Box3 used by canonical views (Front/Top/etc.) and Section box. Built from `batch.bounds` during load via `World.expandWorld()`. NOT affected by FilteringExtension visibility.
- `renderer.visibleSceneBox` — Box3 computed on-the-fly from visible render view ranges. IS affected by FilteringExtension hide state.
- `renderer.clippingVolume` — used by `zoomExtents` (FIT button). Falls back to `visibleSceneBox` when local clipping is off (default).
- `FilteringExtension.hideObjects(ids, stateKey, includeDescendants, ghost)` — takes `node.model.id` (full URL, **not** `raw.id` hash). Updates visibility state and affects `visibleSceneBox`.
- `renderData.id` — equals `node.model.id` (full URL), set in `RenderTree.buildRenderNode`. **NOT** the same as `raw.id` (short hash). This distinction caused v1.12.0–v1.13.0 to fail.
- `SpeckleConverter.delegates` map — routes speckle_type → converter function. Types absent from this map fall through to default child traversal, which converts all sub-objects to renderable geometry. The no-op Camera entry prevents this.

---

## TODO

- [ ] Configure backup schedule (`speckle-backup setup`) if persistent dev data is needed.
- [ ] Set up upstream remote in the git clone (`git remote add upstream git@github.com:specklesystems/speckle-server.git`).
- [ ] Rotate MinIO default credentials (`minioadmin` / `minioadmin`).
