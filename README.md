# gha-runner

Self-hosted GitHub Actions runners for NTUIM-IMTA, running on k3s via
[Actions Runner Controller](https://github.com/actions/actions-runner-controller).

## Repository layout

```
.
├── go1.26-node26/   # custom runner image (one folder per Go/Node combo)
│   └── Dockerfile
├── athens/          # in-cluster Go module proxy manifest
│   └── athens.yaml
├── seaweedfs/       # S3 object store backing Athens + Verdaccio (atomic writes)
│   ├── seaweedfs.yaml
│   └── cache-gc-cronjob.yaml  # watermark-based eviction for both cache buckets
├── verdaccio/       # in-cluster npm / pnpm registry mirror
│   ├── verdaccio.yaml
│   └── Dockerfile   # custom image: verdaccio + aws-s3-storage plugin
├── dockerhub-mirror/  # Docker Hub pull-through cache (CI service containers)
│   └── dockerhub-mirror.yaml
├── playwright-cache/  # Playwright browser CDN cache (CI e2e jobs)
│   └── playwright-cache.yaml
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

All `kubectl apply -f …` commands below run from a clone of this repo:

```bash
git clone https://github.com/NTUIM-IMTA/gha-runner.git ~/gha-runner
cd ~/gha-runner
```

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
kubectl create namespace arc-runners --dry-run=client -o yaml | kubectl apply -f -
kubectl -n arc-runners create secret generic gh-config \
  --from-literal=github_token=$GH_TOKEN \
  --dry-run=client -o yaml | kubectl apply -f -
```

### 5. Apply the in-cluster mirrors and registry

The custom `verdaccio-s3` image on GHCR is private, so the verdaccio namespace
needs a pull secret **before** its deployment can start (a PAT with
`read:packages`; skip this block if the package is made public like the runner
image):

```bash
GH_USER=eric2969          # GitHub account that can read the package
GHCR_TOKEN=ghp_xxx        # PAT with read:packages, never commit the real value
kubectl create namespace verdaccio --dry-run=client -o yaml | kubectl apply -f -
kubectl -n verdaccio create secret docker-registry ghcr-pull \
  --docker-server=ghcr.io --docker-username=$GH_USER \
  --docker-password=$GHCR_TOKEN \
  --dry-run=client -o yaml | kubectl apply -f -
```

Then the mirrors and the registry — SeaweedFS first, since Athens and Verdaccio
crash-loop until the S3 endpoint is up:

```bash
kubectl apply -f seaweedfs/seaweedfs.yaml
kubectl apply -f seaweedfs/cache-gc-cronjob.yaml
kubectl -n seaweedfs rollout status deploy/seaweedfs
kubectl apply -f athens/athens.yaml
kubectl apply -f verdaccio/verdaccio.yaml
kubectl apply -f dockerhub-mirror/dockerhub-mirror.yaml
kubectl apply -f playwright-cache/playwright-cache.yaml
kubectl apply -f registry/registry.yaml
kubectl apply -f registry/gc-cronjob.yaml
kubectl -n athens rollout status deploy/athens
kubectl -n verdaccio rollout status deploy/verdaccio
kubectl -n dockerhub-mirror rollout status deploy/dockerhub-mirror
kubectl -n playwright-cache rollout status deploy/playwright-cache
kubectl -n registry rollout status deploy/registry
```

SeaweedFS pre-creates the `gomods` and `verdaccio` buckets on startup (Athens
also self-creates `gomods` as a backstop); verify with:

```bash
kubectl -n seaweedfs exec deploy/seaweedfs -- sh -c "echo s3.bucket.list | weed shell"
```

See [Object storage: SeaweedFS](#object-storage-seaweedfs) for why both proxies
use object storage rather than local disk, and the note that **Verdaccio needs
the custom S3 image built first** ([BUILD.md](BUILD.md)).

The registry deploy above is enough for the cluster itself; for **consumers** to
push/pull over `:5000` it also needs a one-time node-level config (DNS + a
containerd HTTP mirror). See [本地 image registry](#本地-image-registry) for that
node setup, the daily GC CronJob, and how each app's build wires into it.

### 6. Install the runner scale set

`values.yaml` carries the runner image, the explicit dind sidecar (with the
Docker Hub mirror wiring), `maxRunners`, `GOPROXY`, `NPM_CONFIG_REGISTRY`,
and `PLAYWRIGHT_DOWNLOAD_HOST` — the chart picks up the rest from the
controller install. The image it pins is prebuilt and public on GHCR; rebuilding it is a
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
| 4 | ~8 | Safe default |
| 8 | ~16 | Current setting — watch node headroom |
| 16+ | risky | Only after scaling the node |

Watch `kubectl top node` while runs queue; bump down on memory pressure.

### dind sidecar (explicit template, was `containerMode: dind`)

`values.yaml` no longer uses `containerMode: dind` — the chart's auto-injected
dind sidecar accepts no extra dockerd flags, and we pass `--registry-mirror`
(see [Docker Hub mirror](#docker-hub-mirror--playwright-cdn-cache)). Instead,
the pod template is an explicit verbatim copy of what the chart would render
for containerMode dind on 0.14.2, plus the `[delta]`-marked additions.

Removing the dind block entirely disables `docker/setup-buildx-action`, any
`docker build`, and every `services:` block in consumer workflows ("cannot
connect to docker daemon").

> **Migration gotcha:** a release installed with `containerMode: dind` keeps
> that value in its stored release values, and `--reuse-values` merges it with
> this file — the chart then injects a second dind ("Duplicate value:
> dind-sock/dind"). The switch to the explicit template needs ONE upgrade
> without `--reuse-values` (pass `--set githubConfigUrl=… --set
> githubConfigSecret=gh-config` alongside `-f values.yaml`); afterwards
> `--reuse-values` works again.

**When bumping the chart version**: re-render the auto-injected shape and
re-diff the template block in `values.yaml`:

```bash
helm template my-runners \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --version <new-version> -n arc-runners \
  --set githubConfigUrl=https://github.com/NTUIM-IMTA \
  --set githubConfigSecret=gh-config \
  --set controllerServiceAccount.namespace=arc-systems \
  --set controllerServiceAccount.name=arc-gha-rs-controller \
  --set containerMode.type=dind
```

### Runner image

```yaml
template:
  spec:
    containers:
      - name: runner
        image: ghcr.io/ntuim-imta/gha-runner:go1.26-node26
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

## Docker Hub mirror + Playwright CDN cache

The consumer CI's **integration and e2e jobs** pull more than Go modules and
npm packages, and runner pods are ephemeral — without a local cache these
downloads repeat on every single run:

| What | Cached by | Wired up via |
|---|---|---|
| `services:` images (postgres, redis, …) | `dockerhub-mirror/` — registry:2 pull-through cache | dind `--registry-mirror` flag in `values.yaml` |
| `playwright install chromium` (~170 MB) | `playwright-cache/` — nginx `proxy_cache` in front of `cdn.playwright.dev` | `PLAYWRIGHT_DOWNLOAD_HOST` env in `values.yaml` |
| chromium system libs / qpdf + ghostscript | baked into the runner image ([BUILD.md](BUILD.md)) | e2e jobs pass `--with-deps` only on the `gha` target |

Notes:

- The mirror only covers `docker.io` pulls; `ghcr.io` etc. go direct. If the
  mirror is down, dockerd falls back to Docker Hub transparently.
- Mirrored content self-expires (`REGISTRY_PROXY_TTL`, 168h); the nginx cache
  evicts by LRU (`max_size=8g`, `inactive=90d`). Neither bucket needs the
  SeaweedFS cache-gc CronJob.
- `PLAYWRIGHT_DOWNLOAD_HOST` has **no fallback chain** — if the cache pod is
  down, e2e browser installs fail until it is back
  (`kubectl -n playwright-cache rollout status deploy/playwright-cache`).
- Verify a cache hit: re-run any e2e job and check the nginx logs
  (`kubectl -n playwright-cache logs deploy/playwright-cache`) or the
  `X-Cache-Status: HIT` response header.

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

**Cache eviction.** Neither Athens nor Verdaccio ever deletes anything, and
`local-path` does not enforce the PVC size, so both buckets would grow until the
node disk fills. `seaweedfs/cache-gc-cronjob.yaml` runs daily and copies
kubelet's image-GC design: per-bucket high/low watermarks (default 10 GiB / 8
GiB). A bucket under the high watermark is never touched — unlike a TTL, hot
entries are not expired and re-downloaded for no reason. Above it, the oldest
objects are evicted until usage is back under the low watermark, and a call to
the master's `/vol/vacuum` HTTP endpoint compacts the freed space. Evicted entries are simply cache
misses: Athens treats a version as cached only if all three of its objects
exist (`go.mod`, `<v>.info`, `source.zip`), so partial eviction self-heals, and
Verdaccio re-fetches from npmjs (its package index `verdaccio-s3-db.json` is
excluded from eviction).

The S3 access key/secret is one logical credential duplicated across four
Secrets (`seaweedfs-s3-config` defines the server-side identity; `athens`,
`verdaccio`, and the cache-gc CronJob in `seaweedfs` each hold a matching
client copy) and must be kept in sync.
SeaweedFS is `ClusterIP`-only; the trust boundary is the PVE firewall, same as
the image registry.

> **Caveats / migration.**
> - **Verdaccio needs Verdaccio >= 7**, which currently only ships as a beta
>   (Docker tag `7.x-next`, pinned by digest). Build/push the custom image
>   before applying ([BUILD.md](BUILD.md)); a critical mirror runs on a beta here.
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
kubectl delete -f dockerhub-mirror/dockerhub-mirror.yaml
kubectl delete -f playwright-cache/playwright-cache.yaml
kubectl delete -f seaweedfs/seaweedfs.yaml
kubectl delete -f registry/gc-cronjob.yaml
kubectl delete -f registry/registry.yaml
```
