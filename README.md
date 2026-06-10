# gha-runner

Self-hosted GitHub Actions runners for NTUIM-IMTA, running on k3s via
[Actions Runner Controller](https://github.com/actions/actions-runner-controller).

## Repository layout

```
.
├── go1.26-node24/   # custom runner image (one folder per Go/Node combo)
│   └── Dockerfile
├── athens/          # in-cluster Go module proxy manifest
│   └── athens.yaml
├── seaweedfs/       # S3 object store backing Athens + Verdaccio (atomic writes)
│   └── seaweedfs.yaml
├── verdaccio/       # in-cluster npm / pnpm registry mirror
│   ├── verdaccio.yaml
│   └── Dockerfile   # custom image: verdaccio + aws-s3-storage plugin
├── registry/        # in-cluster image registry + daily GC cronjob
│   ├── registry.yaml
│   └── gc-cronjob.yaml
├── values.yaml      # gha-runner-scale-set helm overrides
├── README.md
└── BUILD.md         # appendix: build/push the runner + verdaccio-s3 images
```

> **Note (2026):** the MinIO community edition was archived upstream. The
> object store backing Athens and Verdaccio is [SeaweedFS](#object-storage-seaweedfs).

## Set up from a blank OS

Target: Ubuntu 24.04 LTS (or Debian-equivalent), x86_64, single node.

### 1. k3s

```bash
curl -sfL https://get.k3s.io | sh - && sudo chmod 644 /etc/rancher/k3s/k3s.yaml
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc
kubectl get nodes
```

### 2. helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
rm get_helm.sh
helm version
```

### 3. ARC controller

```bash
helm install arc \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
  --version 0.14.2 \
  -n arc-systems --create-namespace
```

### 4. GitHub config secret

Create a Personal Access Token (`ghp_…`) with `repo` + `admin:org`
scopes (ARC's documented auth; a GitHub App with equivalent permissions also
works). Then:

```bash
GH_TOKEN=github_pat_xxx   # PAT, never commit the real value
kubectl create namespace arc-runners
kubectl -n arc-runners delete secret gh-config
kubectl -n arc-runners create secret generic gh-config \
  --from-literal=github_token=$GH_TOKEN
```

### 5. Apply the in-cluster mirrors and registry

```bash
# SeaweedFS first — Athens and Verdaccio crash-loop until the S3 endpoint is up.
kubectl apply -f seaweedfs/seaweedfs.yaml
kubectl -n seaweedfs rollout status deploy/seaweedfs
kubectl apply -f athens/athens.yaml
kubectl apply -f verdaccio/verdaccio.yaml
kubectl apply -f registry/registry.yaml
kubectl apply -f registry/gc-cronjob.yaml
kubectl get ns seaweedfs athens verdaccio registry
kubectl -n athens rollout status deploy/athens
kubectl -n verdaccio rollout status deploy/verdaccio
kubectl -n registry rollout status deploy/registry
```

SeaweedFS pre-creates the `gomods` and `verdaccio` buckets on startup (Athens
also self-creates `gomods` as a backstop). See
[Object storage: SeaweedFS](#object-storage-seaweedfs) for why both proxies use
object storage rather than local disk, and the note that **Verdaccio needs the
custom S3 image built first** ([BUILD.md](BUILD.md)).

The registry deploy above is enough for the cluster itself; for **consumers** to
push/pull over `:5000` it also needs a one-time node-level config (DNS + a
containerd HTTP mirror). See [本地 image registry](#本地-image-registry) for that
node setup, the daily GC CronJob, and how each app's build wires into it.

### 6. Install the runner scale set

`values.yaml` carries the runner image, dind, `maxRunners`, `GOPROXY`,
and `NPM_CONFIG_REGISTRY` — the chart picks up the rest from the controller
install. The image it pins is prebuilt and public on GHCR; rebuilding it is a
one-time task — see [Appendix: Building the runner image](BUILD.md).

Runners register into the org's **`Default`** runner group (no `runnerGroup`
override). A named custom group also works on this cluster, but Default keeps
the setup simple and needs no group to be pre-created in GitHub.

```bash
helm install my-runners \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --version 0.14.2 \
  -n arc-runners \
  --set githubConfigUrl=https://github.com/NTUIM-IMTA \
  --set githubConfigSecret=gh-config \
  -f values.yaml
```

### 7. Verify

```bash
kubectl -n arc-systems get pods
kubectl -n arc-runners get autoscalingrunnerset my-runners
```

In any consumer repo's workflow, set `runs-on: my-runners`.

## Workflow conventions

```yaml
jobs:
  test:
    runs-on: my-runners
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: "1.26"   # matches a folder name in this repo
          cache: false         # GOPROXY routes through Athens instead
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: false         # NPM_CONFIG_REGISTRY routes through Verdaccio
      - run: cd backend && go test -race -cover ./...
      - run: cd frontend && pnpm install --frozen-lockfile
```

## Tuning knobs

All knobs live in `values.yaml`. After editing:

```bash
helm upgrade my-runners \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --version 0.14.2 \
  -n arc-runners \
  --reuse-values \
  -f values.yaml
```

### Concurrency cap (`maxRunners`)

```yaml
maxRunners: 4
```

| Value | Peak pods | Notes |
|---|---|---|
| 2 | ~4 | Diagnosing OOM |
| 4 | ~8 | Current setting (default) |
| 8 | ~16 | Only after checking node headroom |
| 16+ | risky | Only after scaling the node |

Watch `kubectl top node` while runs queue; bump down on memory pressure.

### dind sidecar (`containerMode`)

```yaml
containerMode:
  type: dind
```

Disables: `docker/setup-buildx-action` and any `docker build` in the workflow
will fail with "cannot connect to docker daemon".

### Runner image

```yaml
template:
  spec:
    containers:
      - name: runner
        image: ghcr.io/ntuim-imta/gha-runner:go1.26-node24
        imagePullPolicy: Always
```

`Always` re-pulls on every job, so a re-`--push` with the same tag (see
[BUILD.md](BUILD.md)) goes live on the next run with no helm change.

### GOPROXY

```yaml
env:
  - name: GOPROXY
    value: http://athens.athens.svc.cluster.local:3000,https://proxy.golang.org,direct
```

Fallback chain — if Athens is unreachable, jobs still resolve modules upstream.

### NPM_CONFIG_REGISTRY

```yaml
env:
  - name: NPM_CONFIG_REGISTRY
    value: http://verdaccio.verdaccio.svc.cluster.local:4873/
```

Honoured by npm, pnpm, and yarn. Verdaccio proxies missing packages to
`registry.npmjs.org` and caches them on its PVC.

## Object storage: SeaweedFS

Both proxies keep their cache in the in-cluster SeaweedFS S3 store
(`seaweedfs/seaweedfs.yaml`) — Athens in the `gomods` bucket, Verdaccio in the
`verdaccio` bucket — not on local disk.

**Why.** Athens' `disk` backend and Verdaccio's local-fs both write some files
straight to their final path with no atomic rename, and neither integrity-checks
on read. So if the pod is killed mid-download (e.g. several builds hammer it at
once and OOM it), it leaves a **0-byte or truncated** file at the real path and
serves that poison cache forever:

- **Athens** → Go fails with a fatal `checksum mismatch`, which — unlike a
  network error — does **not** fall back to the next `GOPROXY`. Every build of
  that module breaks until the bad file is deleted by hand.
- **Verdaccio** → a 0-byte `package.json` makes `JSON.parse("")` throw, Verdaccio
  returns HTTP 500, and pnpm (which checks publish age before fetching) can't get
  the metadata so it fails closed with `ERR_PNPM_MINIMUM_RELEASE_AGE_VIOLATION`.

**The fix is structural, not a cleanup job.** An S3 object only becomes readable
once its write fully completes (`PutObject`, or multipart `CompleteUpload` /
server-side compose), so a crash leaves the key **absent**, never half-written —
and the proxy simply re-fetches from upstream on the next request. The
poison-cache class of bug is eliminated by construction; no 0-byte/truncation
sweeper is needed.

**Why SeaweedFS and not MinIO.** The MinIO community edition was archived
upstream in 2026 (read-only, no more releases). SeaweedFS is an actively
maintained, Apache-2.0, S3-compatible store. Athens talks to it with its generic
S3 client (its storage type is confusingly named `minio`, but the endpoint is
SeaweedFS); Verdaccio talks to it via the `verdaccio-aws-s3-storage` plugin
(AWS SDK v3) baked into a custom image (`verdaccio/Dockerfile`).

The S3 access key/secret is one logical credential duplicated across three
namespaces (the `seaweedfs-s3-config` Secret defines the server-side identity;
`athens` and `verdaccio` each hold a matching Secret) and must be kept in sync.
SeaweedFS is `ClusterIP`-only; the trust boundary is the PVE firewall, same as
the image registry.

> **Caveats / migration.**
> - **Verdaccio needs Verdaccio >= 7**, which currently only ships as a beta —
>   the custom image pins `7.0.0-beta.4`. Build/push it before applying
>   ([BUILD.md](BUILD.md)); a critical mirror runs on a beta here.
> - **Verify Athens' `source.zip` path** after cutover: it uses minio-go's
>   `ComposeObject` (server-side multipart copy), the classic S3-compat edge
>   case. Smoke-test with a large module (e.g. force a fresh `pdfcpu` fetch) and
>   confirm the build passes.
> - Switching starts with empty buckets; the first builds repopulate from
>   upstream. The old `athens-storage` PVC and any live-deployed
>   `athens-cache-janitor` CronJob (a disk-era stopgap that only swept 0-byte
>   files) are obsolete:
>
>   ```bash
>   kubectl -n athens delete cronjob athens-cache-janitor --ignore-not-found
>   kubectl -n athens delete pvc athens-storage --ignore-not-found
>   ```

## 本地 image registry

叢集內建一個 image registry，讓各服務的 image 留在本地、不必經外網推送／拉取到 GHCR。
以 **insecure HTTP** 對外服務於 `registry.imta.im.ntu.edu.tw:5000`，對外存取由
**Proxmox（PVE）層的防火牆**控管（node 與 k3s 本身不另設防火牆）。

manifest：

- `registry/registry.yaml` — Namespace、PVC（20Gi，`local-path`）、`registry:2`
  Deployment（已開啟 `REGISTRY_STORAGE_DELETE_ENABLED`）、`LoadBalancer:5000`
  （k3s ServiceLB 綁 node IP）。
- `registry/gc-cronjob.yaml` — 每日 04:00 的 GC CronJob：每個 repo 保留最新 5 個
  `sha-*` tag，刪除其餘後回收 blob 空間。

### node 一次性設定

1. 將 DNS `registry.imta.im.ntu.edu.tw` 指向 node 的 IP。
2. 讓 containerd 以 HTTP 拉取此 registry —— 編輯 `/etc/rancher/k3s/registries.yaml`：

```yaml
mirrors:
  "registry.imta.im.ntu.edu.tw:5000":
    endpoint:
      - "http://registry.imta.im.ntu.edu.tw:5000"
```

   containerd 預設對 registry 走 HTTPS；若不指定，部署拉取會出現
   `http: server gave HTTP response to HTTPS client`。
3. `sudo systemctl restart k3s` 套用。
4. 對外的 5000 埠請在 PVE 防火牆限制到信任來源 —— 此 registry 無 TLS、無認證，
   防火牆是唯一防線。

### 部署

```bash
kubectl apply -f registry/registry.yaml
kubectl apply -f registry/gc-cronjob.yaml
```

驗證：`curl http://registry.imta.im.ntu.edu.tw:5000/v2/` 應回 `200`。

### 與各服務 build 的整合

各 app 的 `build-image.yml` 會依觸發方式選擇 image 推送目的地，並同步改寫 ArgoCD
helm chart 的 image 來源：

- 由 push 觸發（CI 後 `workflow_run`）→ 一律推到**本地 registry**。
- 手動 `workflow_dispatch` → 可選 `local` 或 `gha`（ghcr.io），**預設 `local`**。

GHCR 的舊版本清理仍由各 repo 既有的 `Delete-Old-Packages.yaml` 負責；本地
registry 的清理則由上述 GC CronJob 處理。

## Update / uninstall

```bash
# Upgrade chart version
helm upgrade my-runners \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --version <new-version> \
  -n arc-runners --reuse-values -f values.yaml

# Tear down
helm uninstall my-runners -n arc-runners
helm uninstall arc -n arc-systems
kubectl delete -f athens/athens.yaml
kubectl delete -f verdaccio/verdaccio.yaml
kubectl delete -f seaweedfs/seaweedfs.yaml
kubectl delete -f registry/gc-cronjob.yaml
kubectl delete -f registry/registry.yaml
```
