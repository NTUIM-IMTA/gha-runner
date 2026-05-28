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
└── README.md
```

## Set up from a blank OS

Target: Ubuntu 24.04 LTS (or Debian-equivalent), x86_64, single node.

### 1. k3s

```bash
curl -sfL https://get.k3s.io | sh -
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
sed -i "s/127.0.0.1/$(hostname -I | awk '{print $1}')/g" ~/.kube/config
export KUBECONFIG=~/.kube/config
kubectl get nodes
```

### 2. helm

```bash
curl https://baltocdn.com/helm/signing.asc | sudo gpg --dearmor -o /usr/share/keyrings/helm.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] \
  https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update && sudo apt-get install -y helm
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

Create a Personal Access Token with `repo` + `admin:org` scopes (or a GitHub
App with equivalent permissions), then:

```bash
kubectl create namespace arc-runners
kubectl -n arc-runners create secret generic gh-config \
  --from-literal=github_token='ghp_xxx'
```

### 5. Build & push the runner image

Once-only login:

```bash
gh auth refresh -h github.com -s write:packages,read:packages
gh auth token | docker login ghcr.io -u <gh-user> --password-stdin
```

Versions must match the latest patch in
[actions/go-versions](https://raw.githubusercontent.com/actions/go-versions/main/versions-manifest.json)
and
[actions/node-versions](https://raw.githubusercontent.com/actions/node-versions/main/versions-manifest.json):

```bash
cd go1.26-node24
docker buildx build \
  --platform linux/amd64 \
  --build-arg GO_VERSION=1.26.3 \
  --build-arg NODE_VERSION=24.16.0 \
  -t ghcr.io/ntuim-imta/gha-runner:go1.26-node24 \
  -t ghcr.io/ntuim-imta/gha-runner:go1.26.3-node24.16.0 \
  --push .
```

### 6. Apply the in-cluster mirrors

```bash
kubectl apply -f athens/athens.yaml
kubectl apply -f verdaccio/verdaccio.yaml
kubectl -n athens rollout status deploy/athens
kubectl -n verdaccio rollout status deploy/verdaccio
```

### 7. Install the runner scale set

`values.yaml` carries the runner image, dind, proxy, `maxRunners`, `GOPROXY`,
and `NPM_CONFIG_REGISTRY` — the chart picks up the rest from the controller
install.

```bash
helm install my-runners \
  oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set \
  --version 0.14.2 \
  -n arc-runners \
  --set githubConfigUrl=https://github.com/NTUIM-IMTA \
  --set githubConfigSecret=gh-config \
  --set runnerGroup=gh-runner \
  -f values.yaml
```

### 8. Verify

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
maxRunners: 8
```

| Value | Peak pods | Notes |
|---|---|---|
| 2 | ~4 | Diagnosing OOM |
| 4 | ~8 | Conservative default |
| 8 | ~16 | Current setting |
| 16+ | risky | Only after scaling the node |

Watch `kubectl top node` while runs queue; bump down on memory pressure.

### Egress proxy (`proxy:`)

```yaml
proxy:
  http:
    url: http://140.112.106.22:3128
  https:
    url: http://140.112.106.22:3128
  noProxy:
    - localhost
    - 127.0.0.1
    - .svc
    - .cluster.local
    - 10.0.0.0/8
    - 140.112.0.0/16
    - ghcr.io
    - github.com
    - api.github.com
```

Append to `noProxy` to bypass Squid for a specific host. Comment the whole
`proxy:` block out to fall back to direct egress.

Verify inside a runner pod:

```bash
kubectl -n arc-runners exec <runner-pod> -c runner -- env | grep -i proxy
```

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

`Always` re-pulls on every job, so a `--push` with the same tag goes live on
the next run with no helm change.

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

## Bumping Go / Node

1. Look up the latest patch in the actions/*-versions manifests linked above.
2. Create a new folder `goX.Y-nodeZ/` and copy/edit the `Dockerfile` ARGs.
3. Build & push with both the floating tag (`goX.Y-nodeZ`) and the immutable
   tag (`goX.Y.Z-nodeA.B.C`).
4. Update `template.spec.containers[0].image` in `values.yaml`, run the
   `helm upgrade` from the tuning section.
5. Keep the old folder until no workflow pins it.

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
