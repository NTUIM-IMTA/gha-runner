# Appendix: Building the runner image

The runner image `ghcr.io/ntuim-imta/gha-runner:go1.27-node26` is **already
built and public** on GHCR, so a fresh cluster install can skip this entirely —
[README](README.md) step 6 pulls it directly. You only need this page when
**creating a new Go/Node combo** or **rebuilding** an existing tag.

Routine bumps (newest Go / Node / pnpm / golangci-lint plus a refreshed action
cache) are scripted in the `runner-image-update` skill under `.agents/skills/`;
this page is the background for what that skill does.

One folder per Go/Node combo lives at the repo root (e.g. `go1.27-node26/`),
each holding a `Dockerfile` driven by `GO_VERSION` / `NODE_VERSION` build args.

> **Note:** `go1.27-node26` also bakes in chromium's system libraries
> (`playwright install-deps`) and qpdf + ghostscript. The consumer e2e jobs
> run `playwright install chromium` **without** `--with-deps` on the
> self-hosted target, so they depend on a runner image built from the current
> Dockerfile — re-push this tag before merging CI changes that drop
> `--with-deps`.

## 1. Log in to GHCR (once per machine)

```bash
GH_USER=eric2969
gh auth login -h github.com -s write:packages,read:packages
gh auth token | docker login ghcr.io -u "$GH_USER" --password-stdin
```

## 2. Build & push

Versions must match the latest patch in
[actions/go-versions](https://raw.githubusercontent.com/actions/go-versions/main/versions-manifest.json)
and
[actions/node-versions](https://raw.githubusercontent.com/actions/node-versions/main/versions-manifest.json):

```bash
docker buildx build go1.27-node26 \
  --platform linux/amd64 \
  --build-arg GO_VERSION=1.27.1 \
  --build-arg NODE_VERSION=26.8.1 \
  -t ghcr.io/ntuim-imta/gha-runner:go1.27-node26 \
  -t ghcr.io/ntuim-imta/gha-runner:go1.27.1-node26.8.1 \
  --push
```

The floating tag (`go1.27-node26`) is what `values.yaml` pins. With
`imagePullPolicy: Always`, re-pushing the same tag goes live on the next job
with no helm change.

## Bumping Go / Node

1. Look up the latest patch in the actions/*-versions manifests linked above.
2. Create a new folder `goX.Y-nodeZ/` and copy/edit the `Dockerfile` ARGs.
3. Build & push with both the floating tag (`goX.Y-nodeZ`) and the immutable
   tag (`goX.Y.Z-nodeA.B.C`) using the command above.
4. Update `template.spec.containers[0].image` in `values.yaml`, then run the
   `helm upgrade` from the README's tuning section.
5. Keep the old folder until no workflow pins it.

# Appendix: Building the Verdaccio S3 image

Verdaccio's storage backend is the in-cluster SeaweedFS S3 store (see
[README](README.md#object-storage-seaweedfs)). The `verdaccio-aws-s3-storage`
plugin is **not** in the stock image, so `verdaccio/Dockerfile` bakes it into a
custom image. Rebuild this only when bumping the Verdaccio or plugin version.

The plugin requires Verdaccio >= 7, which at the time of writing only ships as a
beta — published on Docker Hub under the rolling `7.x-next` tag (no per-beta
tag), so the Dockerfile pins it by digest.

```bash
# Log in to GHCR first (see step 1 above).
docker buildx build verdaccio \
  --platform linux/amd64 \
  --provenance=false \
  -t ghcr.io/ntuim-imta/verdaccio-s3:7.x-next \
  --push
```

The package is public on GHCR, so the cluster pulls it without a secret. The
manifest (`verdaccio/verdaccio.yaml`) pins this tag with `imagePullPolicy: Always`,
so re-pushing the same tag goes live on the next pod restart. To bump: update the pinned `7.x-next@sha256:…` base digest (and/or
plugin version) in `verdaccio/Dockerfile`, then rebuild with the command above.
Look up the current digest with
`docker buildx imagetools inspect verdaccio/verdaccio:7.x-next`.
