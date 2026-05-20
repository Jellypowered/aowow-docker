# AoWoW Docker Stack

Containerized deployment for AoWoW (WotLK 3.3.5a) connected to an external AzerothCore server.

AoWoW reads from AzerothCore's world, auth, and characters databases. It owns its own dedicated database for internal state (config, cache, user accounts).

## Prerequisites

* World of Warcraft 3.3.5a Client
* Running AzerothCore server with world, auth, and characters databases
* Docker & Docker Compose

## Setup

1.  **Configure Environment**
    ```bash
    cp .env.example .env
    ```
    Edit `.env`. Essential variables:
    * `WOW_CLIENT_PATH`: Path to WoW 3.3.5a installation directory.
    * `WOW_LOCALE`: Space-separated list (e.g., `enUS deDE`).
        * Database credentials for world, auth, and characters databases (must match your AzerothCore deployment).
        * If AzCoreWeb is already configured, copy the matching DB host/user/password values from `/bulk/Source/AzCoreWeb/backend/.env` and keep the AzerothCore DB names aligned with:
            * `acore_world`
            * `acore_auth`
            * `acore_characters`

2.  **Start Containers**
    ```bash
    docker compose up -d --build
    ```

3.  **Wait for Extraction**
    The initialization process extracts MPQs and converts audio. **This takes a significant amount of time.**
    Monitor progress:
    ```bash
    docker compose logs -f web
    ```

## Finalization

Once the logs show `Deployment setup complete!`:

1.  **Run AoWoW Installer**
    ```bash
    docker compose exec web php aowow --setup
    ```

2.  **Cleanup (Free Disk Space)**
    ```bash
    docker compose exec web rm -rf setup/mpqdata
    ```

3.  **Access**
    Visit `http://localhost:8080` (or configured `WEB_PORT`).

## Database Ownership

| Database | Owned By | Managed By |
|----------|----------|------------|
| AoWoW | AoWoW service | aowow-docker (creates DB, imports schema) |
| World | AzerothCore | External AC deployment (read-only access) |
| Auth | AzerothCore | External AC deployment (read-only access) |
| Characters | AzerothCore | External AC deployment (read-only access) |

AoWoW never creates, modifies, or patches AzerothCore databases.

The `.env.example` defaults are aligned with AzCoreWeb's AzerothCore database names so the handoff between the two repos is straightforward.

## Reverse Proxy (Caddy)

A Caddy reverse proxy is provided as an optional gateway for SSL termination.

```bash
docker compose -f docker-compose.yml -f docker-compose.caddy.yml up -d
```

Configure `Caddyfile` with your domain name. The Caddy container proxies to `aowow_web:80` on the private network.

## Deployment Guide

For detailed runtime validation, troubleshooting, and AzCoreWeb integration instructions, see [DEPLOYMENT.md](DEPLOYMENT.md).