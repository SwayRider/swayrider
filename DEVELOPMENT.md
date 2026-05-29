# Development Guide

This guide covers day-to-day local development against the shared dev server.

---

## Prerequisites

- Go workspace set up — `go.work` in `~/Dev/swayrider-public/` covering all repos you are
  working on (see [README.md](README.md#go-workspace))
- `swctl` available — `go run ./swctl/cmd/swctl` from the repo root, or build and add to `PATH`
- WireGuard connected (see below)

---

## Connecting via WireGuard

The dev server runs a [wg-easy](https://github.com/wg-easy/wg-easy) VPN.

1. Open the wg-easy admin UI:
   `http://<DEV_IP>:51821` — or `https://vpn.<DEV_DOMAIN>` if HTTPS is enabled
2. Log in with the admin password
3. Click **+ New** to create a client config for your machine
4. Download the generated `.conf` file
5. Import and activate it:
   ```bash
   # Linux
   sudo wg-quick up /path/to/swayrider-dev.conf

   # Mac / Windows — use the official WireGuard GUI app
   ```
6. Verify connectivity:
   ```bash
   ping <DEV_IP>
   ```

Once connected, all dev server ports listed below are reachable at `<DEV_IP>`.

---

## Dev Server Reference

All ports are on the dev server host. Replace `<DEV_IP>` with the server's LAN IP.

### Layer 00 — Infrastructure

| Service | Port | Protocol |
|---------|------|----------|
| PostgreSQL | 35432 | TCP |
| Redis | 36379 | TCP |
| Elasticsearch | 39200 | HTTP |
| WireGuard UI | 51821 | HTTP |
| Traefik | 30080 / 30443 | HTTP / HTTPS |

### Layer 10 — Geospatial engines

| Service | Port range | Notes |
|---------|------------|-------|
| Valhalla | 33001–33008 | One per region |
| Pelias API | 33111–33181 | One per region |

Check `infra/dev-mini/layer-10/.env` for the exact region-to-port mapping.

### Layer 20 — Backend services

| Service | HTTP | gRPC |
|---------|------|------|
| authservice | 34001 | 34101 |
| mailservice | 34002 | 34102 |
| regionservice | 34003 | 34103 |
| routerservice | 34004 | 34104 |
| searchservice | 34007 | 34107 |
| tilesservice | 34005 | — |

### Layer 30 — API gateway

| Service | HTTP |
|---------|------|
| swayrider-api | 34000 |

---

## Local Port Assignments

When running all layer-20 services at once, each needs a unique port. The suggested
non-conflicting local assignments (override `HTTP_PORT`/`GRPC_PORT` in each `.env`):

| Service | HTTP | gRPC | Web |
|---------|------|------|-----|
| authservice | 8080 | 8081 | 8000 |
| mailservice | 8082 | 8083 | — |
| regionservice | 8084 | 8085 | — |
| routerservice | 8086 | 8087 | — |
| searchservice | 8088 | 8089 | — |
| tilesservice | 8090 | — | — |
| swayrider-api | 8888 | — | — |

swayrider-api is never run alongside layer-20 on the same machine, so it reuses 8080.

---

## Scenario A — Backend development (all layer-20 local)

Run all six layer-20 services locally and call them directly — no layer-30 involved.
The shared local authservice ensures all services use the same JWT signing keys.

### Service dependency table

| Service | Calls locally | Calls remotely |
|---------|---------------|----------------|
| authservice | mailservice :8083 | PostgreSQL `<DEV_IP>:35432` |
| mailservice | authservice :8081 | SMTP (external) |
| regionservice | — (geodata files) | — |
| routerservice | authservice :8081, regionservice :8085 | Valhalla, Pelias |
| searchservice | authservice :8081, regionservice :8085 | Pelias |
| tilesservice | authservice :8081 | — (MBTiles files) |

### Syncing data files

regionservice and tilesservice need local copies of server data. Data can be downloaded
from **https://geodata.hevanto-it.com** — request login credentials from the team before
your first download.

**regionservice** — download `border.tar.bz2` and extract into `localdata/geodata`:

```bash
mkdir -p ~/Dev/SwayRider/localdata/geodata
tar -xjf border.tar.bz2 -C ~/Dev/SwayRider/localdata/geodata
```

This produces `manifest.yml`, `contours/`, and `border-crossings/` under `localdata/geodata`.

**tilesservice** — download `tiles.tar` and extract into `localdata/tiles`:

```bash
mkdir -p ~/Dev/SwayRider/localdata/tiles
tar -xf tiles.tar -C ~/Dev/SwayRider/localdata/tiles
```

This produces `L0.mbtiles`, `L1/`, and `L2/` under `localdata/tiles`.

### Service setup

For each service: copy `env.example` to `.env`, then set the values below.

**authservice**

```env
HTTP_PORT=8080
GRPC_PORT=8081
WEB_PORT=8000
DB_HOST=<DEV_IP>
DB_PORT=35432
DB_USER=postgresadmin
DB_PASSWORD=<same as layer-00>
DB_NAME=authdb
DB_SSL_MODE=disable
ADMIN_EMAIL=admin@example.com
ADMIN_PASSWORD=<admin password>
MAILER_ADDRESS=swayrider@example.com
MAILSERVICE_HOST=localhost
MAILSERVICE_PORT=8083
```

**mailservice**

```env
HTTP_PORT=8082
GRPC_PORT=8083
AUTHSERVICE_HOST=localhost
AUTHSERVICE_GRPC_PORT=8081
SMTP_HOST=<your SMTP host>
SMTP_PORT=587
SMTP_USER=<smtp user>
SMTP_PASSWORD=<smtp password>
```

**regionservice**

```env
HTTP_PORT=8084
GRPC_PORT=8085
GEODATA_DIR=/home/<user>/dev/swayrider/localdata/geodata
```

**routerservice**

```env
HTTP_PORT=8086
GRPC_PORT=8087
AUTHSERVICE_HOST=localhost
AUTHSERVICE_PORT=8081
REGIONSERVICE_HOST=localhost
REGIONSERVICE_PORT=8085
VALHALLA_REGION_HOSTS=benelux:<DEV_IP>,france:<DEV_IP>,germany:<DEV_IP>
VALHALLA_REGION_PORTS=benelux:33001,france:33002,germany:33003
PELIAS_API_REGION_HOSTS=benelux:<DEV_IP>,france:<DEV_IP>,germany:<DEV_IP>
PELIAS_API_REGION_PORTS=benelux:33111,france:33121,germany:33131
```

**searchservice**

```env
HTTP_PORT=8088
GRPC_PORT=8089
AUTHSERVICE_HOST=localhost
AUTHSERVICE_PORT=8081
REGIONSERVICE_HOST=localhost
REGIONSERVICE_PORT=8085
PELIAS_REGIONS=benelux=http://<DEV_IP>:33111/v1,france=http://<DEV_IP>:33121/v1,germany=http://<DEV_IP>:33131/v1
```

**tilesservice**

```env
HTTP_PORT=8090
AUTHSERVICE_HOST=localhost
AUTHSERVICE_PORT=8081
TILES_PATH=/home/<user>/dev/swayrider/localdata/tiles
SERVICE_HOST=http://localhost
SERVICE_PORT=8090
SERVICE_PREFIX=/v1/tiles
```

### Running all services

Run each service from its own directory (`.env` is loaded from the working directory):

```bash
cd authservice   && go run ./cmd/authservice   &
cd mailservice   && go run ./cmd/mailservice   &
cd regionservice && go run ./cmd/regionservice &
cd routerservice && go run ./cmd/routerservice &
cd searchservice && go run ./cmd/searchservice &
cd tilesservice  && go run ./cmd/tilesservice  &
```

### Optional: also run swayrider-api locally (A+B)

Add to `swayrider-api/.env`:

```env
AUTHSERVICE_HOST=localhost
AUTHSERVICE_PORT=8081
ROUTERSERVICE_HOST=localhost
ROUTERSERVICE_PORT=8087
SEARCHSERVICE_HOST=localhost
SEARCHSERVICE_PORT=8089
REGIONSERVICE_HOST=localhost
REGIONSERVICE_PORT=8085
TILESSERVICE_HOST=localhost
TILESSERVICE_PORT=8090
REDIS_HOST=<DEV_IP>
REDIS_PORT=36379
SWAYRIDER_API_CLIENT_ID=<see credential registration below>
SWAYRIDER_API_CLIENT_SECRET=<see credential registration below>
```

Then run:

```bash
cd swayrider-api && go run ./cmd/swayrider-api
```

Credentials must be registered against the **local** authservice — see
[Registering service credentials](#registering-service-credentials) below, but use
`--auth-host localhost --auth-port 8081`.

---

## Scenario A2 — Backend development without authservice (no DB credentials needed)

Run mailservice, regionservice, routerservice, searchservice, and tilesservice locally
while keeping authservice on the remote dev server. No PostgreSQL credentials required —
all local services validate JWTs against the remote authservice.

### Service dependency table

| Service | Calls locally | Calls remotely |
|---------|---------------|----------------|
| mailservice | — | authservice `<DEV_IP>:34101` |
| regionservice | — (geodata files) | — |
| routerservice | regionservice :8085 | authservice `<DEV_IP>:34101`, Valhalla, Pelias |
| searchservice | regionservice :8085 | authservice `<DEV_IP>:34101`, Pelias |
| tilesservice | — (MBTiles files) | authservice `<DEV_IP>:34101` |

### Syncing data files

Same as Scenario A — see [above](#syncing-data-files) for archive names and extraction commands.

### Service setup

**mailservice**

```env
HTTP_PORT=8082
GRPC_PORT=8083
AUTHSERVICE_HOST=<DEV_IP>
AUTHSERVICE_GRPC_PORT=34101
SMTP_HOST=<your SMTP host>
SMTP_PORT=587
SMTP_USER=<smtp user>
SMTP_PASSWORD=<smtp password>
```

**regionservice**

```env
HTTP_PORT=8084
GRPC_PORT=8085
GEODATA_DIR=/home/<user>/dev/swayrider/localdata/geodata
```

**routerservice**

```env
HTTP_PORT=8086
GRPC_PORT=8087
AUTHSERVICE_HOST=<DEV_IP>
AUTHSERVICE_PORT=34101
REGIONSERVICE_HOST=localhost
REGIONSERVICE_PORT=8085
VALHALLA_REGION_HOSTS=benelux:<DEV_IP>,france:<DEV_IP>,germany:<DEV_IP>
VALHALLA_REGION_PORTS=benelux:33001,france:33002,germany:33003
PELIAS_API_REGION_HOSTS=benelux:<DEV_IP>,france:<DEV_IP>,germany:<DEV_IP>
PELIAS_API_REGION_PORTS=benelux:33111,france:33121,germany:33131
```

**searchservice**

```env
HTTP_PORT=8088
GRPC_PORT=8089
AUTHSERVICE_HOST=<DEV_IP>
AUTHSERVICE_PORT=34101
REGIONSERVICE_HOST=localhost
REGIONSERVICE_PORT=8085
PELIAS_REGIONS=benelux=http://<DEV_IP>:33111/v1,france=http://<DEV_IP>:33121/v1,germany=http://<DEV_IP>:33131/v1
```

**tilesservice**

```env
HTTP_PORT=8090
AUTHSERVICE_HOST=<DEV_IP>
AUTHSERVICE_PORT=34101
TILES_PATH=/home/<user>/dev/swayrider/localdata/tiles
SERVICE_HOST=http://localhost
SERVICE_PORT=8090
SERVICE_PREFIX=/v1/tiles
```

### Running all services

```bash
cd mailservice   && go run ./cmd/mailservice   &
cd regionservice && go run ./cmd/regionservice &
cd routerservice && go run ./cmd/routerservice &
cd searchservice && go run ./cmd/searchservice &
cd tilesservice  && go run ./cmd/tilesservice  &
```

### Optional: also run swayrider-api locally

```env
AUTHSERVICE_HOST=<DEV_IP>
AUTHSERVICE_PORT=34101
ROUTERSERVICE_HOST=localhost
ROUTERSERVICE_PORT=8087
SEARCHSERVICE_HOST=localhost
SEARCHSERVICE_PORT=8089
REGIONSERVICE_HOST=localhost
REGIONSERVICE_PORT=8085
TILESSERVICE_HOST=localhost
TILESSERVICE_PORT=8090
REDIS_HOST=<DEV_IP>
REDIS_PORT=36379
SWAYRIDER_API_CLIENT_ID=<from registration — see Scenario B>
SWAYRIDER_API_CLIENT_SECRET=<from registration — see Scenario B>
```

---

## Scenario B — API gateway development (swayrider-api local only)

Run only swayrider-api locally. All layer-20 services stay on the dev server.

### Registering service credentials

swayrider-api authenticates to backend services using a service client credential issued
by authservice. Register it once against the remote authservice:

```bash
swctl auth create-service-client \
  --auth-host <DEV_IP> --auth-port 34101 \
  --user admin@example.com --password <admin password> \
  swayrider-api region:query routing:execute search:execute tiles:serve
```

Copy the returned `clientId` and `clientSecret` into `swayrider-api/.env`.

### Service setup

```env
AUTHSERVICE_HOST=<DEV_IP>
AUTHSERVICE_PORT=34101
ROUTERSERVICE_HOST=<DEV_IP>
ROUTERSERVICE_PORT=34104
SEARCHSERVICE_HOST=<DEV_IP>
SEARCHSERVICE_PORT=34107
REGIONSERVICE_HOST=<DEV_IP>
REGIONSERVICE_PORT=34103
TILESSERVICE_HOST=<DEV_IP>
TILESSERVICE_PORT=34005
REDIS_HOST=<DEV_IP>
REDIS_PORT=36379
SWAYRIDER_API_CLIENT_ID=<from registration above>
SWAYRIDER_API_CLIENT_SECRET=<from registration above>
CORS_ALLOWED_ORIGINS=http://localhost:5173
```

### Running

```bash
cd swayrider-api && go run ./cmd/swayrider-api
```

The gateway starts on `http://localhost:8080`.
