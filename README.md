# SwayRider

A multi-region routing, geocoding, and geolocation platform built with Go microservices,
backed by Valhalla (routing), Pelias (geocoding), and PostgreSQL.

## Repository Layout

| GitHub Repo | Language | Purpose |
|---|---|---|
| [SwayRider/swayrider](https://github.com/SwayRider/swayrider) | — | This repo — setup docs |
| [SwayRider/protos](https://github.com/SwayRider/protos) | Go / protobuf | gRPC protobuf definitions |
| [SwayRider/grpcclients](https://github.com/SwayRider/grpcclients) | Go | gRPC client wrappers |
| [SwayRider/swlib](https://github.com/SwayRider/swlib) | Go | Shared library (app bootstrap, auth, logging) |
| [SwayRider/swayrider-api](https://github.com/SwayRider/swayrider-api) | Go | API gateway — JWT validation, rate limiting, queue |
| [SwayRider/swayrider-api](https://github.com/SwayRider/swayrider-api) | Go | API gateway (sole external entry point) |
| [SwayRider/authservice](https://github.com/SwayRider/authservice) | Go | User auth & JWT |
| [SwayRider/mailservice](https://github.com/SwayRider/mailservice) | Go | Transactional email |
| [SwayRider/regionservice](https://github.com/SwayRider/regionservice) | Go | Geographic region queries |
| [SwayRider/routerservice](https://github.com/SwayRider/routerservice) | Go | Multi-modal routing |
| [SwayRider/searchservice](https://github.com/SwayRider/searchservice) | Go | Geocoding / search |
| [SwayRider/tilesservice](https://github.com/SwayRider/tilesservice) | Go | Vector tile serving |
| [SwayRider/swlib](https://github.com/SwayRider/swlib) | Go | Shared library used by every service |
| [SwayRider/grpcclients](https://github.com/SwayRider/grpcclients) | Go | gRPC client wrappers per service |
| [SwayRider/protos](https://github.com/SwayRider/protos) | Protocol Buffers | Proto definitions and generated Go code |
| [SwayRider/swctl](https://github.com/SwayRider/swctl) | Go | Management CLI |
| [SwayRider/swayriderapp](https://github.com/SwayRider/swayriderapp) | Dart / Flutter | Mobile app |
| [SwayRider/data-pipeline](https://github.com/SwayRider/data-pipeline) | Python | OSM tile data pipeline |
| [SwayRider/infra](https://github.com/SwayRider/infra) | — | Docker Compose, deploy scripts |
| [SwayRider/testing](https://github.com/SwayRider/testing) | — | Bruno API test collections |

---

## System Requirements (Linux)

### Core tools

Install these on every developer machine:

```bash
# Debian/Ubuntu
sudo apt-get install -y \
  protobuf-compiler \
  lz4 \
  build-essential \
  cmake \
  git \
  curl
```

For other distros:

| Tool | Arch | Fedora/RHEL | openSUSE |
|------|------|------------|---------|
| protoc | `protobuf` | `protobuf` | `protobuf` |
| lz4 | `lz4` | `lz4` | `lz4` |

### Go

Install Go ≥ 1.26.2 (the workspace currently pins `go 1.26.4` in `go.work`). The recommended approach is snap or direct from <https://go.dev/dl/>:

```bash
# snap (recommended)
sudo snap install go --classic

# Direct install
curl -L https://go.dev/dl/go1.26.4.linux-amd64.tar.gz | sudo tar -C /usr/local -xz
export PATH=$PATH:/usr/local/go/bin  # add to ~/.profile
```

### Docker

Install Docker Engine and the Compose plugin:

```bash
# https://docs.docker.com/engine/install/ubuntu/
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER   # then log out and back in
```

### Go tool dependencies (protoc plugins)

Run once after installing Go:

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway@latest
go install github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2@latest
```

Or use the shortcut from the `infra` repo (also fetches googleapis):

```bash
cd ~/Dev/swayrider-public/infra && make install-deps
```

### Python (data pipeline only)

Python ≥ 3.12 is required for the data pipeline. Use [pyenv](https://github.com/pyenv/pyenv)
or your distro's package manager. The pipeline also requires `tippecanoe` for tile generation
and `osmium-tool` for OSM data extraction — both are **server-side only**.

### Flutter (mobile only)

The mobile client (`swayriderapp`) is a Flutter app — see its own README/DEVELOPMENT.md for the Flutter/Dart SDK versions and setup instructions.

---

## Cloning All Repositories

```bash
mkdir -p ~/Dev/swayrider-public && cd ~/Dev/swayrider-public

# Core Go modules (required for all backend work)
git clone git@github.com:SwayRider/protos.git
git clone git@github.com:SwayRider/grpcclients.git
git clone git@github.com:SwayRider/swlib.git

# API gateway
git clone git@github.com:SwayRider/swayrider-api.git

# Backend services (clone whichever you need)
git clone git@github.com:SwayRider/authservice.git
git clone git@github.com:SwayRider/mailservice.git
git clone git@github.com:SwayRider/regionservice.git
git clone git@github.com:SwayRider/routerservice.git
git clone git@github.com:SwayRider/searchservice.git
git clone git@github.com:SwayRider/tilesservice.git
git clone git@github.com:SwayRider/swctl.git

# Infrastructure & ops
git clone git@github.com:SwayRider/infra.git

# Optional
git clone git@github.com:SwayRider/swayriderapp.git
git clone git@github.com:SwayRider/data-pipeline.git
```

---

## Go Workspace

Create a `go.work` file in `~/Dev/swayrider-public/`. This file is **not committed** to any
repo — each developer creates their own based on which repos they have cloned.

```bash
cd ~/Dev/swayrider-public

go work init
go work use \
  ./protos \
  ./grpcclients \
  ./swlib \
  ./swayrider-api \
  ./authservice \
  ./mailservice \
  ./regionservice \
  ./routerservice \
  ./searchservice \
  ./tilesservice \
  ./swctl
```

The workspace lets `go build` and `go test` resolve sibling repos from disk instead of fetching
published versions, which is essential for cross-module development.

---

## Building the Backend

All services depend on `protos`, `grpcclients`, and `swlib`. With `go.work` in place you can
build any service directly, but follow this order for a clean first build:

### 1. Generate protobuf code

Run once, or after any `.proto` file changes:

```bash
cd ~/Dev/swayrider-public/protos && make
```

This regenerates the `.pb.go` files committed in the `protos` repo and must be done before
building any service that imports them.

### 2. Build order

```
protos        → go build ./...   (no swayrider deps)
grpcclients   → go build ./...   (depends on protos)
swlib         → go build ./...   (depends on grpcclients + protos)
swayrider-api → go build ./...   (depends on swlib + grpcclients)
services      → go build ./...   (depend on swlib + grpcclients + protos)
swctl         → go build ./...   (depends on grpcclients + swlib)
```

```bash
for repo in protos grpcclients swlib swayrider-api authservice mailservice \
            regionservice routerservice searchservice tilesservice swctl; do
  echo "--- $repo ---"
  cd ~/Dev/swayrider-public/$repo && go build ./...
done
```

### 3. Run tests

```bash
cd ~/Dev/swayrider-public/swlib && go test ./...
# Repeat per service repo as needed
```

---

## Setting Up the Dev Server

The dev server runs all shared infrastructure and SwayRider services in Docker. The
`infra` repo contains four layered Compose configurations that must be started in order.

### Layer 00 — Base infrastructure

Services: **Traefik**, **PostgreSQL**, **Redis**, **Elasticsearch**, **WireGuard**

```bash
cd ~/Dev/swayrider-public/infra/dev-mini/layer-00
cp env.example .env
# Edit .env — set data paths for your server
docker compose -f compose.yaml up -d
```

Key variables in `layer-00/.env`:

```env
ES_DATA_PATH=/path/to/elasticsearch/data
ES_SNAPSHOTS_PATH=/path/to/elasticsearch/snapshots
POSTGRES_DATA_PATH=/path/to/postgres/data

POSTGRES_USER=postgresadmin
POSTGRES_PASSWORD=<choose a password>
POSTGRES_DB=swayriderdb
```

Exposed ports: PostgreSQL `35432`, Elasticsearch `39200/39300`, Redis `36379`,
Traefik `30080/30443` (HTTP/HTTPS), `38080` (dashboard), WireGuard `51820/udp` (or `WG_UDP_PORT`).

### Database migrations

After layer-00 is up, run the AuthService migrations. The Makefile loads `authservice/.env`,
uses `envsubst` to substitute the connection variables into `dbconfig.yml.in` producing
`dbconfig.yml`, and passes that to `sql-migrate`. Configure `.env` first:

```env
DB_HOST=<dev-server-ip>
DB_PORT=35432
DB_USER=postgresadmin
DB_PASSWORD=<same as layer-00>
DB_NAME=authdb
DB_SSL_MODE=disable
```

Then run:

```bash
cd ~/Dev/swayrider-public/authservice
cp env.example .env   # if not done yet, then fill in the values above
make migrate-up
```

The `migrate-up` target also creates the `authdb` database automatically if it does not exist.

### Layer 10 — Geospatial engines

Services: **Valhalla** (8 routing regions), **Pelias** (geocoding)

> Layer 10 requires that regional OSM data and Valhalla tiles have already been deployed to
> the data paths (see [Data Pipeline](#data-pipeline) below).

```bash
cd ~/Dev/swayrider-public/infra/dev-mini/layer-10
cp env.example .env
# Edit .env — set data paths and thread count
docker compose -f compose.yaml up -d
```

Key variables in `layer-10/.env`:

```env
VALHALLA_DATA_PATH=/path/to/valhalla
GEODATA_PATH=/path/to/geodata
PELIAS_DATA_PATH=/path/to/pelias
VALHALLA_SERVER_THREADS=2
```

Exposed ports: Valhalla instances `33001–33008` (one per region),
Pelias API `33111–33181`, Pelias Placeholder `33100`.

### Layer 20 — SwayRider services

> **Before starting layer-20**, the data pipeline must have run and its output deployed to the
> server data paths. See [DATAPIPELINE.md](DATAPIPELINE.md) for the full procedure.

Services: authservice, mailservice, regionservice, routerservice, searchservice, tilesservice.

```bash
cd ~/Dev/swayrider-public/infra/dev-mini/layer-20
cp env.example .env
# Edit .env — see below
docker compose -f compose.yml up -d
```

Key variables in `layer-20/.env`:

```env
POSTGRES_USER=postgresadmin
POSTGRES_PASSWORD=<same as layer-00>

GEODATA_PATH=/path/to/geodata
TILES_DATA_PATH=/path/to/tilesdata
TILES_CACHE_PATH=/path/to/tilescache

AUTH_ADMIN_EMAIL=admin@example.com
AUTH_ADMIN_PASSWORD=<initial admin password>
AUTH_MAILER_ADDRESS=noreply@example.com

MAIL_SMTP_USER=noreply@example.com
MAIL_SMTP_PASSWORD=<smtp password>
```

Exposed ports:

| Service | HTTP | gRPC |
|---|---|---|
| authservice | 34001 | 34101 |
| mailservice | 34002 | 34102 |
| regionservice | 34003 | 34103 |
| routerservice | 34004 | 34104 |
| searchservice | 34006 | 34106 |
| tilesservice | 34005 | — |

### Layer 30 — Public gateway

Services: **swayrider-api**

> **Before starting layer-30**, layer-20 must be up and the `swayrider-api-register`
> init job must have completed. That job registers API credentials and writes them to
> the `sw-dev-api-credentials` Docker volume that layer-30 mounts read-only.

```bash
cd ~/Dev/swayrider-public/infra/dev-mini/layer-30
cp env.example .env
# Edit .env — set domain and CORS origins
docker compose -f compose.yml up -d
```

Key variables in `layer-30/.env`:

```env
DEV_DOMAIN=<must match layer-00>
CORS_ALLOWED_ORIGINS=http://localhost:5173

ROUTE_WORKER_COUNT=5
SEARCH_WORKER_COUNT=10
QUEUE_MAX_DEPTH=500
RESULT_TTL_SECONDS=300

RATE_LIMIT_IP_AUTH=10
RATE_LIMIT_IP_PUBLIC=600
RATE_LIMIT_USER_API=300
RATE_LIMIT_USER_EXPENSIVE=20
```

Exposed ports: `swayrider-api` direct access `34000`. Also reachable via Traefik at
`api.<DEV_DOMAIN>` (port 30080/30443) — it is the only service attached to the
Traefik-facing network.

---

## Running a Local Service Against the Dev Server

Each service repo contains a detailed local development guide. The short version:

1. Copy `env.example` to `.env` in the service repo.
2. Set `*_HOST` and `*_PORT` env vars to the dev server's IP address.
3. Run:
   ```bash
   go run ./cmd/<servicename>
   ```

Services communicate exclusively over gRPC. When running a service locally, it connects to
its dependencies (other services, database, Valhalla, Pelias) on the dev server via the
configured host/port env vars.

See the individual service repos for the exact environment variables.

---

## Data Pipeline

> **Server only.** The pipeline is CPU- and disk-intensive. Do not run it on a developer
> workstation.

The `data-pipeline` repo processes OpenStreetMap PBF data into:
- MBTiles files served by `tilesservice` (road network, water, landuse, place labels)
- Valhalla routing tiles
- Pelias geocoding indices (loaded into Elasticsearch)

### Setup (on the server)

```bash
cd ~/Dev/swayrider-public/data-pipeline
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

### Running the tile pipeline

```bash
./build-tiles --config config/config.yml --tag <TAG>
```

Each pipeline run is incremental — completed steps are recorded in the manifest and skipped on re-run. Pass `--clean` to force a full rebuild.

### Deployment scripts

The `infra` repo includes scripts that (re)deploy individual data components:

```bash
cd ~/Dev/swayrider-public/infra/dev/scripts
./deploy-all.sh        # full redeploy (all regions, all components)
./deploy-valhalla.sh   # Valhalla routing tiles only
./deploy-pelias.sh     # Pelias geocoding data only
./deploy-tiles.sh      # MBTiles for tilesservice only
./deploy-osm.sh        # base OSM extract only
./deploy-border.sh     # Border crossing data only
```

---

## Mobile (Flutter)

The mobile client (`swayriderapp`) is a Flutter app, not the Kotlin/Jetpack Compose
Android prototype it originally started as.

```bash
cd ~/Dev/swayrider-public/swayriderapp
flutter run   # defaults to main_development.dart, wired against the dev AuthService
```

See the [swayriderapp repo](https://github.com/SwayRider/swayriderapp) README and
`DEVELOPMENT.md` for the Flutter/Dart SDK versions, build flavors, and how to point
it at the dev server.

---

## Troubleshooting

**`protoc-gen-go: program not found`**
Make sure `$GOPATH/bin` (default: `~/go/bin`) is in your `PATH`.

**`go: cannot find module providing package github.com/swayrider/...`**
Ensure `go.work` exists at `~/Dev/swayrider-public/` and lists the relevant repos with
`go work use`.

**Service fails to start — missing env var**
Each service logs missing required env vars on startup. Check the service's `env.example`
for a complete list.

**Layer-10 services crash immediately**
The geospatial data paths (Valhalla tiles, Pelias index) must exist and be populated before
starting layer-10. Run the data pipeline and deploy scripts first.
