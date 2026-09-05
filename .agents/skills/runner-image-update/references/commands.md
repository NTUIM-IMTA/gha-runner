# runner-image-update — command reference

All commands assume a local checkout of the five consumer repos under `~/imta/`
(`admission intranet theses print ws_management`) and `gh` authenticated.

## §1 Resolve toolchain versions

```bash
# Go: newest stable on go.dev, and the newest patches actions/setup-go can install
curl -fsSL 'https://go.dev/dl/?mode=json' | jq -r '.[] | select(.stable) | .version' | head -2
curl -fsSL https://raw.githubusercontent.com/actions/go-versions/main/versions-manifest.json \
  | jq -r '[.[] | select(.version|startswith("1.27."))][0:2][] | .version'

# Node: newest patch of the LTS-track major in use (26 today)
curl -fsSL https://raw.githubusercontent.com/actions/node-versions/main/versions-manifest.json \
  | jq -r '[.[] | select(.version|startswith("26."))][0].version'

# pnpm
npm view pnpm version

# golangci-lint: latest release, and whether it supports the chosen Go minor
gh release list -R golangci/golangci-lint --limit 3
gh release view -R golangci/golangci-lint --json body --jq .body | grep -inE 'go ?1\.[0-9]+ support'
```

Rule of thumb: bump Go to a new minor only when golangci-lint's release notes list
support for it; otherwise stay on the newest patch of the current minor.

## §2 Resolve GitHub Actions

```bash
cd ~/imta
# What the consumers use (tags and SHA pins)
grep -rhoE 'uses: *[^ ]+' */.github/workflows/*.y*ml | sed 's/uses: *//' | sort | uniq -c | sort -rn

# Latest major tag + the commit it resolves to, for each owner/repo
for a in actions/checkout actions/setup-go actions/setup-node actions/upload-artifact \
         actions/setup-python actions/delete-package-versions pnpm/action-setup \
         golangci/golangci-lint-action docker/setup-buildx-action docker/login-action \
         docker/metadata-action crazy-max/ghaction-github-runtime rjstone/discord-webhook-notify; do
  latest=$(gh api "repos/$a/tags?per_page=100" \
    --jq '[.[].name | select(test("^v[0-9]+(\\.[0-9]+)*$"))] | sort_by(sub("^v";"") | split(".") | map(tonumber)) | last')
  major=${latest%%.*}
  echo "$a $latest $major $(gh api "repos/$a/commits/$major" --jq .sha)"
done

# Breaking-change scan for a major bump
gh release view v7.0.0 -R actions/upload-artifact --json body --jq .body | grep -iE 'break|node ?2[04]|runner'
```

The runner caches action tarballs by the SHA a tag resolves to at job time, so the
seed list must use the SHA of the **major** tag (`v7`), which is what workflows
reference. ws-management pins some actions by SHA with a `# vX.Y.Z — pinned:` comment;
update both the SHA and the comment there.

## §3 Build and push on the build host

```bash
ssh imta@10.1.106.110
cd ~/gha-runner && git fetch origin && git status   # must be clean and at origin/main
# apply the Dockerfile / values / docs edits (or push them to main first and pull)

nohup docker buildx build go1.27-node26 \
  --platform linux/amd64 \
  --build-arg GO_VERSION=1.27.1 \
  --build-arg NODE_VERSION=26.8.1 \
  -t ghcr.io/ntuim-imta/gha-runner:go1.27-node26 \
  -t ghcr.io/ntuim-imta/gha-runner:go1.27.1-node26.8.1 \
  --push > /tmp/runner-build.log 2>&1 &

tail -f /tmp/runner-build.log            # ends with "pushing manifest ... done"
docker buildx imagetools inspect ghcr.io/ntuim-imta/gha-runner:go1.27-node26 | grep Digest
```

Never `pkill -f "buildx build"` from an ssh command line that itself contains the
string — it kills the ssh session. Match on `^docker buildx build` via `ps` instead.

## §4 Update the consumer repos

Per repo, the files that carry toolchain versions:

| File | Fields |
|---|---|
| `.github/workflows/*.y*ml` | `go-version`, `node-version`, pnpm `version`, golangci-lint `version`, every `uses:` |
| `backend/go.mod` | `go X.Y.Z` |
| `Dockerfile` (+ `backend/Makefile`, `docker-compose.yml` in print) | `golang:X.Y.Z-alpine*`, `node:A.B.C-alpine`, corepack `pnpm@` |
| `frontend/package.json`, `e2e/package.json` | `packageManager` |
| `Makefile`, `CLAUDE.md`, `README.md`, `docs/**` | golangci-lint version, Go/pnpm requirement text |

`.devcontainer/docker-compose.yml` stays on `devcontainers/go:1.NN-bookworm` (no
per-patch tag exists). Leave `engines.node` alone. Pin exact versions in CI: the
runner's toolcache is pre-seeded, and `setup-go` with a bare minor (`"1.26"`) will
happily use a stale cached patch that is older than `go.mod`'s minimum.

```bash
export GOTOOLCHAIN=auto
for p in admission intranet theses print ws_management; do (cd ~/imta/$p && make lint && make test); done
```

## §5 Verify and commit

```bash
# CI status per repo
for r in admission intranet theses print ws-management; do
  gh run list -R NTUIM-IMTA/$r --branch develop --limit 1 --json name,status,conclusion,headSha \
    --jq '.[] | "\(.name) \(.status) \(.conclusion) \(.headSha[0:8])"'
done
# The runner always prints "Download action repository" (even on a cache hit; the hit
# itself is only logged at debug level). Check that every SHA is in the seed list:
gh run view <run-id> -R NTUIM-IMTA/<repo> --log \
  | grep -oE "Download action repository '[^']*' \(SHA:[0-9a-f]{8}" | sort -u

# gha-runner commit: the build host has no GitHub push credentials
ssh imta@10.1.106.110 'cd ~/gha-runner && git add -A && git -c user.name=<name> -c user.email=<email> commit -m "..."'
git clone git@github.com:NTUIM-IMTA/gha-runner.git /tmp/gha-runner && cd /tmp/gha-runner
git fetch imta@10.1.106.110:gha-runner HEAD && git push origin FETCH_HEAD:main
```
