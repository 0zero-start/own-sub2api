# Docker Compose Project Rename Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Rename the local Docker Compose project from `deploy` to `own-sub2api` without losing runtime data.

**Architecture:** Set `COMPOSE_PROJECT_NAME` only in the ignored local `.env`, remove the old project's containers and network without volumes, then recreate the same services against the unchanged bind-mounted directories.

**Tech Stack:** Docker Desktop, Docker Compose, PowerShell

---

### Task 1: Capture Pre-Rename State

**Files:**
- Inspect: `deploy/.env`
- Inspect: `deploy/data/`
- Inspect: `deploy/postgres_data/`
- Inspect: `deploy/redis_data/`

- [ ] **Step 1: Verify old services are healthy and runtime paths exist**

Run from `deploy/`:

```powershell
docker compose -p deploy --env-file .env -f docker-compose.local.yml ps
foreach ($dir in 'data','postgres_data','redis_data') {
  if (-not (Test-Path $dir)) { throw "Missing runtime directory: $dir" }
}
```

Expected: three healthy services and all runtime directories present.

### Task 2: Set the Local Project Name

**Files:**
- Modify locally: `deploy/.env`

- [ ] **Step 1: Verify `.env` is ignored**

Run:

```powershell
git check-ignore -v deploy/.env
```

Expected: `deploy/.gitignore` excludes `.env`.

- [ ] **Step 2: Add or replace the Compose project name without printing secrets**

Run from the repository root:

```powershell
$path = Join-Path $PWD 'deploy/.env'
$content = [IO.File]::ReadAllText($path)
if ($content -match '(?m)^COMPOSE_PROJECT_NAME=') {
  $content = [Text.RegularExpressions.Regex]::Replace(
    $content,
    '(?m)^COMPOSE_PROJECT_NAME=.*$',
    'COMPOSE_PROJECT_NAME=own-sub2api'
  )
} else {
  $content = "COMPOSE_PROJECT_NAME=own-sub2api`r`n$content"
}
[IO.File]::WriteAllText($path, $content, [Text.UTF8Encoding]::new($false))
```

Expected: exactly one `COMPOSE_PROJECT_NAME=own-sub2api` line exists; no secret is printed.

### Task 3: Recreate Under the New Name

**Files:**
- Read: `deploy/.env`
- Read: `deploy/docker-compose.local.yml`

- [ ] **Step 1: Stop the old project without deleting volumes or bind directories**

Run from `deploy/`:

```powershell
docker compose -p deploy --env-file .env -f docker-compose.local.yml down
```

Expected: old containers and `deploy_sub2api-network` are removed; no `--volumes` option is used.

- [ ] **Step 2: Validate and start the new project**

Run from `deploy/`:

```powershell
docker compose --env-file .env -f docker-compose.local.yml config --quiet
docker compose --env-file .env -f docker-compose.local.yml up -d
```

Expected: the same three containers start under project `own-sub2api`.

### Task 4: Verify Rename, Health, and Git Isolation

**Files:**
- Inspect: `deploy/.env`
- Inspect: `deploy/data/`
- Inspect: `deploy/postgres_data/`
- Inspect: `deploy/redis_data/`

- [ ] **Step 1: Wait for all three health checks**

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
  throw 'Renamed Compose services did not become healthy'
}
```

Expected: three running, healthy services before the deadline.

- [ ] **Step 2: Verify the Compose project name and HTTP responses**

Run:

```powershell
docker compose ls
Invoke-WebRequest -UseBasicParsing http://127.0.0.1:8080/health
Invoke-WebRequest -UseBasicParsing http://127.0.0.1:8080/
```

Expected: `own-sub2api` is running, `deploy` is absent, and both requests return HTTP 200.

- [ ] **Step 3: Verify runtime data remains local**

Run from the repository root:

```powershell
git check-ignore -v deploy/.env deploy/data deploy/postgres_data deploy/redis_data
git ls-files deploy/.env deploy/data deploy/postgres_data deploy/redis_data
git status --short --branch
```

Expected: all runtime paths are ignored, none is tracked, and only documentation commits differ from `origin/main`.
