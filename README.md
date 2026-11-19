# shared-services

Utility packages and small supporting microservices for deploying services and core infrastructure.

Multi-package repository: each package lives in its own directory under `packages/` and is published as a separate GHCR package:

```
ghcr.io/dev1-alexwei-net/shared-services/<package>
```

## Layout

```
packages/
  <package-name>/     # one self-contained directory per package
```

Each package contains its own `Dockerfile` (or equivalent build), a short `README.md`, and any tests.

## CI / Publish

A single workflow (`.github/workflows/build-packages.yml`) builds and publishes **every** package in the repo to GHCR. It detects any `packages/*/Dockerfile`, builds `amd64` + `arm64`, pushes per-arch tags, and creates a multi-arch `latest` manifest. Adding a new package requires **no** workflow changes — just create `packages/<name>/Dockerfile` and push to `main`.

## Packages

| Package | Description | Image |
| ------- | ----------- | ----- |
| [`sendloki`](./packages/sendloki/README.md) | One-shot boot event → Grafana Loki ECS sidecar (Bash · curl · jq) | `ghcr.io/dev1-alexwei-net/shared-services/sendloki` |
