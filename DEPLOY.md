# Deployment Runbook

Steps to deploy this swarm infrastructure with GitHub Actions GitOps.

## Swarm Topology

| Node | Role | Docker | RAM | Services |
|------|------|--------|-----|----------|
| **percy** | Manager (leader) | 28+ | 128GB | Traefik (host-mode 80/443), Home Assistant, GitHub Actions runner |
| **hercules** | Manager | 28+ | 128GB | NFS-based stacks (prowlarr, seerr, termix, homepage) |
| **Apollo** | Manager (Unraid NAS) | 27.5.1 | 64GB | qflood, lidarr, radarr, sonarr (all local bind mounts) |

All nodes are managers (3-node raft quorum — survives one node failure). All on 10G internal LAN.

`*.parke.dev` DNS must point to **percy's** IP (Traefik ingress node).

---

## Prerequisites

### 1. Verify Docker versions

```bash
# On percy (must be 28+)
docker version --format '{{.Server.Version}}'

# On hercules (must be 28+)
ssh hercules 'docker version --format "{{.Server.Version}}"'

# On Apollo (expected: 27.5.x — cannot upgrade)
ssh apollo 'docker version --format "{{.Server.Version}}"'
```

### 2. Ensure overlay network exists

Run on **percy**:

```bash
docker network create --driver overlay --attachable traefik_traefik_proxy 2>/dev/null \
  || echo "Network already exists"
```

### 3. Ensure Docker secrets exist

### 4. Verify directories on Apollo

```bash
# QFlood
ssh apollo 'ls -d /mnt/user/swarm-data/qflood/config /mnt/user/swarm-data/qflood/data'

# *arr apps
ssh apollo 'ls -d /mnt/user/appdata/{lidarr,radarr,sonarr}/{config,data}'

# If any are missing:
# ssh apollo 'mkdir -p /mnt/user/swarm-data/qflood/{config,data}'
# ssh apollo 'mkdir -p /mnt/user/appdata/{lidarr,radarr,sonarr}/{config,data}'
```

### 5. Verify NFS directories for subpath stacks

```bash
ssh apollo 'ls -d /mnt/user/swarm-data/{traefik,homepage,prowlarr,seerr,termix}'
```

### 6. Ensure DNS is correct

`*.parke.dev` must resolve to **percy's** IP address.

---

## Set Up GitHub Actions Secrets

In your GitHub repo (`https://github.com/LukasParke/swarm`), go to **Settings > Secrets and variables > Actions** and add:

| Secret | Value |
|--------|-------|
| `VPN_PIA_USER` | Your PIA VPN username |
| `VPN_PIA_PASS` | Your PIA VPN password |
| `VPN_PIA_PREFERRED_REGION` | Your preferred PIA region |

These are injected into the deploy workflow and substituted into qflood's compose file.

---

## Deploy the GitHub Actions Runner (Bootstrap)

The runner is the **only stack deployed manually**. Everything else is deployed by the workflow it runs.

### 1. Generate a GitHub Personal Access Token

Go to `https://github.com/settings/tokens` and create a **classic** token with `repo` scope. Copy it.

### 2. Deploy the runner on percy

```bash
ssh percy

# Clone the repo
git clone https://github.com/LukasParke/swarm.git ~/swarm
# OR: cd ~/swarm && git pull

# Deploy with your GitHub PAT
export GITHUB_RUNNER_TOKEN="ghp_your_token_here"
docker stack deploy -c ~/swarm/github-runner/docker-compose.yml github-runner
```

### 3. Verify the runner registered

```bash
# Check the service is running
docker service ls | grep github-runner

# Check logs for successful registration
docker service logs github-runner_runner
```

Also verify in GitHub: go to **repo > Settings > Actions > Runners** — you should see `swarm-deployer` listed as idle.

---

## Trigger the First Deploy

### Option A: Push a commit

Any push to `main` triggers the workflow. Push the repo changes:

```bash
git add -A
git commit -m "Switch to GitHub Actions GitOps with self-hosted runner"
git push
```

### Option B: Manual trigger

Go to **repo > Actions > Deploy Stacks > Run workflow** and click "Run workflow".

### Verify

Watch the workflow run in the GitHub Actions tab. Then on percy:

```bash
# All stacks should appear
docker stack ls

# Verify qflood is on Apollo
docker service ps qflood_qflood

# Verify Traefik is on percy
docker service ps traefik_traefik

# Test an endpoint
curl -sk https://whoami.parke.dev
```

---

## Day-to-Day Operations

### Updating a stack

1. Edit the compose file
2. `git commit && git push`
3. GitHub Actions deploys it within seconds

### Adding a new stack

1. Create `new-stack/docker-compose.yml`
2. Edit `.github/workflows/deploy.yml` — add `new-stack` to the `stacks` array
3. If the stack uses NFS subpath, ensure directories exist:
   ```bash
   ssh apollo 'mkdir -p /mnt/user/swarm-data/new-stack/data'
   ```
4. `git commit && git push`

### Removing a stack

1. Remove the stack name from `.github/workflows/deploy.yml`
2. Push the change
3. Manually remove the stack from the swarm:
   ```bash
   docker stack rm old-stack
   ```
4. Clean up orphaned volumes:
   ```bash
   docker volume ls | grep old-stack
   docker volume rm old-stack_volume-name
   ```

### Re-running a failed deploy

Go to **repo > Actions**, find the failed run, and click "Re-run all jobs".

---

## Cleaning Up Old Volumes

After migrating to the new volume setup, old per-subdirectory NFS volumes will be orphaned:

```bash
# List all volumes
docker volume ls

# Remove orphans (examples)
docker volume rm qflood_nfs-data
docker volume rm traefik_nfs-root traefik_traefik-certs traefik_traefik-dynamic
# Repeat for each stack's old volumes
```

---

## Updating the Runner

If the runner image or config needs updating:

```bash
ssh percy
cd ~/swarm && git pull
export GITHUB_RUNNER_TOKEN="ghp_your_token_here"
docker stack deploy -c ~/swarm/github-runner/docker-compose.yml github-runner
```

---

## Troubleshooting

### Workflow not triggering

- Verify the runner is online: **repo > Settings > Actions > Runners**
- Check runner logs: `docker service logs github-runner_runner`

### Stack failing to deploy

```bash
docker stack ps <stack-name> --no-trunc
docker service logs <stack-name>_<service-name>
```

### Volume mount failures on Apollo

Directories must exist before deploying:

```bash
ssh apollo 'mkdir -p /mnt/user/swarm-data/<stack>/<subdir>'
docker stack rm <stack>
sleep 10
docker volume rm <stack>_<volume-name>
# Push a commit or manually re-trigger the workflow
```

### Node failure

With 3 managers, the swarm tolerates one node going down:
- **Apollo down:** qflood + *arr apps go offline (pinned to Apollo). Everything else continues.
- **Hercules down:** NFS stacks reschedule to percy (if not pinned). Runner may move.
- **Percy down:** Traefik goes offline (pinned to percy). Raft quorum still holds with 2/3 managers. Runner reschedules to hercules and continues deploying.
