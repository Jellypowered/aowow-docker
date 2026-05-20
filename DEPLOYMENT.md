# AoWoW Deployment Guide

Containerized AoWoW (WotLK 3.3.5a) connected to an external AzerothCore server.

## Architecture Overview

```
                    ┌─────────────┐
                    │   Caddy     │  (optional, port 443/80)
                    │  Reverse    │
                    │   Proxy     │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  aowow_web  │  PHP-FPM + Apache (port 80)
                    │             │  Reads: world, auth, characters (SELECT)
                    │             │  Owns: aowow (full access)
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │    db       │  MariaDB 10.11
                    │  (internal) │  AoWoW DB schema lives here
                    └─────────────┘
                           │
              ┌────────────▼────────────┐
              │  External AzerothCore   │
              │  world / auth / chars   │
              │  (read-only access)     │
              └─────────────────────────┘
```

## Database Ownership

| Database | Owned By | Managed By | Access |
|----------|----------|------------|--------|
| `aowow` | AoWoW service | `aowow-docker` entrypoint creates DB, imports `setup/sql/*.sql` schema | Full (CREATE/ALTER/INSERT/UPDATE/DELETE) |
| `world` | AzerothCore | External AC deployment — **never modified by AoWoW** | SELECT only |
| `auth` | AzerothCore | External AC deployment — **never modified by AoWoW** | SELECT only |
| `characters` | AzerothCore | External AC deployment — **never modified by AoWoW** | SELECT only |

**Critical:** AoWoW must never create, alter, or write to AzerothCore databases. The entrypoint only creates the `aowow` database and grants full privileges to the AoWoW user on that database. World/auth/characters credentials are used for read-only queries only.

## Required Database Credentials

All credentials are configured via `.env` (copy from `.env.example`).

### AoWoW Database (created by aowow-docker)

| Variable | Default | Description |
|----------|---------|-------------|
| `AOWOW_DB_HOST` | `db` | MariaDB container hostname (internal) |
| `AOWOW_DB_DATABASE` | `aowow` | Database name (created by entrypoint) |
| `AOWOW_DB_USER` | `aowow_user` | AoWoW application user |
| `AOWOW_DB_PASSWORD` | `aowow_password` | AoWoW application password |

### AzerothCore Databases (externally managed)

| Variable | Default | Description |
|----------|---------|-------------|
| `WORLD_DB_HOST` | `host.docker.internal` | External AC world DB host |
| `WORLD_DB_DATABASE` | `world` | AC world database name |
| `WORLD_DB_USER` | `world_user` | Read-only AC world user |
| `WORLD_DB_PASSWORD` | `world_password` | AC world password |
| `AUTH_DB_HOST` | `host.docker.internal` | External AC auth DB host |
| `AUTH_DB_DATABASE` | `auth` | AC auth database name |
| `AUTH_DB_USER` | `auth_user` | Read-only AC auth user |
| `AUTH_DB_PASSWORD` | `auth_password` | AC auth password |
| `CHARACTERS_DB_HOST` | `host.docker.internal` | External AC characters DB host |
| `CHARACTERS_DB_DATABASE` | `characters` | AC characters database name |
| `CHARACTERS_DB_USER` | `characters_user` | Read-only AC characters user |
| `CHARACTERS_DB_PASSWORD` | `characters_password` | AC characters password |

**Note:** The defaults use `host.docker.internal` because world/auth/characters databases are external to the compose stack. If your AzerothCore runs on a different host or network, update these values accordingly.

### Root Credentials (for AoWoW DB creation only)

| Variable | Default | Description |
|----------|---------|-------------|
| `MYSQL_ROOT_PASSWORD` | `rootpassword` | MariaDB root password — used only by the entrypoint to create the AoWoW database and user |

## Public URL / Host Configuration

AoWoW must know its public-facing URL to generate correct links in pages, emails, and API responses.

| Variable | Default | Description |
|----------|---------|-------------|
| `SITE_HOST` | `localhost:8080` | Public hostname:port for the main site |
| `STATIC_HOST` | `localhost:8080/static` | Public hostname:port for static assets |

### Without Reverse Proxy

If accessing AoWoW directly (no Caddy):
```
SITE_HOST=localhost:8080
STATIC_HOST=localhost:8080/static
```

### With Caddy Reverse Proxy

If using the included Caddy proxy (recommended for production):
```
SITE_HOST=your-domain.com
STATIC_HOST=your-domain.com/static
```

Update the `Caddyfile` with your domain and SSL configuration. The Caddy container proxies to `aowow_web:80` on the private network.

**Important:** `SITE_HOST` and `STATIC_HOST` must match what users actually type in their browsers. If you use HTTPS behind a reverse proxy, set `AOWOW_FORCE_SSL=1`.

## Reverse Proxy Expectations

### Caddy (included)

The `docker-compose.caddy.yml` file adds a Caddy container that:
- Listens on port 443 (HTTPS) and/or 80 (HTTP)
- Proxies all traffic to `aowow_web:80` on the `aowow_private_net` network
- Handles SSL termination (configure certificates in `Caddyfile`)

To start with Caddy:
```bash
docker compose -f docker-compose.yml -f docker-compose.caddy.yml up -d
```

### External Reverse Proxy

AoWoW is a standard PHP web application behind Apache. Any reverse proxy that forwards `Host` headers correctly will work. Key requirements:

1. **Forward the `Host` header** — AoWoW uses `SITE_HOST` for link generation; if the proxy strips or rewrites the host, links will be incorrect.
2. **Pass `/static/` paths** — Static assets are served directly by Apache inside the web container. The proxy should forward `/static/*` requests to the web container.

## AzCoreWeb Integration (Optional)

If you want to serve AoWoW alongside AzCoreWeb on the same domain, you can proxy AoWoW under a subpath. This is one option — AoWoW can also run on its own domain.

### Example: Subpath Proxy

```
https://your-domain.com/aowow/          → AoWoW frontend
https://your-domain.com/                → AzCoreWeb
```

### Nginx Proxy Example

```nginx
location /aowow/ {
    proxy_pass http://aowow_web:80/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

### AoWoW Configuration for Subpath

When served under a subpath, set `SITE_HOST` and `STATIC_HOST` to include the prefix:
```
SITE_HOST=your-domain.com/aowow
STATIC_HOST=your-domain.com/aowow/static
```

### Standalone Deployment

If AoWoW runs on its own domain, no subpath configuration is needed:
```
SITE_HOST=aowow.your-domain.com
STATIC_HOST=aowow.your-domain.com/static
```

## Network Configuration

The compose stack creates a named bridge network `aowow_private_net` that connects the MariaDB and AoWoW web containers. This network is not isolated — the `aowow_web` container can reach external hosts (including external AzerothCore databases) via `host.docker.internal` (Docker Desktop) or the host's DNS (Linux).

If your AzerothCore databases run on a separate Docker network, add that network to the compose configuration so the web container can resolve those hosts.

## Quick Start Checklist

- [ ] AzerothCore server is running with world, auth, and characters databases populated
- [ ] Read-only database users exist for world, auth, and characters
- [ ] `cp .env.example .env` and edit credentials
- [ ] `WOW_CLIENT_PATH` points to a WoW 3.3.5a installation
- [ ] `SITE_HOST` / `STATIC_HOST` match your public URL
- [ ] `docker compose up -d --build` completes without errors
- [ ] `docker compose logs -f web` shows `Deployment setup complete!`
- [ ] `docker compose exec web php aowow --setup` completes successfully
- [ ] Web interface loads at `http://SITE_HOST` (or configured URL)
- [ ] Realms appear in the profiler (auth DB connection verified)

## Live Runtime Validation

Use this procedure to verify a new deployment against a live AzerothCore server.

### Preconditions

- AzerothCore is running with `world`, `auth`, and `characters` databases populated
- A WoW 3.3.5a client exists at a known path (e.g., `/Users/you/WoW`)
- Read-only DB users exist for world/auth/characters on the AC database server
- Docker and Docker Compose are available
- You have a terminal in the `aowow-docker` directory

### Step 1 — Configure `.env`

```bash
cp .env.example .env
```

Edit `.env` and set:

```env
WOW_CLIENT_PATH=/path/to/your/wow-3.3.5a
WOW_LOCALE=enUS
WORLD_DB_HOST=<your-ac-world-db-host>
WORLD_DB_DATABASE=world
WORLD_DB_USER=<readonly-world-user>
WORLD_DB_PASSWORD=<world-password>
AUTH_DB_HOST=<your-ac-auth-db-host>
AUTH_DB_DATABASE=auth
AUTH_DB_USER=<readonly-auth-user>
AUTH_DB_PASSWORD=<auth-password>
CHARACTERS_DB_HOST=<your-ac-characters-db-host>
CHARACTERS_DB_DATABASE=characters
CHARACTERS_DB_USER=<readonly-characters-user>
CHARACTERS_DB_PASSWORD=<characters-password>
```

Leave `AOWOW_DB_HOST=db` (uses the local MariaDB container).

### Step 2 — Build and Start

```bash
docker compose up -d --build
```

### Step 3 — Wait for Extraction

```bash
docker compose logs -f web
```

Wait until you see:

```
[Success] Deployment setup complete!
```

Then press `Ctrl+C`.

### Step 4 — Run AoWoW Setup

```bash
docker compose exec web php aowow --setup
```

Follow the interactive prompts. When you reach the database test screen:
- AoWoW DB should show `OK`
- World DB should show `OK` or `WARN` (not `ERR`)
- Auth DB should show `OK`

**Note:** The ACDB vendor-prefix rejection has been removed. A `WARN` with version info is acceptable. `ERR` means credentials or host are unreachable, or the world DB lacks a `version` table — investigate the error message.

### Step 5 — Disable Maintenance Mode

```bash
docker compose exec web php aowow config maintenance 0
```

### Step 6 — Verify Web Interface

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080
```

### Step 7 — Verify Realm Listing

```bash
curl -s http://localhost:8080/?realm=list | head -50
```

### Step 8 — Verify Static Assets

First discover an available asset path:

```bash
docker compose exec web ls /var/www/html/static/images/wow/Interface/Buttons/ 2>/dev/null | head -5
```

Then test the asset:

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/static/images/wow/Interface/Buttons/<discovered-file>
```

### Step 9 — Verify Reverse Proxy (Optional)

```bash
docker compose -f docker-compose.yml -f docker-compose.caddy.yml up -d
curl -s -o /dev/null -w "%{http_code}" https://localhost
```

### What to Verify After Each Step

| Step | Expected Outcome | Failure Sign |
|------|-----------------|--------------|
| 2 | Build completes, containers start, no `ERROR` in compose output | `ERROR` in compose output, container exits immediately |
| 3 | Logs show `Deployment setup complete!` | Logs show `Cannot connect to ... — check credentials and network` |
| 4 | All three DBs show `OK` or `WARN` (not `ERR`) | Any DB shows `ERR` — credentials or host unreachable |
| 5 | No error output from the command | Error about missing config key |
| 6 | HTTP status `200` | `000` (connection refused), `502` (upstream down), `404` |
| 7 | HTML contains realm names from your AC `realmlist` | Empty page, or SQL error in HTML |
| 8 | HTTP status `200` | `404` — MPQ extraction may not have completed, or path is wrong |
| 9 | HTTP status `200` or `301` (redirect to HTTPS) | `502` — Caddy config error or web container not reachable |

### Common Failure Signatures and Likely Causes

#### `Cannot connect to <db> at <host> — check credentials and network`

**Cause:** Wrong host, wrong credentials, or AC database not reachable from the container.

**Fix:**
- Verify the host resolves from inside the container:
  ```bash
  docker compose exec web ping -c 1 <WORLD_DB_HOST>
  ```
- Test credentials manually:
  ```bash
  docker compose exec web mysql -h<WORLD_DB_HOST> -u<user> -p<pass> -e "SELECT 1" <database>
  ```
- On Linux Docker daemon, `host.docker.internal` may not work. Use the host's actual IP or add `extra_hosts` to `docker-compose.yml`.

#### `DB test failed to find version table. World database not yet imported?`

**Cause:** The AC world database is empty or the `version` table is missing.

**Fix:** Verify the AC world database has the `version` table:
```bash
mysql -h<WORLD_DB_HOST> -u<user> -p<pass> <database> -e "SHOW TABLES LIKE 'version';"
```

#### `WoW.exe not found in client directory!`

**Cause:** `WOW_CLIENT_PATH` is wrong or the mounted path doesn't contain `WoW.exe`.

**Fix:** Verify the mount:
```bash
docker compose exec web ls -la /wow-client/WoW.exe
```

#### `No valid locales found!`

**Cause:** `WOW_LOCALE` value doesn't match a directory in `/wow-client/Data/`.

**Fix:** Check available locales:
```bash
docker compose exec web ls /wow-client/Data/
```

#### HTTP 500 or SQL error in browser

**Cause:** AoWoW query class references a table/column that doesn't exist in the AC schema.

**Fix:** Check the web container error log:
```bash
docker compose exec web tail -100 /var/log/apache2/error.log
```
Look for `SQLSTATE` errors. These indicate schema mismatches that need to be addressed in the PHP query classes.

#### Static asset returns 404

**Cause:** MPQ extraction hasn't completed, or the asset path doesn't exist in the extracted data.

**Fix:** Check if extraction completed:
```bash
docker compose exec web ls /var/www/html/static/images/wow/Interface/
docker compose exec web ls /var/www/html/static/wowsounds/
```

### Minimal Smoke-Test Page List

After the web interface loads, visit these URLs and verify each returns content (not a SQL error):

| URL | What It Tests |
|-----|--------------|
| `http://localhost:8080/` | Homepage loads, no SQL errors |
| `http://localhost:8080/?realm=list` | Realm listing from `realmlist` table |
| `http://localhost:8080/?creature=1` | Creature lookup from `creature_template` |
| `http://localhost:8080/?item=1` | Item lookup from `item_template` |
| `http://localhost:8080/?spell=1` | Spell lookup from `spell` |
| `http://localhost:8080/?quest=1` | Quest lookup from `quest_template` |
| `http://localhost:8080/?achievement=1` | Achievement lookup |
| `http://localhost:8080/static/images/wow/Interface/Buttons/<discovered-file>` | Static asset serving (use a file discovered in Step 8) |
