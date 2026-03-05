# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

Docker Swarm stack definitions for a home infrastructure. Each subdirectory is an independent stack containing a `docker-compose.yml`. Stacks are automatically deployed via **GitHub Actions** when changes are pushed to `main`.

## Key Infrastructure

- **Reverse proxy:** Traefik v3 (defined in `traefik/`), host-mode ports 80/443 pinned to `percy`
- **Domain pattern:** `<service>.parke.dev` — all services routed through Traefik via deploy labels
- **NAS storage:** Apollo (Unraid) at `10.10.10.215`, NFS root `/mnt/user/swarm-data`
- **Fallback routing:** Unmatched hosts forwarded via TCP passthrough to Dokploy at `10.10.10.54` (configured in `traefik/dynamic/catchall.yaml`)
- **Dashboard:** Homepage service discovers other services via `homepage.*` deploy labels
- **GitOps:** GitHub Actions workflow (`.github/workflows/deploy.yml`) runs on a self-hosted runner inside the swarm

## Swarm Nodes

All three nodes are **managers** (raft quorum tolerates one node failure).

- `percy` — Main manager, Docker 28+, 128GB RAM. Runs: Traefik (host-mode ports 80/443), Home Assistant
- `hercules` — Manager, Docker 28+, 128GB RAM. Available for NFS-based stacks
- `Apollo` — Manager, Unraid NAS, Docker 27.5.1, 64GB RAM. Runs: sonarr, radarr, lidarr, qflood — all with local bind mounts. **Cannot use volume subpath** (requires Docker 28+)
- All nodes connected via 10G internal LAN, 2.5G external WAN
- Stacks using NFS subpath volumes **must** use `node.hostname != Apollo` to avoid Docker 27.5.1

## Storage Patterns

**NFS subpath (Percy/Hercules stacks):** Single `nfs-data` volume at the NFS root + `volume.subpath` per mount. Requires Docker 28+.

```yaml
volumes:
  - type: volume
    source: nfs-data
    target: /app/data
    volume:
      subpath: <stack>/data

volumes:
  nfs-data:
    driver: local
    driver_opts:
      type: nfs
      o: "addr=10.10.10.215,rw,nfsvers=3,soft"
      device: ":/mnt/user/swarm-data"
```

**Local bind mounts (Apollo stacks):** Named volumes with `type: none, o: bind`. Used by qflood, lidarr, radarr, sonarr.

```yaml
volumes:
  app-config:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /mnt/user/swarm-data/<stack>/config
```

## GitOps Flow (GitHub Actions)

A **self-hosted GitHub Actions runner** (`github-runner/`) runs inside the swarm with Docker socket access. It's the bootstrap stack — deployed manually once, then handles all future deployments.

**How it works:**
1. Push a commit to `main`
2. GitHub triggers `.github/workflows/deploy.yml`
3. The self-hosted runner (on percy or hercules) runs `docker stack deploy` for all 13 stacks
4. Docker Swarm handles rolling updates — deploys are idempotent (no-op if nothing changed)
5. Can also be triggered manually via `workflow_dispatch` in the GitHub Actions UI

**VPN credentials** for qflood are stored as GitHub Actions secrets (`VPN_PIA_USER`, `VPN_PIA_PASS`, `VPN_PIA_PREFERRED_REGION`) and exported as env vars during deploy.

**Adding a new stack:**
1. Create `new-stack/docker-compose.yml` in this repo
2. Add the stack name to the `stacks` array in `.github/workflows/deploy.yml`
3. Push to `main` — the workflow deploys it automatically

**The runner does NOT manage itself** — `github-runner/` is the bootstrap stack, deployed manually.

## Stack Conventions (from `.cursor/rules/swarm-stack-conventions.mdc`)

### Traefik Labels (required for HTTP access)

Every service needs: the `traefik_proxy` network, an HTTPS router, an HTTP router with `https-redirect@swarm` middleware, and a loadbalancer port. Router names must be globally unique (use service name as prefix). Do NOT publish ports if Traefik handles ingress.

```yaml
networks:
  traefik_proxy:
    external: true
    name: traefik_traefik_proxy
```

### Deploy Defaults

```yaml
deploy:
  replicas: 1
  restart_policy:
    condition: any
    delay: 5s
    max_attempts: 0
    window: 120s
  update_config:
    parallelism: 1
    delay: 10s
    order: stop-first
```

### Environment Variables

- Secrets use `${VARIABLE}` syntax in compose files, substituted from GitHub Actions secrets during deploy
- Non-sensitive config hardcoded directly in compose files
- `.env` files are gitignored — they exist only for local/manual deploys

### Host-Mode Ports

Services using `mode: host` ports must have a placement constraint pinning to a specific node hostname.
