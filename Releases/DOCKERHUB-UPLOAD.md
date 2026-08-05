# DOCKERHUB-UPLOAD — Publishing WatchNexus Images to DockerHub & Other Registries

How to publish the three tier images (`standard` / `pro` / `ultra`) to
Docker Hub and alternative registries (GitHub Container Registry, Quay.io,
self-hosted Harbor, AWS/Google/Azure container registries).

---

## 1. Prerequisites

- Docker Engine installed and running (the daemon builds + pushes).
- A registry account + credentials.
- `build/docker-build.sh` from this repo (builds all three tier images
  with tier-excluded controllers baked in at build time).
- Built release artifacts (optional): the Docker path uses the `Dockerfile`
  at the repo root, so **no pre-built artifacts are needed** — it compiles
  backend + frontend inside the image.

---

## 2. Local build (sanity check)

```bash
cd /home/auz/Downloads/git/WatchNexus-Master

# Build all three tiers, do NOT push
./build/docker-build.sh
# or just one tier
./build/docker-build.sh standard
```

Verify:

```bash
docker images 'watchnexus/*'
# watchnexus/watchnexus:1.0.0-standard
# watchnexus/watchnexus:1.0.0-pro
# watchnexus/watchnexus:1.0.0-ultra
# watchnexus/watchnexus:latest-standard|pro|ultra  (plus :latest → ultra)
```

Smoke-test one image locally before publishing:

```bash
docker run --rm -p 8002:8002 watchnexus/watchnexus:1.0.0-standard
curl -i http://localhost:8002/api/cellar/first-launch   # expect 200
```

---

## 3. Docker Hub (default registry)

### 3.1 Log in

```bash
docker login
# Username / password for hub.docker.com (or a PAT scoped to the repo)
```

### 3.2 Push

The script pushes `{DOCKER_REGISTRY}/watchnexus:<VERSION>-<tier>` and
`:latest-<tier>`, and tags ultra as plain `:latest`.

```bash
# Push all three tiers (default registry: watchnexus)
./build/docker-build.sh --push

# Or one tier to a custom namespace
DOCKER_REGISTRY=yourname ./build/docker-build.sh ultra --push
```

### 3.3 Resulting tags

| Image | Description |
|---|---|
| `watchnexus/watchnexus:1.0.0-standard` | Standard, pinned |
| `watchnexus/watchnexus:latest-standard` | Standard, rolling |
| `watchnexus/watchnexus:1.0.0-pro` | Pro, pinned |
| `watchnexus/watchnexus:latest-pro` | Pro, rolling |
| `watchnexus/watchnexus:1.0.0-ultra` | Ultra, pinned |
| `watchnexus/watchnexus:latest-ultra` | Ultra, rolling |
| `watchnexus/watchnexus:latest` | Ultra (default) |

### 3.4 Verify on the hub

```bash
docker pull watchnexus/watchnexus:latest
docker run --rm -p 8002:8002 watchnexus/watchnexus:latest
curl -i http://localhost:8002/api/cellar/first-launch
```

---

## 4. Other registries

All of these use the same `docker-build.sh` with
`DOCKER_REGISTRY=<namespace>`; the script prepends that namespace to every
tag.

### GitHub Container Registry (GHCR)

```bash
echo "$GHCR_TOKEN" | docker login ghcr.io -u WN-Admin --password-stdin
DOCKER_REGISTRY=ghcr.io/wn-admin ./build/docker-build.sh --push
```

### Quay.io

```bash
docker login quay.io
DOCKER_REGISTRY=quay.io/your-org ./build/docker-build.sh --push
```

### Self-hosted Harbor / local registry

```bash
docker login harbor.example.com
DOCKER_REGISTRY=harbor.example.com/watchnexus ./build/docker-build.sh --push

# Or manually:
docker tag watchnexus/watchnexus:1.0.0-ultra harbor.example.com/watchnexus:1.0.0-ultra
docker push harbor.example.com/watchnexus:1.0.0-ultra
```

### Cloud registries (ECR / GCR / ACR)

Authenticate via the provider's CLI, then set `DOCKER_REGISTRY`:

```bash
# AWS ECR
aws ecr get-login-password | docker login --username AWS --password-stdin <account>.dkr.ecr.<region>.amazonaws.com
DOCKER_REGISTRY=<account>.dkr.ecr.<region>.amazonaws.com/watchnexus ./build/docker-build.sh --push

# GCR
gcloud auth configure-docker
DOCKER_REGISTRY=gcr.io/<project>/watchnexus ./build/docker-build.sh --push

# Azure ACR
az acr login --name <registry>
DOCKER_REGISTRY=<registry>.azurecr.io/watchnexus ./build/docker-build.sh --push
```

---

## 5. Image manifest / labels

Each image carries these labels (set by `docker-build.sh`):

- `com.watchnexus.tier` = `standard|pro|ultra`
- `com.watchnexus.version` = `1.0.0`
- `org.opencontainers.image.created` = ISO timestamp

Inspect after push:

```bash
docker inspect watchnexus/watchnexus:1.0.0-ultra \
  --format '{{index .Config.Labels "com.watchnexus.tier"}}'
```

---

## 6. Offline distribution (air-gapped)

If a customer can't pull from the internet, export the image and load it
locally:

```bash
docker save watchnexus/watchnexus:1.0.0-ultra \
  -o watchnexus-ultra-1.0.0-docker.tar
# on the target:
docker load -i watchnexus-ultra-1.0.0-docker.tar
docker run -p 8002:8002 watchnexus/watchnexus:1.0.0-ultra
```

The `docker save` tarball is also produced by `build-installers.fish --docker`
into `release/<tier>/docker/`.

---

## 7. Release checklist

- [ ] `docker login` succeeded for each target registry.
- [ ] `./build/docker-build.sh` built all tiers locally with no errors.
- [ ] One image smoke-tested (`first-launch` returns 200 on port 8002).
- [ ] `./build/docker-build.sh --push` pushed every tier + `latest-*` tags.
- [ ] Ultra tagged as plain `latest` (script does this automatically).
- [ ] GHCR / other registries listed in README of each image repo.
- [ ] SHA256 digests recorded for the release notes
      (`docker inspect <image> --format '{{index .RepoDigests 0}}'`).
