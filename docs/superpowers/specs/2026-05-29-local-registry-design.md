# Local registry with branch-selectable local/GHCR image flow

- **Date:** 2026-05-29
- **Status:** Draft (pending review)
- **Scope:** `gha-runner` (registry + GC + docs), the four app repos' `build-image.yml`
  (`admission`, `intranet`, `theses`, `print`), and verification against
  `argoCD-helm-chart`.

## Context

Today every app builds its image on the self-hosted runners and pushes to
`ghcr.io/ntuim-imta/<repo>-<branch>:sha-<short>`. The `build-image.yml` workflow
then clones `NTUIM-IMTA/argoCD-helm-chart` and repins `.main.image.tag` in
`environments/{prod,dev}/<repo>.yaml`. ArgoCD reconciles and the node's
containerd pulls the image from GHCR using the `ghcr-login-token` pull secret.

This couples every deploy to an outbound WAN path to GHCR. The campus network is
restricted and the round-trip is slow.

## Goals

- Stand up an in-cluster OCI registry, exposed externally over **insecure HTTP**
  at `registry.imta.im.ntu.edu.tw:5000` (firewall is the only access control,
  managed by the operator out of band).
- Make image build/push **target-selectable**:
  - `push`-driven builds (via the `workflow_run` after CI) always target **local**.
  - `workflow_dispatch` lets the operator pick `local` or `gha`, **defaulting local**.
- The workflow auto-selects the push destination and rewrites the helm chart
  image source (`repository`, `tag`, `imagePullSecrets`) to match the target.
- Bound registry growth with a daily in-cluster GC that keeps the most recent
  **5** `sha-*` tags per repository.

## Non-goals (YAGNI)

- TLS / authentication on the registry — insecure HTTP + firewall only.
- Read-only-during-GC hardening — GC runs at an off-peak hour and accepts the
  small concurrent-write risk on a single-node, low-build-rate cluster.
- Changing the existing GHCR retention path. `Delete-Old-Packages.yaml` stays
  **GHCR-only**; local retention is handled solely by the new in-cluster CronJob.
- A registry web UI.

## Architecture

### 1. Registry deployment (new, in `gha-runner` repo)

A new `registry/registry.yaml` (sibling of `athens/`, `verdaccio/`):

- Namespace `registry`.
- `registry:2` Deployment with env:
  - `REGISTRY_STORAGE_DELETE_ENABLED=true` (required for tag deletion + GC).
- A PVC (k3s default `local-path`, **20Gi**) mounted at `/var/lib/registry`.
- `Service` `type: LoadBalancer`, port `5000` → k3s ServiceLB binds the node IP
  on `:5000`. DNS `registry.imta.im.ntu.edu.tw` → node IP (operator-managed).

### 2. Node one-time config (manual, documented — NOT GitOps)

`/etc/rancher/k3s/registries.yaml` so containerd pulls over HTTP:

```yaml
mirrors:
  "registry.imta.im.ntu.edu.tw:5000":
    endpoint:
      - "http://registry.imta.im.ntu.edu.tw:5000"
```

Then `systemctl restart k3s`. Documented in `README.md` / `BUILD.md`, plus the
DNS requirement and the firewall note for port 5000.

### 3. buildkit push side (`build-image.yml` → `setup-buildx-action`)

Keep the existing `driver-opts: network=host` (already added for Athens) and add
an insecure-registry config so buildkit pushes over HTTP:

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3
  with:
    driver-opts: network=host
    buildkitd-config-inline: |
      [registry."registry.imta.im.ntu.edu.tw:5000"]
        http = true
```

The DNS name resolves inside the runner pod via CoreDNS → upstream → node IP, so
no `hostAliases` are needed.

### 4. Build target selection (`build-image.yml`)

- Add a `workflow_dispatch` input:
  ```yaml
  workflow_dispatch:
    inputs:
      target:
        description: "Image registry target"
        type: choice
        options: [local, gha]
        default: local
  ```
- Resolve `TARGET`: if `github.event_name == 'workflow_dispatch'` use
  `inputs.target`; otherwise (`workflow_run`) force `local`.
- Derive per target:

  | target | `REGISTRY`                          | `repository`                  | pull secret        | `docker/login-action` |
  |--------|-------------------------------------|-------------------------------|--------------------|-----------------------|
  | local  | `registry.imta.im.ntu.edu.tw:5000`  | `ntuim-imta/<repo>-<branch>`  | *(empty)*          | skipped               |
  | gha    | `ghcr.io`                           | `ntuim-imta/<repo>-<branch>`  | `ghcr-login-token` | run                   |

- `docker/login-action` runs only when `TARGET == 'gha'` (`if:` guard).
- `docker/metadata-action` `images:` and the build tags use the resolved
  `REGISTRY`/`repository`.

### 5. Helm source switch (`build-image.yml` "Update Helm chart image tag" step)

In addition to the current `.main.image.tag`, the step rewrites the source for
the resolved env file (`environments/{prod,dev}/<repo>.yaml`):

```sh
yq -i ".main.image.repository = \"${REGISTRY}/ntuim-imta/${REPO}-${HEAD_BRANCH}\"" "$ENV_FILE"
yq -i ".main.image.tag = \"${IMAGE_TAG}\"" "$ENV_FILE"
yq -i ".main.image.imagePullSecrets = \"${PULL_SECRET}\"" "$ENV_FILE"   # "" for local
```

The app charts already guard the secret block:

```gotmpl
{{- if .Values.main.image.imagePullSecrets }}
imagePullSecrets:
  - name: {{ .Values.main.image.imagePullSecrets }}
{{- end }}
```

so an empty string emits no `imagePullSecrets`. Confirmed for `charts/theses`,
`charts/admission`, `charts/print`. **To verify:** `charts/intranet` (new-style)
follows the same guarded pattern.

### 6. Garbage collection (new, in `gha-runner` repo)

A `CronJob` in the `registry` namespace, daily at 04:00 (`0 4 * * *`), using the
`registry:2` image and mounting the same PVC. Two phases in one script:

1. **Tag retention (online, via the live HTTP API):** for each repo from
   `GET /v2/_catalog`, list `sha-*` tags via `GET /v2/<repo>/tags/list`, fetch
   each tag's config-blob `created` timestamp, sort descending, keep the newest
   **5** plus `latest`, and `DELETE /v2/<repo>/manifests/<digest>` the rest
   (digest from the `Docker-Content-Digest` header of a manifest `HEAD`).
2. **Blob sweep (filesystem):** `registry garbage-collect -m
   --delete-untagged=true /etc/docker/registry/config.yml` on the mounted PVC to
   reclaim space from now-unreferenced blobs.

Single-node RWO `local-path` PVC can be mounted by both the registry Deployment
pod and the CronJob pod on the same node.

## Data flow

**Automated (push → local):**
push to `develop`/`main` → CI → `workflow_run` triggers `build-image`
(`target=local`) → buildkit builds and pushes to
`registry.imta.im.ntu.edu.tw:5000/ntuim-imta/<repo>-<branch>:sha-<short>` →
helm-update sets `repository` to the local registry, `tag` to the new sha, and
clears `imagePullSecrets` in `environments/{dev|prod}/<repo>.yaml` → ArgoCD
reconciles → containerd pulls over HTTP per `registries.yaml`.

**Manual (dispatch → gha):**
`workflow_dispatch` with `target=gha` → `docker login ghcr.io` → push
`ghcr.io/ntuim-imta/<repo>-<branch>:sha-<short>` → helm-update points
`repository` back to GHCR and sets `imagePullSecrets: ghcr-login-token`.

Branch→env mapping is unchanged and orthogonal to target: `main`→`prod`,
otherwise `dev`.

## Security

The registry is externally reachable, insecure HTTP, no auth, with delete
enabled. Anyone able to reach `:5000` can pull, push, and delete any image. The
**firewall is the only control** and is the operator's responsibility (stated
requirement). This is an accepted risk for this deployment.

## Affected repos / files

- `gha-runner`: new `registry/registry.yaml` (Deployment, PVC, Service, CronJob);
  `README.md` / `BUILD.md` updates (node `registries.yaml`, DNS, firewall).
- `admission`, `intranet`, `theses`, `print`: `.github/workflows/build-image.yml`
  (target input + selection, conditional login, buildkit insecure config,
  helm-update repository/secret rewrite).
- `argoCD-helm-chart`: no template change required; values already carry
  `repository`/`tag`/`imagePullSecrets`. Verify `charts/intranet` guard.

## Assumptions to confirm during implementation

1. `charts/intranet` guards `imagePullSecrets` with `{{- if }}` like the others.
2. DNS `registry.imta.im.ntu.edu.tw` resolves to the node from both the pod
   network (CoreDNS upstream) and the host.
3. k3s ServiceLB can bind host port `5000` (no conflict; Traefik holds 80/443).
4. `local-path` PVC is mountable by two pods on the single node concurrently.

## Verification plan

- Deploy registry; from a runner pod, push a throwaway image and confirm it lands
  (`GET /v2/_catalog`).
- Trigger a `local` build on `theses` (develop); confirm the image appears in the
  registry, the helm values flip to the local repository with empty pull secret,
  and ArgoCD deploys with containerd pulling over HTTP.
- Trigger a `gha` dispatch; confirm it pushes to GHCR and flips the helm values
  back to GHCR + `ghcr-login-token`.
- Run the GC CronJob manually; confirm tags beyond the newest 5 are deleted and
  blob storage shrinks.
