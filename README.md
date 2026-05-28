# gha-runner

Custom self-hosted runner images for [Actions Runner Controller (ARC)](https://github.com/actions/actions-runner-controller)
`gha-runner-scale-set`, used by NTUIM-IMTA.

The stock `ghcr.io/actions/actions-runner` is a bare Ubuntu without `gcc` and
without an `/opt/hostedtoolcache`. That makes `go test -race` fail (cgo needs a
C toolchain) and forces `actions/setup-go` / `actions/setup-node` to redownload
the SDK on every job (we measured 6 min for `setup-go` alone).

These images fix both:

- install `build-essential` so cgo works
- pre-seed `/opt/hostedtoolcache/{go,node}/<ver>/<arch>` so the setup actions hit
  the in-cache path

## Repository layout

```
.
├── go1.26-node24/      # one folder per (Go major.minor, Node major) combo
│   └── Dockerfile
├── values.yaml         # ARC AutoscalingRunnerSet override (image only)
└── README.md
```

Add a new folder per combination when the workflows pin a new version.

## Build & push

The runner cluster is amd64 — single-arch build is enough.

```bash
cd go1.26-node24

# pick the latest patch published on go.dev/dl and nodejs.org/dist
docker buildx build \
  --platform linux/amd64 \
  --build-arg GO_VERSION=1.26.0 \
  --build-arg NODE_VERSION=24.10.0 \
  -t ghcr.io/ntuim-imta/gha-runner:go1.26-node24 \
  -t ghcr.io/ntuim-imta/gha-runner:go1.26.0-node24.10.0 \
  --push .
```

`docker login ghcr.io -u <gh-user>` with a PAT that has `write:packages` first if
this is a fresh machine.

## Deploy to the cluster

The release is `my-runners` in namespace `arc-runners`, chart
`gha-runner-scale-set` v0.14.2.

```bash
helm upgrade my-runners \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --version 0.14.2 \
  -n arc-runners \
  --reuse-values \
  -f values.yaml
```

`--reuse-values` preserves the existing `githubConfigUrl`, `githubConfigSecret`,
and `runnerGroup` settings. Only the runner pod image is overridden.

After the upgrade, in-flight runner pods finish their current job and exit;
ARC starts new ones from the new image.

## Bumping Go / Node

1. Create a new folder, e.g. `go1.27-node24/`, copy and tweak the `Dockerfile`.
2. Build & push with a tag that reflects the new combo.
3. Update `image:` in `values.yaml` and re-run the `helm upgrade`.
4. Keep the old folder around until no workflow pins the old version.
