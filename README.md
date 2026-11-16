# gha-runner

Self-hosted GitHub Actions runners for NTUIM-IMTA, running on k3s via
[Actions Runner Controller](https://github.com/actions/actions-runner-controller).

## Repository layout

```
.
├── .agents/skills/  # agent skills (.claude/skills and .codex/skills symlink here)
│   └── runner-image-update/   # bump Go/Node/pnpm + refresh the action cache
├── go1.27-node26/   # custom runner image (one folder per Go/Node combo)
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

**New standalone host (with or without a local registry):**

```bash
curl -sfL https://get.k3s.io | sh - && sudo chmod 644 /etc/rancher/k3s/k3s.yaml
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc
kubectl get nodes
```

**Additional worker in an existing cluster:** use the
[K3s agent installation](https://docs.k3s.io/quick-start) instead of the block
above. Obtain the token on the existing server:

```bash
sudo cat /var/lib/rancher/k3s/server/node-token
k3s --version
```

On the new worker, use that token, server IP, and K3s version:

```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_VERSION='<server-k3s-version>' \
  K3S_URL='https://<server-IP>:6443' \
  K3S_TOKEN='<node-token>' \
  sh -
```

Skip steps 2–6 when ARC and the runner scale set already exist in that cluster;
use the existing cluster's kubeconfig for step 7. Runners share the cluster's
cache services and `values.yaml`.

Joining as an agent does not reserve the node for runners. The current registry
Deployment has no `nodeSelector`; use node labels and scheduling rules to keep
registry/cache workloads off the worker. Restrict K3s ServiceLB placement too if
port 5000 should only be exposed on the registry host.

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
  --version 0.15.1 \
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

Both custom images (`gha-runner`, `verdaccio-s3`) are **public** on GHCR, so no
pull secret or `docker login` is needed on the cluster.

Apply the cache services — SeaweedFS first, since Athens and Verdaccio
crash-loop until the S3 endpoint is up. A standalone worker can keep these
caches without hosting the image registry. To omit all caches too, skip this
step and use the values changes in step 6:

```bash
kubectl apply -f seaweedfs/seaweedfs.yaml
kubectl apply -f seaweedfs/cache-gc-cronjob.yaml
kubectl -n seaweedfs rollout status deploy/seaweedfs
kubectl apply -f athens/athens.yaml
kubectl apply -f verdaccio/verdaccio.yaml
kubectl apply -f dockerhub-mirror/dockerhub-mirror.yaml
kubectl apply -f playwright-cache/playwright-cache.yaml
kubectl -n athens rollout status deploy/athens
kubectl -n verdaccio rollout status deploy/verdaccio
kubectl -n dockerhub-mirror rollout status deploy/dockerhub-mirror
kubectl -n playwright-cache rollout status deploy/playwright-cache
```

**Registry host only — skip this block on a standalone worker:**

```bash
kubectl apply -f registry/registry.yaml
kubectl apply -f registry/gc-cronjob.yaml
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

Current repo configuration (versions from `values.yaml` and the runner Dockerfile):

| Component | Version / image |
| --- | --- |
| ARC controller and runner scale set charts | `0.15.1` |
| Runner image | `ghcr.io/ntuim-imta/gha-runner:go1.27-node26` |
| Go toolcache | `1.27.4` |
| Node.js toolcache | `26.13.1` |
| Python toolcache | `3.14.10` |

The runner image tag is floating; `imagePullPolicy: Always` checks it for each
new runner pod. These versions describe the repo configuration, not a check for
the newest upstream releases.

**Standalone worker without cache services:** copy `values.yaml` to
`values-worker.yaml` and make these changes before running the install below:

| Setting | Change |
| --- | --- |
| dind `--registry-mirror` and `--insecure-registry` arguments | Remove both |
| `GOPROXY` | Set to `https://proxy.golang.org,direct` |
| `NPM_CONFIG_REGISTRY` | Set to `https://registry.npmjs.org/` |
| `PLAYWRIGHT_DOWNLOAD_HOST` | Remove |

Use `-f values-worker.yaml` instead of `-f values.yaml` in the install block
and later Helm upgrades. Cache DNS names from another cluster will not work.
The runner image is pulled directly from GHCR; no local build is required.
Workflows that push application images to the local registry still need an
existing registry or a different push destination configured in the consumer repo.

## Multiple runner hosts

NTUIM-IMTA is on GitHub Free and has only the `Default` runner group. GitHub
allows only Team org owners to create additional org-level runner groups, so
the separate-group design described in GitHub's
[multi-cluster guide](https://docs.github.com/en/actions/how-tos/manage-runners/use-actions-runner-controller/deploy-runner-scale-sets#high-availability-and-automatic-failover)
cannot be used here.

Use **one independent k3s cluster per runner host**, each with its own ARC
controller and runner scale set. All scale sets register in `Default`, so each
must have a unique name within that group. Name each set
`imta-runner-<hostname>` and assign all of them the shared `imta-runner` label
from `values.yaml`. Consumer workflows use `runs-on: imta-runner`; GitHub can
send a job to any eligible host, without guaranteeing even distribution.

On each host, set the following before the install, upgrade, verification,
backup, or removal commands below. The hostname must use lowercase letters,
digits, and hyphens and fit Kubernetes/Helm naming limits; set `RUNNER_HOST`
manually if the host's short hostname does not comply.

```bash
RUNNER_HOST="$(hostname -s)" # e.g. ws4090
SCALE_SET="imta-runner-${RUNNER_HOST}"
```

Install each host's scale set using its derived release name:

```bash
helm install "$SCALE_SET" \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --version 0.15.1 \
  -n arc-runners \
  --set githubConfigUrl=https://github.com/NTUIM-IMTA \
  --set githubConfigSecret=gh-config \
  --set runnerScaleSetName="$SCALE_SET" \
  -f values.yaml
```

The scale-set label is registered with GitHub only when the scale set is
created. Adding `scaleSetLabels` with `helm upgrade` to an existing scale set
updates its in-cluster `AutoscalingRunnerSet` but does not update GitHub's
registration; jobs targeting that label will remain queued. To change a
registered label, install a new scale set under a new name, verify it receives
a job, update consumer workflows, and only then uninstall the old set. Do not
reuse the old name while its GitHub registration exists.

Migration from the legacy `my-runners` set:

1. On each host, install `imta-runner-<hostname>` as above, using the existing
   `values.yaml` (or `values-worker.yaml` on a standalone worker).
2. Verify its `runner-scale-set-id` and submit a test job with
   `runs-on: imta-runner` to confirm it starts on the new set.
3. Change consumer workflow `runs-on` values to `imta-runner`.
4. After queued/running jobs drain and the new set is serving jobs, uninstall
   the old scale set on its host. `my-runners` is only the legacy set during
   this migration; do not use it for new installs.

On the host that still has the legacy release, remove it only after the checks
above:

```bash
helm uninstall my-runners -n arc-runners --wait
```

Each host pulls the runner image from GHCR and caches it in its own cluster;
there is no shared image cache between clusters. Configure WARP on the second
host too, or its initial pull (about 1 GB) can be very slow; see the network
setup below.

Consumer `build-image` workflows running on self-hosted runners push images to
`registry.imta.im.ntu.edu.tw:5000`, which resolves to `10.1.106.110` (`ws4090`).
BuildKit's `buildcache` is stored in that same registry. Keep this as the only
registry in the B layout: run the registry on `ws4090`, not on the second host.
On the second host, apply the node DNS and containerd HTTP mirror setup from
[本地 image registry](#本地-image-registry); the runner's dind
`--insecure-registry` is already configured in `values.yaml`. WARP's default
exclude range includes `10.0.0.0/8`, so image pushes to this private registry
continue over the LAN. This makes `ws4090` a single point of failure: while it
is down, the second host can still run tests, but `build-image` cannot push and
production cannot pull newly built images. A future option is moving the
registry to a dedicated host or pushing to GHCR instead.

Athens, Verdaccio, `dockerhub-mirror`, and `playwright-cache` use ClusterIP
services, so the second cluster cannot reach the first cluster's instances.
The recommended setup is to deploy a local copy of these cache services in
each cluster, following [step 5](#5-apply-the-in-cluster-mirrors-and-registry).
Alternatively, use the existing `values-worker.yaml` standalone-worker
settings to connect directly to upstream services, together with WARP; see
[the standalone-worker guidance](#6-install-the-runner-scale-set). These
options and registry setup are described in [step 5](#5-apply-the-in-cluster-mirrors-and-registry)
and [本地 image registry](#本地-image-registry), without duplicating their
deployment instructions here.

### Host network acceleration (Taiwan campus)

Campus access to GitHub/GHCR can be slow (about 340 KB/s in the measured
environment). On `10.1.106.110`, Cloudflare WARP raised throughput to about
9 MB/s. WARP's default bypass list does not include NTU's public address space,
so SSH replies to clients from `140.112.x.x` may leave through WARP and break
the connection. Add the campus range to the tunnel bypass, use tunnel-only
mode so WARP does not take over DNS (which can disrupt k3s CoreDNS), then
connect:

```bash
warp-cli tunnel ip add-range 140.112.0.0/16
warp-cli mode tunnel_only
warp-cli connect
```

For a remote first-time enablement, schedule a rollback before changing WARP;
if the SSH session drops, it disconnects WARP after 90 seconds:

```bash
(sleep 90; warp-cli disconnect) &
ROLLBACK_PID=$!
```

After confirming a new SSH connection succeeds, cancel that exact process with
`kill "$ROLLBACK_PID"`. Do not use `pkill -f`: it can match the SSH shell running
the command itself.

### 7. Verify

```bash
kubectl -n arc-systems get pods
kubectl -n arc-runners get autoscalingrunnerset "$SCALE_SET"
# Also check node readiness and runner placement when adding a worker.
kubectl get nodes -o wide
kubectl -n arc-runners get pods -o wide
```

Consumer workflows should use `runs-on: imta-runner`.

## Workflow conventions

### 手動清理本 repo 的 GHCR images

在 Actions → **Clean up GHCR images** → **Run workflow** 手動觸發
`.github/workflows/cleanup-images.yml`（`workflow_dispatch`）。它會分別清理
`gha-runner` 和 `verdaccio-s3`，依 `updated_at` 保留最新的已標記 image 版本
及其所有 tags、平台 manifests 與 build attestations，刪除其餘版本（包含未標記版本）。
沒有已標記版本時會跳過；此流程不清理本地 registry。

兩個 GHCR package 的 **Package settings → Manage Actions access** 必須授予
本 repo **Admin** 權限，workflow 才能使用 `GITHUB_TOKEN` 刪除版本；
見 [GitHub 的 package 刪除權限說明](https://docs.github.com/en/packages/learn-github-packages/deleting-and-restoring-a-package)。
清理時請避免同時推送 image；舊 tag 被刪除後，引用該 tag 的部署將無法重新拉取。

### Consumer workflows

In each job, keep two `runs-on` lines as a pair: the active pair below uses the
self-hosted runner by default and uses `ubuntu-latest` when
`workflow_dispatch` targets `gha`. The commented pair is the reverse (GitHub
hosted by default, self-hosted when dispatch targets `local`). To switch
defaults, comment/uncomment both pairs together; keep their conditions in sync
with the workflow's dispatch target logic.

```yaml
jobs:
  test:
    # Active pair: self-hosted by default; workflow_dispatch target=gha uses ubuntu-latest.
    runs-on: ${{ github.event_name == 'workflow_dispatch' && inputs.target == 'gha' && 'ubuntu-latest' || 'imta-runner' }}
    # Alternative pair: ubuntu-latest by default; workflow_dispatch target=local uses self-hosted.
    # runs-on: ${{ github.event_name == 'workflow_dispatch' && inputs.target == 'local' && 'imta-runner' || 'ubuntu-latest' }}
    steps:
      - uses: actions/checkout@v7
      - uses: actions/setup-go@v7
        with:
          go-version: "1.27.4" # matches the pre-seeded runner toolcache
          cache: false         # GOPROXY routes through Athens instead
      - uses: pnpm/action-setup@v6
      - uses: actions/setup-node@v7
        with:
          node-version: "26.13.1" # matches the pre-seeded runner toolcache
          cache: false         # NPM_CONFIG_REGISTRY routes through Verdaccio
      - run: cd backend && go test -race -cover ./...
      - run: cd frontend && pnpm install --frozen-lockfile
```

## Tuning knobs

All knobs live in `values.yaml`. After editing:

```bash
RUNNER_HOST="$(hostname -s)" # e.g. ws4090
SCALE_SET="imta-runner-${RUNNER_HOST}"
helm upgrade "$SCALE_SET" \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --version 0.15.1 \
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
for containerMode dind on 0.15.1, plus the `[delta]`-marked additions.

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
RUNNER_HOST="$(hostname -s)" # e.g. ws4090
SCALE_SET="imta-runner-${RUNNER_HOST}"
helm template "$SCALE_SET" \
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
        image: ghcr.io/ntuim-imta/gha-runner:go1.27-node26
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
| `services:` images (postgres, redis, …) | `dockerhub-mirror/` — registry:3 pull-through cache | dind `--registry-mirror` flag in `values.yaml` |
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
> - **Verdaccio needs Verdaccio >= 7**; the custom image uses the stable `7.1.1`
>   release, pinned by digest. Build/push the custom image before applying
>   ([BUILD.md](BUILD.md)).
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

- `registry/registry.yaml` — Namespace、PVC（request 20Gi，`local-path`）、`registry:3`
  Deployment（已開啟 `REGISTRY_STORAGE_DELETE_ENABLED`）、`LoadBalancer:5000`
  （k3s ServiceLB 綁 node IP）。
- `registry/gc-cronjob.yaml` — 每日 04:00 的 GC CronJob：每個 repo 保留最新 5 個
  `sha-*` tag，刪除其餘後回收 blob 空間。

PVC request 為 20Gi，與叢集上已綁定的 PVC 相同；目前約使用 13.8 GB。
`local-path` 不強制容量上限，需自行留意主機磁碟用量。PVC 綁定後不可修改
`spec`，不要靠改 manifest 嘗試調整既有 PVC。

### node 一次性設定

**Registry 主機**需設定下面的 DNS 與對外 port 5000。
**只當 worker** 可略過這兩項；若工作負載仍要從既有 HTTP registry 拉 image，
仍需在 worker 套用下面的 `registries.yaml`。這是 containerd 的拉取設定，
不會設定 runner 的 dind daemon；只拉公開 GHCR image 時不需要此設定。

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
3. 獨立 k3s 主機用 `sudo systemctl restart k3s` 套用；加入既有叢集的 worker
   則用 `sudo systemctl restart k3s-agent`。
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

### 搬遷到另一台主機（計畫性）

所有使用者都透過 `registry.imta.im.ntu.edu.tw:5000` 存取 registry：CI 推送、
ArgoCD image 路徑及各節點 `/etc/rancher/k3s/registries.yaml` 都使用名稱，
因此搬遷只需複製資料並更新 DNS，不需改這些設定。DNS 記錄位於
`NTUIM-IMTA/dns-zonefile` 的 `db.ntuim.cc`，目前為
`registry.imta IN A 10.1.106.110`；zone 預設 `$TTL 3600`。push 後由 cron
執行 `dns-deploy.sh`，自動跑 `named-checkconf` 並以 `rndc reload` 套用。

1. 在新主機安裝 k3s，部署 `registry/registry.yaml`，完成上方「node 一次性設定」
   並在 PVE 防火牆開放 TCP 5000。設定新 registry 的資料 PVC 與服務就緒。
2. 搬遷前一天將 `db.ntuim.cc` 中 registry A 記錄的 TTL 降為 60 秒並 push，
   確認 DNS 部署完成。
3. 保持舊 registry 可寫，先以 `regctl image copy` 或 `regsync` 透過新主機 IP
   直連新 registry 做完整初次同步；不必搬 `*-buildcache` repo。同步來源與目的
   使用各自的 IP 和 `:5000`，避免受 DNS 切換影響。
4. 選擇沒有 CI 推送的時段，將舊 registry 設為唯讀，再做增量同步並切換 DNS：

   ```bash
   kubectl -n registry set env deployment/registry \
     REGISTRY_STORAGE_MAINTENANCE_READONLY_ENABLED=true
   ```

   等 deployment rollout 完成後，用 `regsync` 或 `regctl image copy` 再同步一次，
   將 DNS A 記錄改為新主機 IP 並 push。確認自動部署與 DNS 生效後，不要讓舊、
   新 registry 同時接受寫入。
5. 驗證 `dig registry.imta.im.ntu.edu.tw` 指向新 IP、
   `curl http://registry.imta.im.ntu.edu.tw:5000/v2/_catalog` 可列出 repositories；
   手動觸發一次 `build-image`，並確認 ArgoCD 能拉取、部署新 image。
   CoreDNS 可能快取舊解析約 30 秒，稍候再判斷各節點結果。
6. 在新主機套用 `registry/gc-cronjob.yaml`，確認新 GC 正常後刪除舊主機的
   `registry-gc` CronJob，避免兩邊各自 GC。同一時間只能由目前主 registry 執行 GC。
7. 回滾時將 DNS 改回舊 IP，並在舊主機解除唯讀：

   ```bash
   kubectl -n registry set env deployment/registry \
     REGISTRY_STORAGE_MAINTENANCE_READONLY_ENABLED-
   ```

### 緊急切換（舊主機已失效）

若沒有事先同步，新 registry 會是空的。已運行的 pod 不受影響，因為 image 已在
各節點上；但 pod 重新排程或新部署時會因無法拉取 image 而失敗。先把 DNS 改到
新 registry，再針對各 repo 目前部署中的 commit，以 `workflow_dispatch` 的
`target=local` 重跑 `build-image`。image tag 由 commit SHA 產生，會與目前
ArgoCD chart 中引用的 tag 相同；確認 push 成功後再讓 ArgoCD 部署。

建議平時就在第二台 host 維持一個副 registry，由主 registry 每 10–15 分鐘透過
`regsync` 單向同步 image（排除 `*-buildcache`）。如此主機故障時副本已有近期資料，
切換只需更新 DNS。

### 舊 registry 下線

切換完成後先觀察 1–2 天，再停止舊服務：

1. 確認舊 registry log 沒有健康檢查以外的請求；各 repo、chart 與節點設定沒有
   寫死 `10.1.106.110:5000`；新 registry 已經歷一輪 CI 推送、ArgoCD 部署與 GC。
2. 停止服務但保留資料：

   ```bash
   kubectl -n registry delete cronjob registry-gc
   kubectl -n registry scale deployment/registry --replicas=0
   kubectl -n registry delete svc registry
   ```

   並關閉 PVE 防火牆上舊主機 TCP 5000 的規則。若需立即回復，可將 Deployment
   scale 回 1 並重新 apply registry Service（`kubectl apply -f registry/registry.yaml`）。
3. 再觀察 1–2 週才刪除舊 PVC。`kubectl delete -f registry/registry.yaml` 會一併
   刪除 PVC；`local-path` 的 Delete 回收會清除
   `/var/lib/rancher/k3s/storage/pvc-…_registry_registry-data`。刪除前可用 `sudo`
   備份該目錄。
4. 最後將 DNS TTL 改回 3600，並更新本 README 中 registry 主機的說明。

**不可在 DNS 切換前刪除 PVC，也不可讓新舊 registry 同時執行 GC。**

## Update / uninstall

```bash
# Upgrade ARC (controller + scale set). Helm never upgrades CRDs. For most
# upgrades, follow
# GitHub's documented path: uninstall both, drop the actions.github.com CRDs,
# reinstall. Do it while no jobs are queued/running (runners are gone meanwhile).
# First re-render the dind template for <new-version> and re-diff values.yaml
# (see "When bumping the chart version" above).
RUNNER_HOST="$(hostname -s)" # e.g. ws4090
SCALE_SET="imta-runner-${RUNNER_HOST}"
mkdir -p ~/arc-backup && helm get values "$SCALE_SET" -n arc-runners -o yaml > ~/arc-backup/"${SCALE_SET}"-values.yaml
helm uninstall "$SCALE_SET" -n arc-runners --wait
kubectl get autoscalingrunnerset,ephemeralrunnerset,ephemeralrunner,autoscalinglistener -A   # wait until empty
helm uninstall arc -n arc-systems --wait
kubectl get crd -o name | grep actions.github.com | xargs kubectl delete
helm install arc \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
  --version <new-version> -n arc-systems --wait
helm install "$SCALE_SET" \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --version <new-version> -n arc-runners \
  --set githubConfigUrl=https://github.com/NTUIM-IMTA --set githubConfigSecret=gh-config \
  --set runnerScaleSetName="$SCALE_SET" \
  -f values.yaml --wait

# For a patch release whose CRDs only add validation rules, an in-place upgrade
# can avoid removing the controller/resources. First download both controller
# charts and diff crds/: use this only when the schema changes add validation
# bounds/length/count constraints and no other CRD or controller migration is
# needed. Helm still does not manage CRDs during upgrade.
# For example, ARC 0.15.0 -> 0.15.1 has identical scale-set chart templates;
# the controller chart only adds CRD maxItems/minimum/maximum/minLength
# validation and updates controller-gen, so that patch can use this path.
helm pull oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
  --version <current-version> --untar --untardir /tmp/arc-current
helm pull oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
  --version <new-version> --untar --untardir /tmp/arc-new
diff -ru /tmp/arc-current/gha-runner-scale-set-controller/crds/ \
  /tmp/arc-new/gha-runner-scale-set-controller/crds/
# If the diff is limited to added validation constraints:
kubectl apply --server-side --force-conflicts \
  -f /tmp/arc-new/gha-runner-scale-set-controller/crds/
helm upgrade arc \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller \
  --version <new-version> -n arc-systems --wait
helm upgrade "$SCALE_SET" \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --version <new-version> -n arc-runners --reuse-values -f values.yaml --wait

# Bump an in-cluster mirror image: apply only the workload objects. The PVC
# specs here are immutable once bound (and local-path cannot resize), so a
# whole-file `kubectl apply` fails on them.
python3 - athens/athens.yaml <<'EOF' | kubectl apply -f -
import re, sys
docs = re.split(r"(?m)^---\s*$", open(sys.argv[1]).read())
print("\n---\n".join(d for d in docs if re.search(r"(?m)^kind:\s*(Deployment|CronJob)\s*$", d)))
EOF

# Tear down
helm uninstall "$SCALE_SET" -n arc-runners
helm uninstall arc -n arc-systems
kubectl delete -f athens/athens.yaml
kubectl delete -f verdaccio/verdaccio.yaml
kubectl delete -f dockerhub-mirror/dockerhub-mirror.yaml
kubectl delete -f playwright-cache/playwright-cache.yaml
kubectl delete -f seaweedfs/seaweedfs.yaml
kubectl delete -f registry/gc-cronjob.yaml
kubectl delete -f registry/registry.yaml
```
