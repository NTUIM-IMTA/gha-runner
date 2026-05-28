# gha-runner

Custom self-hosted runner images for [Actions Runner Controller (ARC)](https://github.com/actions/actions-runner-controller)
`gha-runner-scale-set`, plus an in-cluster
[GitHub Actions cache server](https://github.com/falcondev-oss/github-actions-cache-server),
used by NTUIM-IMTA.

The stock `ghcr.io/actions/actions-runner` is a bare Ubuntu without `gcc`,
`docker`, `yq`, or an `/opt/hostedtoolcache`. Workflows that worked on
`ubuntu-latest` break in several ways when pointed at it:

- `go test -race` fails (cgo needs a C toolchain)
- `docker/setup-buildx-action` fails (no docker daemon, no CLI)
- `setup-go` / `setup-node` redownload the SDK every job (measured 6 min for
  `setup-go` alone)
- `actions/cache` restores from GitHub's hosted cache at ~3 MB/s

This repo addresses all four:

| Piece | Fix |
|---|---|
| `build-essential`, `git`, `jq`, `unzip` | apt-installed in the runner image |
| `docker` CLI + buildx-plugin | apt-installed; the daemon lives in a dind sidecar (`containerMode: dind`) |
| `yq` (Mike Farah, Go) | dropped into `/usr/local/bin/yq` |
| Pre-seeded toolcache for setup-go / setup-node | `/opt/hostedtoolcache/{go,node}/<ver>/x64/` + `RUNNER_TOOL_CACHE` env |
| Fast `actions/cache` restore | `cache-server/` deploys a falcondev-oss cache server; runners route to it via `ACTIONS_RESULTS_URL` |

## Repository layout

```
.
├── go1.26-node24/                 # one folder per (Go major.minor, Node major) combo
│   └── Dockerfile
├── cache-server/
│   └── cache-server.yaml          # falcondev-oss in-cluster cache server (Deployment + PVC + Service)
├── values.yaml                    # ARC AutoscalingRunnerSet override
└── README.md
```

Add a new folder per Go/Node combination when the workflows pin a new version.

## Current cluster state

| Thing | Value |
|---|---|
| Cluster | k3s, single node `gh-runner`, default StorageClass `local-path` |
| ARC release | `my-runners` in namespace `arc-runners`, chart `gha-runner-scale-set` v0.14.2 |
| Runner group | `gh-runner` (in `githubConfigUrl: https://github.com/NTUIM-IMTA`) |
| Cache server | Deployment `actions-cache-server` in namespace `actions-cache`, 20Gi PVC |
| Workflow `runs-on:` label | `my-runners` |

## Build & push the runner image

The cluster is x86_64. We build single-arch for `linux/amd64`.

```bash
cd go1.26-node24

# Versions MUST be the exact patch setup-go / setup-node resolve to from
# https://raw.githubusercontent.com/actions/go-versions/main/versions-manifest.json
# https://raw.githubusercontent.com/actions/node-versions/main/versions-manifest.json
docker buildx build \
  --platform linux/amd64 \
  --build-arg GO_VERSION=1.26.3 \
  --build-arg NODE_VERSION=24.16.0 \
  -t ghcr.io/ntuim-imta/gha-runner:go1.26-node24 \
  -t ghcr.io/ntuim-imta/gha-runner:go1.26.3-node24.16.0 \
  --push .
```

On a fresh machine, log in to ghcr first with a token that has `write:packages`:

```bash
gh auth refresh -h github.com -s write:packages,read:packages
gh auth token | docker login ghcr.io -u <gh-user> --password-stdin
```

`imagePullPolicy: Always` on the runner pod means the next job picks up the new
digest automatically — no helm change needed when only the image tag content
changes.

## Deploy the cache server

One-time setup (idempotent on re-apply):

```bash
kubectl apply -f cache-server/cache-server.yaml
```

This creates namespace `actions-cache`, a 20Gi `local-path` PVC, a Deployment
running `ghcr.io/falcondev-oss/github-actions-cache-server`, and a ClusterIP
Service at `http://actions-cache-server.actions-cache.svc.cluster.local:3000`.

Runners reach it via `ACTIONS_RESULTS_URL` (already wired in `values.yaml`).

## Deploy / upgrade the runners

```bash
helm upgrade my-runners \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --version 0.14.2 \
  -n arc-runners \
  --reuse-values \
  -f values.yaml
```

`--reuse-values` preserves the chart-time settings that aren't in `values.yaml`
(`githubConfigUrl`, `githubConfigSecret`, `runnerGroup`). What `values.yaml`
adds on top:

- `maxRunners: 4` — cap concurrent runner pods (each one also spawns a dind
  sidecar, so total container budget is ~8)
- `containerMode.type: dind` — chart auto-injects the `dind` sidecar,
  `/var/run` shared volume, `DOCKER_HOST` env, etc.
- `template.spec.containers[0].image` — our custom image
- `template.spec.containers[0].env[ACTIONS_RESULTS_URL]` — points
  `actions/cache` (and the implicit `setup-go` / `setup-node` cache) at the
  in-cluster server

In-flight runner pods finish their current job and exit; ARC spawns new ones
from the new spec.

## Use it from a workflow

```yaml
jobs:
  test:
    runs-on: my-runners        # selector → AutoscalingRunnerSet name
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.26"   # must match a `<ver>` folder in this repo
      - run: cd backend && go test -race -cover ./...
```

Nothing repo-specific — every workflow under `NTUIM-IMTA` just sets
`runs-on: my-runners`.

## Bumping Go / Node

1. Look up the new resolved patch at the actions/*-versions manifests linked
   above.
2. Create a new folder `goX.Y-nodeZ/`, copy and tweak the `Dockerfile` ARGs.
3. Build & push with tags `goX.Y-nodeZ` (floating) + `goX.Y.Z-nodeA.B.C`
   (immutable, for rollback).
4. Update `image:` in `values.yaml` and re-run the `helm upgrade`.
5. Keep the old folder until no workflow pins it.
