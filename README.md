# gha-runner

Custom self-hosted runner images for [Actions Runner Controller (ARC)](https://github.com/actions/actions-runner-controller)
`gha-runner-scale-set`, used by NTUIM-IMTA.

The stock `ghcr.io/actions/actions-runner` is a bare Ubuntu without `gcc`,
`docker`, `yq`, or an `/opt/hostedtoolcache`. Workflows that worked on
`ubuntu-latest` break in several ways when pointed at it:

- `go test -race` fails (cgo needs a C toolchain)
- `docker/setup-buildx-action` fails (no docker daemon, no CLI)
- `setup-go` / `setup-node` redownload the SDK every job (measured 6 min for
  `setup-go` alone)

This repo addresses all three:

| Piece | Fix |
|---|---|
| `build-essential`, `git`, `jq`, `unzip` | apt-installed in the runner image |
| `docker` CLI + buildx-plugin | apt-installed; the daemon lives in a dind sidecar (`containerMode: dind`) |
| `yq` (Mike Farah, Go) | dropped into `/usr/local/bin/yq` |
| Pre-seeded toolcache for setup-go / setup-node | `/opt/hostedtoolcache/{go,node}/<ver>/x64/` + `RUNNER_TOOL_CACHE` env |

`actions/cache` still goes to GitHub's hosted cache (~3 MB/s for big restores).
A pod-level `ACTIONS_RESULTS_URL` / `ACTIONS_CACHE_URL` override does **not**
work on runner ≥ 2.323 — `Runner.Worker` overwrites those vars from the job
message before any action sees them, so an in-cluster cache server is bypassed.
Speeding cache restore up needs a network-level redirect (sidecar / proxy)
handled outside this repo.

## Repository layout

```
.
├── go1.26-node24/                 # one folder per (Go major.minor, Node major) combo
│   └── Dockerfile
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

- `maxRunners: 8` — cap concurrent runner pods (each one also spawns a dind
  sidecar, so total container budget is ~2x this)
- `containerMode.type: dind` — chart auto-injects the `dind` sidecar,
  `/var/run` shared volume, `DOCKER_HOST` env, etc.
- `proxy:` — runner / dind / initContainer egress goes through the campus
  Squid forward proxy at `140.112.106.22:3128`
- `template.spec.containers[0].image` — our custom image

In-flight runner pods finish their current job and exit; ARC spawns new ones
from the new spec.

## Tuning knobs

All knobs live in `values.yaml`; after every edit run the `helm upgrade`
command above.

### Concurrency (`maxRunners`)

Each in-flight job costs two containers on the node (runner + dind sidecar)
plus the toolcache image overhead. Rough budget on the current single-node
k3s (`gh-runner`):

| `maxRunners` | Peak pods | When to use |
|---|---|---|
| 2 | ~4 | Diagnosing OOM / heavy app build jobs |
| 4 | ~8 | Default for everyday CI |
| 8 | ~16 | Multiple repos pushing in parallel — current setting |
| 16+ | risky | Only if node was scaled up |

Watch `kubectl top node gh-runner` while several runs are queued; bump down
if memory pressure shows.

### Egress proxy

The whole `proxy:` block in `values.yaml`. To bypass Squid temporarily, comment
the block out and `helm upgrade`. To add a host that should never go through
Squid, append it to `proxy.noProxy`. Examples already in place:

- `10.0.0.0/8`, `.svc`, `.cluster.local`, `localhost` — cluster-internal
- `140.112.0.0/16` — campus subnet
- `ghcr.io`, `github.com`, `api.github.com` — direct upstream so image pulls
  and the runner registration API skip Squid

The chart injects `HTTP_PROXY` / `HTTPS_PROXY` / `NO_PROXY` (both upper- and
lowercase forms) into runner + dind + initContainer + listener. Confirm
inside a running pod with:

```bash
kubectl -n arc-runners exec <runner-pod> -c runner -- env | grep -i proxy
```

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
          cache: false         # use in-cluster Athens instead — see below
      - run: cd backend && go test -race -cover ./...
```

Nothing repo-specific — every workflow under `NTUIM-IMTA` just sets
`runs-on: my-runners`.

### Why `cache: false`

The runner pod has `GOPROXY` pointing at the in-cluster Athens proxy
(`http://athens.athens.svc.cluster.local:3000`). `go mod download` therefore
fetches modules from inside the cluster (Gbps) on every job — usually a few
seconds for our codebases.

The default `cache: true` would instead make `setup-go` use `actions/cache`,
which restores a GOMODCACHE tarball from GitHub's hosted cache server. That
crosses the public internet and was measured at ~25-30s per job for a 150 MB
tarball. Skipping it and going straight to Athens is faster and removes one
moving part.

`GOPROXY` is set with a fallback chain:

```
http://athens.../, https://proxy.golang.org, direct
```

so if Athens is down or hasn't seen a module before, the job still finishes
by hitting upstream directly.

## Bumping Go / Node

1. Look up the new resolved patch at the actions/*-versions manifests linked
   above.
2. Create a new folder `goX.Y-nodeZ/`, copy and tweak the `Dockerfile` ARGs.
3. Build & push with tags `goX.Y-nodeZ` (floating) + `goX.Y.Z-nodeA.B.C`
   (immutable, for rollback).
4. Update `image:` in `values.yaml` and re-run the `helm upgrade`.
5. Keep the old folder until no workflow pins it.
