# Zero-Config VPS Docker Deploy

Zero-boilerplate automated continuous deployment to your VPS using Docker Compose, with built-in Doppler integration, container health verification, and image rebuild / cache-invalidation controls.

---

## ⚡ Quick Start: Deploy Any Repo in 8 Lines

In any of your project repositories, simply create `.github/workflows/deploy.yml`:

```yaml
name: Deploy
on:
  push:
    branches: [main, master]
  workflow_dispatch:

jobs:
  deploy:
    uses: isavage/deploy/.github/workflows/deploy.yml@v2
    secrets: inherit
```

That's it! When you push code or click **Run workflow**, your project automatically syncs to your VPS at `/docker/<repo-name>` and restarts your containers.

---

## 🔐 Required GitHub Secrets

Set these secrets in your repository (**Settings > Secrets and variables > Actions**) or at your **GitHub Organization / Personal Account** level so they are shared across all repositories:

| Secret | Scoped Level | Description | Required |
|---|---|---|---|
| `VPS_HOST` | **Org or Repo** | Hostname or IP address of your VPS | **Yes** |
| `VPS_USER` | **Org or Repo** | SSH username on your VPS (e.g. `root` or a sudo-enabled user) | **Yes** |
| `VPS_SSH_KEY` | **Org or Repo** | Private SSH key (matching `~/.ssh/authorized_keys` on VPS) | **Yes** |
| `DOPPLER_TOKEN` | **Repo Only** | Project-specific Service Token from Doppler (`dp.st...`) | **Optional** |
| `ENV_FILE` | **Repo Only** | Complete `.env` content from GitHub secrets | **Optional** |

> [!TIP]
> - **Shared VPS Secrets (`VPS_HOST`, `VPS_USER`, `VPS_SSH_KEY`)**: Add these once at your **GitHub Organization / Account** level so all repositories inherit them automatically.
> - **`DOPPLER_TOKEN` / `ENV_FILE`**: Add at each **individual Repository level** (or Environment level) so secrets remain isolated per project.

---

## 🌟 Environment Variables & Secrets Handling

The action gives you 3 convenient ways to manage environment variables:

### Mode 1: Doppler (Recommended if using Doppler)
If `DOPPLER_TOKEN` is passed:
- Doppler CLI is installed on the VPS (if missing).
- Secrets are injected in memory directly to Docker Compose:
  ```bash
  doppler run -- docker compose up -d
  ```
- No `.env` file needs to be written to disk.

### Mode 2: Secrets from GitHub (`ENV_FILE` / `env_content`)
If not using Doppler, you can store your `.env` contents in GitHub Secrets as `ENV_FILE`:
- With `secrets: inherit`, any secret named `ENV_FILE` in your repo is automatically passed.
- The action safely writes it to `.env` on your VPS and sets file permissions to `600` before starting containers.
- You can also specify a custom filename via `with: env_file_name: ".env.production"`.

### Mode 3: Persistent VPS `.env` File (Auto-Protected)
If you prefer creating and managing your `.env` directly on your VPS:
- **`rsync --delete` will NEVER delete your VPS `.env` files**: The deploy action automatically excludes `.env` and `.env.*` from deletion.
- Docker Compose will read your existing server `.env` on startup.

---

## 🔄 Rebuilding Images & Cache Invalidation

When you make changes to a Dockerfile or source code that needs a clean build without cache, you can trigger a deploy with the `no_cache` option enabled:

### Option A: Manual Trigger via Workflow Dispatch
Add inputs to your caller workflow:

```yaml
name: Deploy
on:
  push:
    branches: [main, master]
  workflow_dispatch:
    inputs:
      no_cache:
        description: 'Rebuild Docker images without cache (--no-cache)'
        required: false
        type: boolean
        default: false
      cleanup:
        description: 'Prune unused docker images'
        required: false
        type: boolean
        default: false

jobs:
  deploy:
    uses: isavage/deploy/.github/workflows/deploy.yml@v2
    secrets: inherit
    with:
      no_cache: ${{ inputs.no_cache || false }}
      cleanup: ${{ inputs.cleanup || false }}
```

### Option B: Force Rebuild Always in Workflow
```yaml
jobs:
  deploy:
    uses: isavage/deploy/.github/workflows/deploy.yml@v2
    secrets: inherit
    with:
      no_cache: true
```

When `no_cache: true` is set, the action executes:
```bash
docker compose build --no-cache
docker compose up -d --force-recreate --remove-orphans
```

---

## ⚙️ Full Configuration Reference

All inputs are optional and have sensible defaults matching standard VPS setups:

| Input | Type | Default | Description |
|---|---|---|---|
| `target_dir` | string | `/docker/<repo-name>` | Target directory on VPS |
| `files` | string | `*` | Files or glob pattern to transfer to VPS |
| `exclude` | string | `""` | Rsync exclude patterns (nothing excluded by default) |
| `compose_file` | string | `docker-compose.yml` | Compose file path (auto-detects `compose.yml` if missing) |
| `port` | string | `22` | SSH port |
| `down` | boolean | `true` | Runs `docker compose down` before recreation |
| `pull` | boolean | `true` | Runs `docker compose pull` before starting |
| `build` | boolean | `false` | Runs `docker compose build` before starting |
| `no_cache` | boolean | `false` | Builds images with `--no-cache` |
| `force_recreate`| boolean | `true` | Runs `docker compose up -d --force-recreate` |
| `cleanup` | boolean | `false` | Runs `docker image prune -f` |
| `cleanup_image` | string | `""` | Specific image to remove (e.g. `user/app:latest`) |
| `service_name` | string | `""` | Specific service name to monitor health for |
| `pre_deploy_command` | string | `""` | Shell command to execute before `compose up` |
| `post_deploy_command`| string | `""` | Shell command to execute after `compose up` |
| `environment` | string | `""` | GitHub Environment to bind for secret protection |
| `env_content` | string | `""` | Optional .env contents passed from secrets |
| `env_file_name` | string | `.env` | File name for written environment variables |

---

## 🧩 Advanced Examples

### 1. Custom Bootstrap Script & Excludes
```yaml
jobs:
  deploy:
    uses: isavage/deploy/.github/workflows/deploy.yml@v2
    secrets: inherit
    with:
      target_dir: "/docker/hermes"
      exclude: ".git, .github, docs"
      service_name: "hermes"
      pre_deploy_command: "bash scripts/bootstrap-hermes-config.sh hermes-data"
```

### 2. Composite Action inside Custom Multi-Step Workflow
If your repository already runs tests or builds assets on GitHub Actions before deploying:

```yaml
name: Test & Deploy
on:
  push:
    branches: [main]

jobs:
  ci-cd:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Tests
        run: npm test

      - name: Deploy to VPS
        uses: isavage/deploy@v2
        with:
          host: ${{ secrets.VPS_HOST }}
          user: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          doppler_token: ${{ secrets.DOPPLER_TOKEN }}
          no_cache: false
```

---

## 🩺 Built-In Health Checks & Failure Prevention
- **Retry Polling**: `compose up -d` can return while a container is still initializing. The action polls container runtime state via `docker inspect` for up to 30 intervals (with sleep) so deployments only report success when the container is truly running and healthy.
- **Instant Error Logs**: If a container fails or crashes on startup, the action automatically streams `docker compose logs --tail=100` into the GitHub Actions run output and halts the pipeline with an error code.
- **Sudo & Permission Resilience**: Automatically supports both `root` users and non-root users with `sudo` privileges without permission errors during directory creation or file syncing.

