# Sub2API Local Docker Compose Deployment Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Run the official Sub2API stack locally with durable, Git-ignored data and push the complete source history to `0zero-start/own-sub2api`.

**Architecture:** Use the official local-directory Compose file to run the published Sub2API image with PostgreSQL and Redis on an isolated Docker network. Bind only the application to `127.0.0.1:8080`; keep generated secrets and all runtime state under ignored paths in `deploy/`.

**Tech Stack:** Git, GitHub CLI, Docker Desktop, Docker Compose v5, PowerShell, Sub2API, PostgreSQL 18, Redis 8

---

## File Map

- Create locally: `deploy/.env` — host-only configuration and generated secrets; ignored by Git.
- Create locally: `deploy/data/` — Sub2API runtime data; ignored by Git.
- Create locally: `deploy/postgres_data/` — PostgreSQL database; ignored by Git.
- Create locally: `deploy/redis_data/` — Redis persistence; ignored by Git.
- Use unchanged: `deploy/docker-compose.local.yml` — official three-service Compose definition.
- Use unchanged: `deploy/.env.example` — official environment template.

### Task 1: Preflight and Secret Isolation

**Files:**
- Inspect: `deploy/docker-compose.local.yml`
- Inspect: `deploy/.env.example`
- Inspect: `deploy/.gitignore`

- [ ] **Step 1: Verify Docker Desktop, Compose, port availability, and repository remotes**

Run:

```powershell
docker version
docker compose version
Get-NetTCPConnection -LocalPort 8080 -State Listen -ErrorAction SilentlyContinue
git remote -v
```

Expected: Docker client and server respond, Compose reports a version, port 8080 has no listener, `origin` points to `0zero-start/own-sub2api`, and `upstream` points to `Wei-Shaw/sub2api`.

- [ ] **Step 2: Verify every runtime path is ignored before creating secrets**

Run:

```powershell
git check-ignore -v deploy/.env deploy/data/example deploy/postgres_data/example deploy/redis_data/example
```

Expected: all four paths match `deploy/.gitignore`.

### Task 2: Generate Local Configuration

**Files:**
- Create: `deploy/.env`
- Create: `deploy/data/`
- Create: `deploy/postgres_data/`
- Create: `deploy/redis_data/`

- [ ] **Step 1: Generate the environment from the official template without printing secrets**

Run from the repository root:

```powershell
$path = Join-Path $PWD 'deploy/.env'
$content = [IO.File]::ReadAllText((Join-Path $PWD 'deploy/.env.example'))
function Set-EnvValue([string]$Text, [string]$Name, [string]$Value) {
  [Text.RegularExpressions.Regex]::Replace($Text, "(?m)^$([Text.RegularExpressions.Regex]::Escape($Name))=.*$", "$Name=$Value")
}
function New-HexSecret {
  $rng = [Security.Cryptography.RandomNumberGenerator]::Create()
  try {
    $bytes = New-Object byte[] 32
    $rng.GetBytes($bytes)
    [BitConverter]::ToString($bytes).Replace('-','').ToLowerInvariant()
  } finally {
    $rng.Dispose()
  }
}
$content = Set-EnvValue $content 'BIND_HOST' '127.0.0.1'
$content = Set-EnvValue $content 'SERVER_PORT' '8080'
$content = Set-EnvValue $content 'TZ' 'Asia/Shanghai'
$content = Set-EnvValue $content 'POSTGRES_PASSWORD' (New-HexSecret)
$content = Set-EnvValue $content 'JWT_SECRET' (New-HexSecret)
$content = Set-EnvValue $content 'TOTP_ENCRYPTION_KEY' (New-HexSecret)
$content = Set-EnvValue $content 'ADMIN_PASSWORD' ''
[IO.File]::WriteAllText($path, $content, [Text.UTF8Encoding]::new($false))
```

Expected: `deploy/.env` exists and no secret value is emitted to the terminal.

- [ ] **Step 2: Create persistent host directories**

Run:

```powershell
New-Item -ItemType Directory -Force deploy/data, deploy/postgres_data, deploy/redis_data | Out-Null
```

Expected: all three directories exist.

- [ ] **Step 3: Validate required values without revealing them**

Run:

```powershell
$required = 'POSTGRES_PASSWORD','JWT_SECRET','TOTP_ENCRYPTION_KEY'
$lines = Get-Content deploy/.env
foreach ($name in $required) {
  $line = $lines | Where-Object { $_ -like "$name=*" } | Select-Object -First 1
  if (-not $line -or ($line -split '=',2)[1].Length -ne 64) { throw "$name is missing or invalid" }
}
if (-not ($lines -contains 'BIND_HOST=127.0.0.1')) { throw 'BIND_HOST is not loopback-only' }
```

Expected: exit code 0 with no secret values printed.

### Task 3: Validate and Start the Stack

**Files:**
- Read: `deploy/.env`
- Read: `deploy/docker-compose.local.yml`

- [ ] **Step 1: Render and validate the Compose configuration**

Run:

```powershell
docker compose --env-file .env -f docker-compose.local.yml config --quiet
```

Working directory: `deploy/`

Expected: exit code 0 and no validation error.

- [ ] **Step 2: Pull the official images**

Run:

```powershell
docker compose --env-file .env -f docker-compose.local.yml pull
```

Working directory: `deploy/`

Expected: Sub2API, PostgreSQL, and Redis images are present locally.

- [ ] **Step 3: Start all services in the background**

Run:

```powershell
docker compose --env-file .env -f docker-compose.local.yml up -d
```

Working directory: `deploy/`

Expected: `sub2api`, `sub2api-postgres`, and `sub2api-redis` are created or started.

### Task 4: Health and Persistence Verification

**Files:**
- Inspect: `deploy/data/`
- Inspect: `deploy/postgres_data/`
- Inspect: `deploy/redis_data/`

- [ ] **Step 1: Wait for all health checks and fail on timeout**

Run from `deploy/`:

```powershell
$deadline = (Get-Date).AddMinutes(5)
do {
  $rows = docker compose --env-file .env -f docker-compose.local.yml ps --format json | ConvertFrom-Json
  $bad = @($rows | Where-Object { $_.State -ne 'running' -or $_.Health -ne 'healthy' })
  if ($bad.Count -eq 0 -and @($rows).Count -eq 3) { break }
  Start-Sleep -Seconds 5
} while ((Get-Date) -lt $deadline)
if ($bad.Count -ne 0 -or @($rows).Count -ne 3) {
  docker compose --env-file .env -f docker-compose.local.yml ps
  docker compose --env-file .env -f docker-compose.local.yml logs --tail 100
  throw 'Compose services did not become healthy'
}
```

Expected: three running, healthy services before the five-minute deadline.

- [ ] **Step 2: Verify HTTP health and Web UI**

Run:

```powershell
$health = Invoke-WebRequest -UseBasicParsing http://127.0.0.1:8080/health
$web = Invoke-WebRequest -UseBasicParsing http://127.0.0.1:8080/
if ($health.StatusCode -ne 200 -or $web.StatusCode -ne 200) { throw 'HTTP verification failed' }
```

Expected: both requests return HTTP 200.

- [ ] **Step 3: Retrieve the generated administrator credential from first-run logs**

Run:

```powershell
docker compose --env-file .env -f docker-compose.local.yml logs sub2api | Select-String -Pattern 'admin password' -CaseSensitive:$false
```

Expected: first-run log contains the generated administrator password. Provide it to the user only in the local conversation; do not write it to a tracked file.

- [ ] **Step 4: Restart and re-verify persistence**

Run:

```powershell
$before = (Get-FileHash .env -Algorithm SHA256).Hash
docker compose --env-file .env -f docker-compose.local.yml restart
$after = (Get-FileHash .env -Algorithm SHA256).Hash
if ($before -ne $after) { throw '.env changed during restart' }
if (-not (Test-Path postgres_data) -or -not (Test-Path redis_data) -or -not (Test-Path data)) { throw 'Persistent directory missing' }
```

Then repeat the health and HTTP checks from Steps 1 and 2.

Expected: services return to healthy state, HTTP checks pass, `.env` is unchanged, and all persistence directories remain.

### Task 5: Git Safety Check and GitHub Sync

**Files:**
- Commit: `docs/superpowers/plans/2026-07-06-local-docker-compose-deployment.md`
- Exclude: `deploy/.env`
- Exclude: `deploy/data/`
- Exclude: `deploy/postgres_data/`
- Exclude: `deploy/redis_data/`

- [ ] **Step 1: Prove secrets and runtime data are ignored and untracked**

Run:

```powershell
git check-ignore -v deploy/.env deploy/data deploy/postgres_data deploy/redis_data
git status --short --ignored
git ls-files deploy/.env deploy/data deploy/postgres_data deploy/redis_data
```

Expected: ignore rules are shown, runtime paths appear only as ignored, and `git ls-files` returns no runtime file.

- [ ] **Step 2: Inspect the exact commits that will be pushed**

Run:

```powershell
git status --short --branch
git log --oneline upstream/main..HEAD
git diff --stat upstream/main..HEAD
git diff --check upstream/main..HEAD
```

Expected: only the reviewed deployment design and implementation plan commits are ahead of official `upstream/main`; no `.env` or runtime data appears.

- [ ] **Step 3: Authenticate GitHub CLI if needed**

Run:

```powershell
gh auth status
gh auth login --web --git-protocol https
```

Expected: browser authentication completes for the GitHub account that can write `0zero-start/own-sub2api`.

- [ ] **Step 4: Push complete history and set branch tracking**

Run:

```powershell
git push -u origin main
```

Expected: push succeeds without force and local `main` tracks `origin/main`.

- [ ] **Step 5: Verify local and remote branch identity**

Run:

```powershell
git fetch origin main
$local = git rev-parse main
$remote = git rev-parse origin/main
if ($local -ne $remote) { throw 'Local and GitHub main branches differ' }
git status --short --branch
```

Expected: local and remote commit IDs match and the working tree has no tracked changes.
