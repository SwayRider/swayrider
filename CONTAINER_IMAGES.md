# Container Images

SwayRider service images are published to the GitHub Container Registry at
`ghcr.io/swayrider/<service>`. The layer-20 Docker Compose file pulls the `latest` tag for
each service.

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

### Services with a Makefile (`authservice`, `mailservice`)

The Makefile derives the image tag from the current branch and the `.version` file:

| Branch | Tags applied |
|---|---|
| `main` | `v{version}`, `latest` |
| `dev` | `v{version}-{date}-dev`, `dev-latest` |
| `test` | `v{version}-{date}-test`, `test-latest` |
| other | `v{version}-{date}-{branch}` |

Build and push from the service directory:

```bash
cd ~/Dev/swayrider-public/authservice
make container-build
```

The Makefile builds from the workspace root (`../../`) so that the Go workspace and all
sibling modules are available to the build context.

To release a new version, bump `.version` first, then run `make container-build` on `main`:

```bash
echo "1.2.3" > .version
git add .version && git commit -m "Release v1.2.3"
git push origin main
make container-build
```

### Services without a Makefile

`regionservice`, `routerservice`, `searchservice`, `tilesservice`, and `swctl` do not have
Makefiles. Build and push manually from the workspace root so all Go modules are in the build
context:

```bash
cd ~/Dev/swayrider-public

SERVICE=regionservice   # change as needed
VERSION=0.1.0

docker buildx build \
  -f $SERVICE/Dockerfile \
  --platform linux/amd64,linux/arm64 \
  -t ghcr.io/swayrider/$SERVICE:v$VERSION \
  -t ghcr.io/swayrider/$SERVICE:latest \
  --push .
```

Replace `SERVICE` and `VERSION` for each service you are releasing.

---

## Image overview

| Service | Image | Compose tag |
|---|---|---|
| authservice | `ghcr.io/swayrider/authservice` | `latest` |
| mailservice | `ghcr.io/swayrider/mailservice` | `latest` |
| regionservice | `ghcr.io/swayrider/regionservice` | `latest` |
| routerservice | `ghcr.io/swayrider/routerservice` | `latest` |
| searchservice | `ghcr.io/swayrider/searchservice` | `latest` |
| tilesservice | `ghcr.io/swayrider/tilesservice` | `latest` |

`swctl` is a CLI tool and is not used in the Compose stack.

---

## Pulling the latest images on the dev server

After pushing new images, pull and restart on the dev server:

```bash
cd ~/Dev/swayrider-public/infra/dev/layer-20
docker compose pull
docker compose up -d
```
