# packages

One directory per package. Each package is self-contained (own Dockerfile or build definition, README, tests) and is published as `ghcr.io/dev1-alexwei-net/shared-services/<package>`.

## Available packages

- [`sendloki`](./sendloki/README.md) — one-shot boot event publisher to Grafana Loki, designed as an ECS sidecar container.

## Adding a new package

1. Create `packages/<name>/` with a `Dockerfile`.
2. Add a `README.md` documenting the package (env vars, usage, image ref).
3. Push to `main` — the CI workflow picks up the new package automatically (no workflow edit required).
