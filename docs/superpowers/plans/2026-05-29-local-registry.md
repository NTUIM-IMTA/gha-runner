# Local registry with branch-selectable local/GHCR image flow — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up an in-cluster insecure registry at `registry.imta.im.ntu.edu.tw:5000` and make each app's image build target-selectable (push → local, dispatch → local|gha, default local), rewriting the ArgoCD helm source to match, with a daily GC keeping the newest 5 `sha-*` tags per repo.

**Architecture:** Phase 1 deploys `registry:2` (PVC-backed, ServiceLB:5000, delete enabled) plus a daily GC CronJob into the `registry` namespace of the single-node k3s cluster, and documents the one-time node `registries.yaml` + DNS + firewall setup. Phase 2 edits each app repo's `build-image.yml` to resolve a registry target, push there, configure buildkit for insecure HTTP, and repin the helm chart's `repository`/`tag`/`imagePullSecrets`.

**Tech Stack:** k3s, ServiceLB, Docker distribution `registry:2`, `regclient/regctl` (retention), GitHub Actions (`docker/setup-buildx-action`, `docker/build-push-action`), `yq`, ArgoCD helm charts.

**Spec:** `docs/superpowers/specs/2026-05-29-local-registry-design.md`

**Reference values (do not parameterize unless noted):**
- Registry DNS/endpoint: `registry.imta.im.ntu.edu.tw:5000` (insecure HTTP)
- In-cluster service address (for jobs/buildkit inside the cluster): `registry.registry.svc.cluster.local:5000`
- Retention: keep newest **5** `sha-*` tags per repo + `latest`; GC daily at `0 4 * * *`
- App repos: `admission`, `intranet`, `theses`, `print`; branch→env: `main`→`prod`, else→`dev`

---

## File structure

- `gha-runner/registry/registry.yaml` — Namespace, PVC, Deployment, Service (one file; these change together and are small).
- `gha-runner/registry/gc-cronjob.yaml` — ConfigMap (retention script) + CronJob.
- `gha-runner/README.md` — node `registries.yaml`, DNS, firewall, deploy steps (appended section).
- `<app>/.github/workflows/build-image.yml` — target resolution, conditional login, buildkit insecure config, helm source rewrite (identical edit across the four repos).

---

## Phase 1 — Registry + GC (gha-runner repo)

> All `kubectl` steps run on a host with cluster access (kubeconfig). The author's
> laptop has no kubeconfig, so these are operator-run verifications.

### Task 1: Registry Deployment, PVC, Service

**Files:**
- Create: `gha-runner/registry/registry.yaml`

- [ ] **Step 1: Write the manifest**

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: registry
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: registry-data
  namespace: registry
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: local-path
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: registry
  namespace: registry
spec:
  replicas: 1
  strategy:
    type: Recreate          # single RWO PVC: avoid two pods racing on rollout
  selector:
    matchLabels:
      app: registry
  template:
    metadata:
      labels:
        app: registry
    spec:
      containers:
        - name: registry
          image: registry:2
          ports:
            - containerPort: 5000
          env:
            - name: REGISTRY_HTTP_ADDR
              value: ":5000"
            - name: REGISTRY_STORAGE_DELETE_ENABLED
              value: "true"
          volumeMounts:
            - name: data
              mountPath: /var/lib/registry
          readinessProbe:
            httpGet:
              path: /v2/
              port: 5000
            initialDelaySeconds: 3
            periodSeconds: 10
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: registry-data
---
apiVersion: v1
kind: Service
metadata:
  name: registry
  namespace: registry
spec:
  type: LoadBalancer
  selector:
    app: registry
  ports:
    - name: http
      port: 5000
      targetPort: 5000
```

- [ ] **Step 2: Validate manifest locally (client-side)**

Run: `kubectl apply --dry-run=client -f gha-runner/registry/registry.yaml`
Expected: each object prints `... (dry run)` with no schema errors.

- [ ] **Step 3: Deploy (operator host)**

Run: `kubectl apply -f gha-runner/registry/registry.yaml`
Then: `kubectl -n registry rollout status deploy/registry`
Expected: `deployment "registry" successfully rolled out`.

- [ ] **Step 4: Verify ServiceLB bound port 5000 and registry answers**

Run: `kubectl -n registry get svc registry -o wide`
Expected: `TYPE=LoadBalancer`, `EXTERNAL-IP=<node IP>`, `PORT(S)=5000:...`.
Run: `curl -s http://registry.imta.im.ntu.edu.tw:5000/v2/ -o /dev/null -w '%{http_code}\n'`
Expected: `200`.

- [ ] **Step 5: Commit**

```bash
git add gha-runner/registry/registry.yaml
git commit -m "feat(registry): deploy in-cluster registry:2 with PVC and ServiceLB:5000"
```

---

### Task 2: GC CronJob (tag retention + blob sweep)

**Files:**
- Create: `gha-runner/registry/gc-cronjob.yaml`

The retention initContainer uses the in-cluster service address (no external DNS
needed for the job). The GC container mounts the same PVC and sweeps blobs.

- [ ] **Step 1: Write the ConfigMap + CronJob**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: registry-gc-script
  namespace: registry
data:
  retention.sh: |
    #!/bin/sh
    set -eu
    REG=registry.registry.svc.cluster.local:5000
    KEEP=5
    regctl registry set --tls disabled "$REG"
    for repo in $(regctl repo ls "$REG"); do
      tags=$(regctl tag ls "$REG/$repo" | grep '^sha-' || true)
      [ -z "$tags" ] && continue
      for tag in $tags; do
        created=$(regctl image config "$REG/$repo:$tag" --format '{{.Created}}' 2>/dev/null || echo "")
        printf '%s\t%s\n' "$created" "$tag"
      done | sort -r | awk -v k="$KEEP" 'NR>k {print $2}' | while read -r old; do
        echo "deleting $repo:$old"
        regctl tag rm "$REG/$repo:$old" || true
      done
    done
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: registry-gc
  namespace: registry
spec:
  schedule: "0 4 * * *"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 1
      template:
        spec:
          restartPolicy: Never
          initContainers:
            - name: retention
              image: ghcr.io/regclient/regctl:v0.8.0-alpine
              command: ["/bin/sh", "/scripts/retention.sh"]
              volumeMounts:
                - name: script
                  mountPath: /scripts
          containers:
            - name: gc
              image: registry:2
              command:
                - /bin/registry
                - garbage-collect
                - -m
                - --delete-untagged=true
                - /etc/docker/registry/config.yml
              volumeMounts:
                - name: data
                  mountPath: /var/lib/registry
          volumes:
            - name: script
              configMap:
                name: registry-gc-script
            - name: data
              persistentVolumeClaim:
                claimName: registry-data
```

- [ ] **Step 2: Validate manifest locally**

Run: `kubectl apply --dry-run=client -f gha-runner/registry/gc-cronjob.yaml`
Expected: ConfigMap and CronJob print `(dry run)` with no errors.

- [ ] **Step 3: Deploy (operator host)**

Run: `kubectl apply -f gha-runner/registry/gc-cronjob.yaml`
Run: `kubectl -n registry get cronjob registry-gc`
Expected: row present with `SCHEDULE 0 4 * * *`.

- [ ] **Step 4: Trigger a manual run and inspect logs**

Run: `kubectl -n registry create job --from=cronjob/registry-gc gc-manual-1`
Run: `kubectl -n registry wait --for=condition=complete job/gc-manual-1 --timeout=300s`
Run: `kubectl -n registry logs job/gc-manual-1 -c retention; kubectl -n registry logs job/gc-manual-1 -c gc`
Expected: retention lists repos (empty on a fresh registry is fine); gc prints a
"blobs marked, N blobs and X manifests eligible for deletion" summary and exits 0.

- [ ] **Step 5: Commit**

```bash
git add gha-runner/registry/gc-cronjob.yaml
git commit -m "feat(registry): daily GC cronjob keeping newest 5 sha tags per repo"
```

---

### Task 3: Node + operator docs

**Files:**
- Modify: `gha-runner/README.md` (append a "Local image registry" section)

- [ ] **Step 1: Append the docs section**

Add to `README.md`:

````markdown
## Local image registry

An in-cluster registry serves images over **insecure HTTP** at
`registry.imta.im.ntu.edu.tw:5000`. Access is gated only by the host firewall —
keep port 5000 closed to untrusted networks.

### One-time node setup

1. Point DNS `registry.imta.im.ntu.edu.tw` at the node's IP.
2. Tell containerd to pull this registry over HTTP — `/etc/rancher/k3s/registries.yaml`:

   ```yaml
   mirrors:
     "registry.imta.im.ntu.edu.tw:5000":
       endpoint:
         - "http://registry.imta.im.ntu.edu.tw:5000"
   ```

3. `sudo systemctl restart k3s`
4. Open the firewall for port 5000 only to trusted sources.

### Deploy

```bash
kubectl apply -f registry/registry.yaml
kubectl apply -f registry/gc-cronjob.yaml
```

Retention: a daily 04:00 CronJob keeps the newest 5 `sha-*` tags per repo and
sweeps unreferenced blobs. GHCR retention is unchanged (each app's
`Delete-Old-Packages.yaml`).
````

- [ ] **Step 2: Verify the file renders (no broken fences)**

Run: `sed -n '/## Local image registry/,$p' gha-runner/README.md | head -40`
Expected: the section prints with intact code fences.

- [ ] **Step 3: Commit**

```bash
git add gha-runner/README.md
git commit -m "docs: document local registry node setup and deploy"
```

---

## Phase 2 — Build target selection (app repos)

> Implement on `theses` first (Task 4–7), verify end-to-end, then replicate the
> identical edit to `admission`, `intranet`, `print` (Task 8).

### Task 4: Add `target` input and resolve registry target (theses)

**Files:**
- Modify: `theses/.github/workflows/build-image.yml`

- [ ] **Step 1: Add the `workflow_dispatch` input**

Replace the trigger block:

```yaml
on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]
    branches: [main, develop]
  workflow_dispatch:
```

with:

```yaml
on:
  workflow_run:
    workflows: ["CI"]
    types: [completed]
    branches: [main, develop]
  workflow_dispatch:
    inputs:
      target:
        description: "Image registry target"
        type: choice
        options: [local, gha]
        default: local
```

- [ ] **Step 2: Remove the static `REGISTRY` env**

Delete these two lines near the top of the file:

```yaml
env:
  REGISTRY: ghcr.io
```

(The registry is now resolved per-run in the next step. If `env:` becomes empty, remove the `env:` key entirely.)

- [ ] **Step 3: Extend the "Resolve build context" step to emit target outputs**

In the `Resolve build context` step (`id: ctx`), append to the `run:` script,
before its closing, these lines (the step already writes `$GITHUB_OUTPUT`):

```bash
          TARGET="${{ github.event_name == 'workflow_dispatch' && inputs.target || 'local' }}"
          if [ "$TARGET" = "gha" ]; then
            REGISTRY="ghcr.io"
            PULL_SECRET="ghcr-login-token"
          else
            REGISTRY="registry.imta.im.ntu.edu.tw:5000"
            PULL_SECRET=""
          fi
          echo "target=${TARGET}"           >> $GITHUB_OUTPUT
          echo "registry=${REGISTRY}"       >> $GITHUB_OUTPUT
          echo "pull_secret=${PULL_SECRET}" >> $GITHUB_OUTPUT
```

- [ ] **Step 4: Verify YAML parses and outputs are wired**

Run: `python3 -c "import yaml,sys; yaml.safe_load(open('theses/.github/workflows/build-image.yml'))" && echo OK`
Expected: `OK`.
Run: `grep -nE "target=|registry=|pull_secret=" theses/.github/workflows/build-image.yml`
Expected: the three `>> $GITHUB_OUTPUT` lines present.

- [ ] **Step 5: Commit**

```bash
git -C theses add .github/workflows/build-image.yml
git -C theses commit -m "ci: add registry target input (push->local, dispatch->local|gha)"
```

---

### Task 5: Conditional login, buildkit insecure config, target-aware tags (theses)

**Files:**
- Modify: `theses/.github/workflows/build-image.yml`

- [ ] **Step 1: Add buildkit insecure-registry config to setup-buildx**

Replace the existing buildx step:

```yaml
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        with:
          driver-opts: network=host
```

with:

```yaml
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
        with:
          driver-opts: network=host
          buildkitd-config-inline: |
            [registry."registry.imta.im.ntu.edu.tw:5000"]
              http = true
```

- [ ] **Step 2: Make the GHCR login conditional**

Replace the login step:

```yaml
      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```

with:

```yaml
      - name: Log in to GitHub Container Registry
        if: steps.ctx.outputs.target == 'gha'
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```

- [ ] **Step 3: Point metadata-action at the resolved registry**

In the `Extract metadata (tags, labels)` step, replace:

```yaml
          images: ${{ env.REGISTRY }}/${{ steps.ctx.outputs.image_name }}
```

with:

```yaml
          images: ${{ steps.ctx.outputs.registry }}/${{ steps.ctx.outputs.image_name }}
```

- [ ] **Step 4: Verify YAML parses and no stale `env.REGISTRY` remains**

Run: `python3 -c "import yaml; yaml.safe_load(open('theses/.github/workflows/build-image.yml'))" && echo OK`
Expected: `OK`.
Run: `grep -n "env.REGISTRY" theses/.github/workflows/build-image.yml || echo "no stale refs"`
Expected: `no stale refs`.

- [ ] **Step 5: Commit**

```bash
git -C theses add .github/workflows/build-image.yml
git -C theses commit -m "ci: conditional ghcr login + buildkit insecure config + target-aware image"
```

---

### Task 6: Rewrite helm source per target (theses)

**Files:**
- Modify: `theses/.github/workflows/build-image.yml` (the "Update Helm chart image tag" step)

- [ ] **Step 1: Add repository + imagePullSecrets rewrites alongside the tag**

In the `Update Helm chart image tag` step, the env block currently exposes
`IMAGE_TAG` and `HEAD_BRANCH`. Add `REGISTRY`, `IMAGE_NAME`, `PULL_SECRET`:

```yaml
        env:
          HELM_CHART_PAT: ${{ secrets.HELM_CHART_PAT }}
          IMAGE_TAG: sha-${{ steps.ctx.outputs.short_sha }}
          HEAD_BRANCH: ${{ steps.ctx.outputs.branch }}
          REGISTRY: ${{ steps.ctx.outputs.registry }}
          IMAGE_NAME: ${{ steps.ctx.outputs.image_name }}
          PULL_SECRET: ${{ steps.ctx.outputs.pull_secret }}
```

Then, in the retry loop, immediately after the existing
`yq -i ".main.image.tag = ..."` line, add:

```bash
            yq -i ".main.image.repository = \"${REGISTRY}/${IMAGE_NAME}\"" "environments/${ENV_DIR}/theses.yaml"
            yq -i ".main.image.imagePullSecrets = \"${PULL_SECRET}\"" "environments/${ENV_DIR}/theses.yaml"
```

(`image_name` already equals `ntuim-imta/theses-<branch>`, so
`${REGISTRY}/${IMAGE_NAME}` yields e.g.
`registry.imta.im.ntu.edu.tw:5000/ntuim-imta/theses-develop` or
`ghcr.io/ntuim-imta/theses-develop`.)

- [ ] **Step 2: Verify YAML parses and the three yq lines are present**

Run: `python3 -c "import yaml; yaml.safe_load(open('theses/.github/workflows/build-image.yml'))" && echo OK`
Expected: `OK`.
Run: `grep -nE "image.repository|image.tag|image.imagePullSecrets" theses/.github/workflows/build-image.yml`
Expected: three `yq -i` lines.

- [ ] **Step 3: Commit**

```bash
git -C theses add .github/workflows/build-image.yml
git -C theses commit -m "ci: rewrite helm image repository + pull secret per target"
```

---

### Task 7: End-to-end verify theses (local + gha)

> Requires Phase 1 deployed and the node setup done.

- [ ] **Step 1: Push and let the automated (local) path run**

```bash
git -C theses push   # on develop
```
Run: `gh -R NTUIM-IMTA/theses run list --workflow "Build and Push Docker Image" --limit 3`
Expected: a `workflow_run`-triggered run starts after CI succeeds.

- [ ] **Step 2: Confirm the image landed in the local registry**

Run (operator host): `curl -s http://registry.imta.im.ntu.edu.tw:5000/v2/ntuim-imta/theses-develop/tags/list`
Expected: JSON containing the new `sha-<short>` tag.

- [ ] **Step 3: Confirm the helm values flipped to local**

Run: `gh api repos/NTUIM-IMTA/argoCD-helm-chart/contents/environments/dev/theses.yaml --jq '.content' | base64 -d | yq '.main.image'`
Expected: `repository: registry.imta.im.ntu.edu.tw:5000/ntuim-imta/theses-develop`,
new `tag`, and `imagePullSecrets: ""`.

- [ ] **Step 4: Confirm the deployment pulls over HTTP**

Run: `kubectl -n <theses-dev-namespace> get pods` then `kubectl describe` the pod.
Expected: image pulled successfully from the local registry (no ImagePullBackOff,
no pull-secret needed).

- [ ] **Step 5: Verify the gha path via dispatch**

Run: `gh -R NTUIM-IMTA/theses workflow run "Build and Push Docker Image" -f target=gha`
After it completes, re-run Step 3's command.
Expected: `repository: ghcr.io/ntuim-imta/theses-develop` and
`imagePullSecrets: ghcr-login-token`.

- [ ] **Step 6: (no commit — verification only)**

---

### Task 8: Replicate to admission, intranet, print

The build-image.yml regions edited in Tasks 4–6 are byte-identical across the four
repos (verified: same `setup-buildx`, `context/file/push`, login, metadata, and
helm-update structure), except each repo references its own
`environments/${ENV_DIR}/<repo>.yaml` path. Apply the **same edits** to each repo,
substituting the repo name in the two `yq` paths.

**Files:**
- Modify: `admission/.github/workflows/build-image.yml`
- Modify: `intranet/.github/workflows/build-image.yml`
- Modify: `print/.github/workflows/build-image.yml`

- [ ] **Step 1: Apply Task 4 edits to each repo**

For each of `admission`, `intranet`, `print`: apply Task 4 Steps 1–3 verbatim
(the `target` input, removing `env.REGISTRY`, and the target-resolution `run:` lines).

- [ ] **Step 2: Apply Task 5 edits to each repo**

Apply Task 5 Steps 1–3 verbatim (buildkit insecure config, conditional login,
metadata `images:` → `steps.ctx.outputs.registry`).

- [ ] **Step 3: Apply Task 6 edits to each repo**

Apply Task 6 Step 1, replacing `theses.yaml` with `<repo>.yaml` in the two new
`yq` lines (e.g. `environments/${ENV_DIR}/admission.yaml`).

- [ ] **Step 4: Verify all three parse and have no stale refs**

```bash
for d in admission intranet print; do
  python3 -c "import yaml; yaml.safe_load(open('$d/.github/workflows/build-image.yml'))" && echo "$d OK"
  grep -n "env.REGISTRY" "$d/.github/workflows/build-image.yml" && echo "$d STALE" || true
done
```
Expected: `admission OK`, `intranet OK`, `print OK`; no `STALE` lines.

- [ ] **Step 5: Confirm `charts/intranet` guards imagePullSecrets**

Run: `grep -n "imagePullSecrets" /Users/eric/imta/argoCD-helm-chart/charts/intranet/templates/deployment.yaml`
Expected: a `{{- if .Values.main.image.imagePullSecrets }}` guard. If absent, add
the guard to `charts/intranet` matching `charts/theses` and commit it to
`argoCD-helm-chart`.

- [ ] **Step 6: Commit each repo**

```bash
for d in admission intranet print; do
  git -C /Users/eric/imta/$d add .github/workflows/build-image.yml
  git -C /Users/eric/imta/$d commit -m "ci: branch-selectable local/gha registry target"
done
```

- [ ] **Step 7: Push and spot-check one repo end-to-end**

Push each repo's `develop`. For `admission`, repeat Task 7 Steps 2–4 (local path),
substituting `admission` for `theses`.
Expected: image in local registry, helm values flipped to local, deployment healthy.

---

## Self-review notes

- **Spec coverage:** registry deploy (T1), GC keep-5 + delete-enabled (T2), node
  config/DNS/firewall docs (T3), target input default-local (T4), buildkit insecure
  + conditional login (T5), helm repository/tag/imagePullSecrets switch (T6),
  end-to-end local+gha verify (T7), four-repo rollout + intranet guard check (T8),
  GHCR retention untouched (no task — correct, out of scope). All spec sections map.
- **Assumptions surfaced in spec** are checked in tasks: ServiceLB:5000 (T1 S4),
  PVC mount (T2 S4 via job sharing PVC), DNS resolution (T7 S2), intranet guard
  (T8 S5).
- **Open risk:** `regctl image config --format '{{.Created}}'` field name — if the
  executor finds the template key differs in regctl v0.8.0, use
  `regctl manifest get --format '{{.GetConfig}}'` / `regctl image inspect` and read
  the `.created` field; keep the sort-desc/keep-5 logic identical.
