---
id: build-custom-docker-images
title: Build Custom Docker Images
weight: 70
---

This guide explains how to build the Medplum server and app images from this repository. It covers two different destinations:

- **Local Docker image store:** build a single-platform image with Buildx `--load`, then run it with Docker Compose without publishing it.
- **Your own container registry:** use the repository scripts, which build multi-platform images and push them to the configured registry.

The [full-stack Docker setup](/docs/self-hosting/running-full-medplum-stack-in-docker) remains the quickest way to run the published images without building from source.

## Prerequisites

- A clone of the Medplum repository, checked out at the revision you want to build.
- Node.js and npm versions supported by the repository's root `package.json`.
- Docker Engine with Buildx enabled.
- Enough disk space for the dependency installation, build outputs, and image layers.

Run the commands below from the repository root unless a command says otherwise.

## Build the application artifacts

Install the dependencies and build the server and app packages:

```bash
npm ci
npm run build
```

If the app should point to a non-default API URL, configure `packages/app/.env` before running the build. The app's Vite environment variables are baked into its static output at build time; see [Install from scratch](/docs/self-hosting/install-from-scratch#configure-the-app-environment) for the available settings.

## Build images for local use

The repository's `build-docker-server.sh` and `build-docker-app.sh` scripts are publishing scripts: they invoke Buildx with `--push`. For a local-only build, create the same input archives and use `docker buildx build --load` instead.

Buildx `--load` loads one platform into the local Docker image store. Choose the platform for the machine where you will run the image. For example, use `linux/arm64` on Apple Silicon and `linux/amd64` on an x86-64 host. To load both architectures locally, build each platform separately with a different tag.

### Medplum server image

From the repository root, create the metadata and runtime archives used by the root `Dockerfile`:

```bash
tar \
  --no-xattrs \
  -czf medplum-server-metadata.tar.gz \
  package.json \
  package-lock.json \
  packages/bot-layer/package.json \
  packages/ccda/package.json \
  packages/core/package.json \
  packages/definitions/package.json \
  packages/fhir-router/package.json \
  packages/server/package.json

tar \
  --no-xattrs \
  --exclude='*.ts' \
  --exclude='*.tsbuildinfo' \
  -czf medplum-server-runtime.tar.gz \
  LICENSE.txt \
  NOTICE \
  packages/ccda/dist \
  packages/core/dist \
  packages/definitions/dist \
  packages/fhir-router/dist \
  packages/server/dist
```

Build and load the image. Replace the platform with the one you need:

```bash
docker buildx build \
  --platform linux/arm64 \
  --load \
  --tag medplum/medplum-server:local \
  .
```

### Medplum app image

Create the app archive expected by `packages/app/Dockerfile`, then build from that directory:

```bash
tar \
  --no-xattrs \
  -czf ./packages/app/medplum-app.tar.gz \
  -C packages/app/dist .

docker buildx build \
  --platform linux/arm64 \
  --load \
  --tag medplum/medplum-app:local \
  packages/app
```

Use the same platform for both images when running them together. Confirm that both tags are present locally:

```bash
docker image inspect \
  medplum/medplum-server:local \
  medplum/medplum-app:local
```

### Run the local images with Docker Compose

Create a Compose override next to the repository's `docker-compose.full-stack.yml`:

```yaml
services:
  medplum-server:
    image: medplum/medplum-server:local
  medplum-app:
    image: medplum/medplum-app:local
```

Save it as `docker-compose.local-images.yml`, then start the stack:

```bash
docker compose \
  -f docker-compose.full-stack.yml \
  -f docker-compose.local-images.yml \
  up -d
```

The override changes only the server and app image references; the database, cache, ports, environment, volumes, and health checks remain from the full-stack Compose file.

## Publish images to your own registry

For a multi-platform build that should be pushed to a registry, use the repository scripts. Authenticate to your registry first with `docker login`, then set the destination image names and the commit SHA:

```bash
export SERVER_DOCKERHUB_REPOSITORY=registry.example.com/team/medplum-server
export APP_DOCKERHUB_REPOSITORY=registry.example.com/team/medplum-app
export GITHUB_SHA="$(git rev-parse HEAD)"

./scripts/build-docker-server.sh
./scripts/build-docker-app.sh
```

Despite the variable names, the values are image references and can include your registry hostname and namespace. The server script requires `SERVER_DOCKERHUB_REPOSITORY`. The app script skips the app image if `APP_DOCKERHUB_REPOSITORY` is unset.

Both scripts use Buildx for `linux/amd64` and `linux/arm64` and push the resulting image. They tag images with `GITHUB_SHA`; passing `--latest` adds a `latest` tag, and passing `--release` also adds version tags derived from the root `package.json`. These scripts are intended for registry publishing, not local-only builds.

## Related implementation

The commands in this guide follow the current repository build inputs:

- `scripts/build-docker-server.sh` creates the server metadata and runtime archives, then builds and pushes the root `Dockerfile`.
- `scripts/build-docker-app.sh` creates `packages/app/medplum-app.tar.gz`, then builds and pushes `packages/app/Dockerfile`.
- The root `Dockerfile` consumes the two server archives; `packages/app/Dockerfile` consumes the app archive.
