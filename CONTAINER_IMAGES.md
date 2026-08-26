# Container Images

SwayRider service images are published to the GitHub Container Registry at
`ghcr.io/swayrider/<service>`. The layer-20 Docker Compose files currently pull
the `latest` tag for each service (`infra/dev/layer-20`) or a pinned version
tag (`infra/dev-mini/layer-20`, for reproducible mini-deployments);
`dev-latest` exists as a commented-out alternative but is not active by
default.

Every service ships a `Makefile` with a `container-build` target that builds
both `linux/amd64` and `linux/arm64` and pushes the result. The image tag is
derived from the git state of the service's checkout.

---

## Registry access control

The registry is hosted under the `SwayRider` GitHub organization. Push access is restricted
to organization members with the **Owner** or **Member** role, plus any outside collaborators
explicitly granted write access on a per-package basis.

**To grant or revoke push access:**

1. Go to the package page on GitHub:
   `https://github.com/orgs/SwayRider/packages/container/<service-name>/settings`
2. Under **Manage Actions access**, ensure only trusted repositories and users are listed.
3. Under **Manage access**, add or remove individual collaborators with **Write** role.

Anyone who needs to push images must also authenticate locally (see below). Read access for
pulling is public by default — no login is required to pull images.

---

## Prerequisites

- Docker with the `buildx` plugin (included in Docker Desktop and Docker Engine ≥ 23)
- A GitHub Personal Access Token (PAT) with the `write:packages` scope

**Create a PAT:**

1. Go to <https://github.com/settings/tokens/new>
2. Select **write:packages** (this also requires **read:packages** and **repo**)
3. Copy the token — you will not see it again

---

## Authenticating with the registry

Run once per machine (or when the token is rotated):

```bash
echo "<YOUR_PAT>" | docker login ghcr.io -u <your-github-username> --password-stdin
```

Docker stores the credential in `~/.docker/config.json`. Re-run this command whenever you
rotate your PAT.

---

## Building and pushing images

All images are built for `linux/amd64` and `linux/arm64`. A multi-platform builder must be
active:

```bash
docker buildx create --use --name multiplatform --driver docker-container
```

This only needs to be done once per machine.

### From a service directory

Each service repo (`authservice`, `mailservice`, `regionservice`, `routerservice`,
`searchservice`, `swayrider-api`, `swctl`, `tilesservice`) has its own `Makefile`:

```bash
cd <service>
make container-build
```

Each service's Dockerfile uses the service directory as its build context and relies solely
on `go mod download` — no workspace or sibling modules. The `go.work` file is for local
development only.

### Building all services at once

`tools/containerbuild.py` builds every service in one pass (see `--help` for
`--dry-run`, `--show`, `--no-push`, `--services`, and `--dev-latest`):

```bash
tools/containerbuild.py
```

---

## Tag scheme

The Makefile derives tags from the git state of the checkout:

| Branch / state | Tags applied |
|----------------|--------------|
| Version-tagged commit (`v1.2.3`) | `v1.2.3`, `latest` |
| `main` (untagged) | `v{last}-{date}-dev-b{N}`, `dev-latest` |
| Other branch | `v{last}-{branch}-b{N}` |
| Detached HEAD | `v{last}-{sha}-b{N}` |

- **Release builds** (version tag on HEAD) are immutable: exactly `v1.2.3` + `latest`.
- **Non-release builds** get an incrementing build number (`-b{N}`) so repeated builds of the
  same branch don't overwrite each other. The number is computed by querying the registry
  (`tags/list`) for the highest existing `-b{N}` tag on the same base tag and adding 1. The
  build fails with a clear error if the registry can't be reached.
- `dev-latest` / `latest` are floating tags that point at the most recent build in their
  class; the pinned `-b{N}` tags preserve every build for rollback and comparison.

### FORCE_DEV_LATEST

Only `main` (untagged HEAD) pushes `dev-latest` automatically. Set `FORCE_DEV_LATEST=1` to
also push `dev-latest` from a release build (a version-tagged commit) or from any other
branch:

```bash
FORCE_DEV_LATEST=1 make container-build
```

Or, across all services at once, `tools/containerbuild.py --dev-latest`. Use this when a
release — or a branch build — should also advance environments that track `dev-latest`.

### Releasing a new version

Tag the commit with a `vX.Y.Z` tag and build on that commit:

```bash
git tag v1.2.3
make container-build        # pushes v1.2.3 + latest
```

---

## Image overview

| Service | Image | Compose tag |
|---|---|---|
| swayrider-api | `ghcr.io/swayrider/swayrider-api` | `dev-latest` (dev) / `latest` (prod) |
| authservice | `ghcr.io/swayrider/authservice` | `latest` |
| mailservice | `ghcr.io/swayrider/mailservice` | `latest` |
| regionservice | `ghcr.io/swayrider/regionservice` | `latest` |
| routerservice | `ghcr.io/swayrider/routerservice` | `latest` |
| searchservice | `ghcr.io/swayrider/searchservice` | `latest` |
| tilesservice | `ghcr.io/swayrider/tilesservice` | `latest` |
| swctl | `ghcr.io/swayrider/swctl` | pinned version tag (init job only) |

`swctl`'s image is not run as a long-lived service, but is used as the one-shot
`swayrider-api-register` init-job container in `infra/dev-mini/layer-20/compose.yml`.

---

## Pulling the latest images on the dev server

After pushing new images, pull and restart on the dev server:

```bash
cd ~/Dev/swayrider-public/infra/dev/layer-20
docker compose pull
docker compose up -d
```
