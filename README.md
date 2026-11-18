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

## Packages

| Package | Description |
| ------- | ----------- |
| _(none yet)_ | First package story: see issue on this repo |
