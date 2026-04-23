# Data Pipeline

> **Server only.** The pipeline is CPU- and disk-intensive. Do not run it on a developer
> workstation. A full Europe build requires approximately **1.5 TB of free disk space** and
> **64 GB of RAM**. This is best run on one or more dedicated servers.

The data pipeline builds all geodata required by the SwayRider backend:

| Output | Consumed by |
|---|---|
| Vector tile MBTiles | `tilesservice` |
| Valhalla routing graph tiles | `routerservice` (via Valhalla containers in layer-10) |
| Pelias geocoding data | `searchservice` (via Pelias containers in layer-10) |
| Border-crossing metadata | `regionservice` |
| Per-region OSM extracts | Valhalla, Pelias, border pipelines |

---

## Prerequisites

Install on the build machine:

```bash
sudo apt-get install -y osmium-tool git cmake make tar
```

Python ≥ 3.12:

```bash
sudo snap install python312   # or pyenv / distro package
```

Python dependencies (in the `data-pipeline` repo):

```bash
cd ~/Dev/swayrider-public/data-pipeline
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

---

## Configuration

Config files live in `data-pipeline/config/`. The default is `config/config.yml`; all scripts
accept `--config` to point at an alternate file.

Key sections to edit before the first run:

| Section | Key fields | Description |
|---|---|---|
| `build_paths` | `download_dir`, `temp_dir`, `tools_dir`, `result_dir` | Local scratch paths — must exist and have sufficient disk space |
| `output` | `geodata_dir` | Where `./publish` copies finished artifacts |
| `max_workers` | — | Parallelism degree; set to CPU count or lower |

The pipeline compiles Valhalla and Tippecanoe from source on first run and caches the
binaries in `tools_dir`. Subsequent runs reuse them.

---

## Pipeline overview

Five independent pipelines share the same config and track progress in separate manifest
files so they can run and be re-run independently.

| Pipeline | Script | Manifest | Output archive |
|---|---|---|---|
| OSM data | `./build-osm` | `manifest-osm.yml` | `osm.tar.bz2` |
| Border data | `./build-border-data` | `manifest-border.yml` | `border.tar.bz2` |
| Valhalla routing data | `./build-valhalla-data` | `manifest-valhalla.yml` | `valhalla.tar.bz2` |
| Pelias geocoding data | `./build-pelias-data` | `manifest-pelias.yml` | `pelias-data.tar.bz2` + `pelias-es-snapshot.tar.bz2` |
| Tiles | `./build-tiles` | `manifest-tiles.yml` | `tiles.tar` |

### Dependencies

```
build-osm  ──┬──▶  build-border-data
             ├──▶  build-valhalla-data
             └──▶  build-pelias-data

build-tiles  (independent — runs in parallel or separately)
```

`build-border-data`, `build-valhalla-data`, and `build-pelias-data` verify that the required
OSM output files from `build-osm` are present before starting.

---

## Running the pipeline

All scripts accept `--tag TAG` (defaults to today's date `YYYY-MM-DD`) which is used as the
version label in manifests and output paths. Use a consistent tag across all five scripts in
one run.

```bash
cd ~/Dev/swayrider-public/data-pipeline
source .venv/bin/activate

# Step 1 — always first
./build-osm --config config/config.yml --tag 2026-04-23

# Step 2 — these three can run in parallel or in any order
./build-border-data   --config config/config.yml --tag 2026-04-23
./build-valhalla-data --config config/config.yml --tag 2026-04-23
./build-pelias-data   --config config/config.yml --tag 2026-04-23

# Step 3 — independent, can run at any time alongside the above
./build-tiles --config config/config.yml --tag 2026-04-23
```

Each pipeline is incremental: completed steps are recorded in the manifest file and skipped
on re-runs. To force a full rebuild of a pipeline, pass `--clean`.

---

## Publishing artifacts

After a pipeline completes, publish its output to `geodata_dir` (configured in `config.yml`):

```bash
# Publish each pipeline's output (run after each script, or all at the end)
./publish --config config/config.yml --manifest manifest-osm.yml
./publish --config config/config.yml --manifest manifest-border.yml
./publish --config config/config.yml --manifest manifest-valhalla.yml
./publish --config config/config.yml --manifest manifest-pelias.yml
./publish --config config/config.yml --manifest manifest-tiles.yml
```

The published layout under `geodata_dir`:

```
geodata/
├── data/{tag}/
│   ├── osm/
│   ├── borders/
│   ├── border-crossings/
│   ├── valhalla/{region}/
│   ├── pelias/
│   └── manifest-*.yml
└── tiles/{tag}/
    ├── tiles.tar
    └── manifest-tiles.yml
```

---

## Transferring output to the dev server

SCP the published artifacts to a staging directory on the dev server:

```bash
scp -r geodata/ user@dev-server:/path/to/staging/
```

The staging directory must contain the tar archives and all `manifest-*.yml` files — the
deploy scripts read the manifests to know which files to extract and where.

---

## Deploying on the dev server

The deploy scripts live in `infra/dev/scripts/`. They read destination paths from the layer
`.env` files, so those must be configured first (see the layer-00 and layer-10 setup in the
main README).

### Deploy everything at once

```bash
cd ~/Dev/swayrider-public/infra/dev/scripts

# Dry run first (recommended)
./deploy-all.sh --input /path/to/staging --dry-run

# Apply
./deploy-all.sh --input /path/to/staging
```

`deploy-all.sh` runs the individual scripts in dependency order. Elasticsearch (layer-00)
must be running before deploying Pelias data.

### Deploy individual components

```bash
./deploy-osm.sh     --input /path/to/staging
./deploy-border.sh  --input /path/to/staging
./deploy-valhalla.sh --input /path/to/staging
./deploy-pelias.sh  --input /path/to/staging --es-host localhost --es-port 39200
./deploy-tiles.sh   --input /path/to/staging
```

### What goes where

| Data | Env var | Destination |
|---|---|---|
| OSM | `GEODATA_PATH` | `{GEODATA_PATH}/osm/` |
| Border | `GEODATA_PATH` | `{GEODATA_PATH}/` (borders + border-crossings) |
| Valhalla | `VALHALLA_DATA_PATH` | `{VALHALLA_DATA_PATH}/{region}/` |
| Pelias ES snapshot | `ES_SNAPSHOTS_PATH` | `{ES_SNAPSHOTS_PATH}/` + ES API restore |
| Pelias data | `PELIAS_DATA_PATH` | `{PELIAS_DATA_PATH}/{region}/` |
| Tiles | `TILES_DATA_PATH` | `{TILES_DATA_PATH}/` |

---

## Restarting services after deployment

After deploying new data, restart the affected Docker services:

```bash
# Valhalla + Pelias (layer-10)
cd ~/Dev/swayrider-public/infra/dev/layer-10
docker compose restart

# regionservice + tilesservice (layer-20)
cd ~/Dev/swayrider-public/infra/dev/layer-20
docker compose restart
```

| Data updated | Services to restart |
|---|---|
| OSM | None |
| Border | `regionservice` (layer-20) |
| Valhalla | Valhalla containers (layer-10) |
| Pelias | Pelias containers (layer-10) |
| Tiles | `tilesservice` (layer-20) |
