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
├── verdaccio/       # in-cluster npm / pnpm registry mirror manifest
│   └── verdaccio.yaml
├── values.yaml      # gha-runner-scale-set helm overrides
├── README.md
└── BUILD.md         # appendix: build/push the runner image (one-time)
```

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
GH_TOKEN=ghp_xxx   # classic PAT, never commit the real value
kubectl create namespace arc-runners
kubectl -n arc-runners delete secret gh-config
kubectl -n arc-runners create secret generic gh-config \
  --from-literal=github_token=$GH_TOKEN
```

### 5. Apply the in-cluster mirrors

```bash
kubectl apply -f athens/athens.yaml
kubectl apply -f verdaccio/verdaccio.yaml
kubectl get ns athens verdaccio
kubectl -n athens rollout status deploy/athens
kubectl -n verdaccio rollout status deploy/verdaccio
```

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
```
