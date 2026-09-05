# Custom ARC runner image
# - base: ghcr.io/actions/actions-runner (bare Ubuntu)
# - adds build-essential so `go test -race` (cgo) works
# - pre-seeds /opt/hostedtoolcache so actions/setup-go and actions/setup-node
#   hit "Found in cache" instead of downloading every run
# - pre-seeds the action archive cache so "Download action repository" steps
#   skip the per-job GitHub download (ephemeral pods re-download otherwise)
FROM ghcr.io/actions/actions-runner:latest

# Link the GHCR package to this repo so it shows up under
# https://github.com/NTUIM-IMTA/gha-runner/pkgs/container/gha-runner
LABEL org.opencontainers.image.source=https://github.com/NTUIM-IMTA/gha-runner

# Must match the exact version setup-go / setup-node resolve to when the
# workflow says `go-version: "1.27"` / `node-version: 26`.
# Check the latest published patch at:
#   https://raw.githubusercontent.com/actions/go-versions/main/versions-manifest.json
#   https://raw.githubusercontent.com/actions/node-versions/main/versions-manifest.json
ARG GO_VERSION=1.27.1
ARG NODE_VERSION=26.8.1

# Self-hosted runners default RUNNER_TOOL_CACHE to ~/_work/_tool (ephemeral).
# Point it at the persistent toolcache we pre-seed below so setup-go /
# setup-node hit "Found in cache".
ENV RUNNER_TOOL_CACHE=/opt/hostedtoolcache

# Local cache of action tarballs ({owner}_{repo}/{sha}.tar.gz). The runner
# checks this folder before downloading an action; on a miss it falls back to
# downloading from GitHub, so stale SHAs degrade gracefully.
ENV ACTIONS_RUNNER_ACTION_ARCHIVE_CACHE=/opt/action-archive-cache

USER root

# 1) C toolchain + common build deps + docker CLI (the daemon lives in a
#    dind sidecar; this image only needs the client side).
#    qpdf + ghostscript are for the print repo's CI (backend-test /
#    backend-integration / e2e shell out to them); its "Install PDF tooling"
#    steps guard with `command -v`, so baking them here turns those apt-get
#    runs into no-ops.
#    netcat-openbsd provides `nc`, which service-readiness gates use
#    (`nc -z localhost <port>`); the base runner image ships without it.
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
      build-essential \
      ca-certificates \
      curl \
      ghostscript \
      git \
      gnupg \
      jq \
      netcat-openbsd \
      pkg-config \
      qpdf \
      unzip; \
    install -m 0755 -d /etc/apt/keyrings; \
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc; \
    chmod a+r /etc/apt/keyrings/docker.asc; \
    . /etc/os-release; \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu ${VERSION_CODENAME} stable" \
      > /etc/apt/sources.list.d/docker.list; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
      docker-ce-cli \
      docker-buildx-plugin; \
    rm -rf /var/lib/apt/lists/*; \
    # yq (mikefarah, Go) — used by build-image workflow to edit Helm chart values
    curl -fsSL https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 \
      -o /usr/local/bin/yq; \
    chmod +x /usr/local/bin/yq

# 2) Pre-seed toolcache using the exact tarballs setup-go / setup-node download.
#    Cluster is linux/amd64; the GitHub toolcache arch label for x86_64 is "x64".
RUN set -eux; \
    # Go
    GO_URL="$(curl -fsSL https://raw.githubusercontent.com/actions/go-versions/main/versions-manifest.json \
      | jq -r --arg v "$GO_VERSION" '.[] | select(.version==$v) | .files[] | select(.platform=="linux" and .arch=="x64") | .download_url' \
      | head -n1)"; \
    test -n "$GO_URL"; \
    mkdir -p "/opt/hostedtoolcache/go/${GO_VERSION}/x64"; \
    curl -fsSL "$GO_URL" \
      | tar -xz -C "/opt/hostedtoolcache/go/${GO_VERSION}/x64" --strip-components=1; \
    touch "/opt/hostedtoolcache/go/${GO_VERSION}/x64.complete"; \
    # Node
    NODE_URL="$(curl -fsSL https://raw.githubusercontent.com/actions/node-versions/main/versions-manifest.json \
      | jq -r --arg v "$NODE_VERSION" '.[] | select(.version==$v) | .files[] | select(.platform=="linux" and .arch=="x64") | .download_url' \
      | head -n1)"; \
    test -n "$NODE_URL"; \
    mkdir -p "/opt/hostedtoolcache/node/${NODE_VERSION}/x64"; \
    curl -fsSL "$NODE_URL" \
      | tar -xz -C "/opt/hostedtoolcache/node/${NODE_VERSION}/x64" --strip-components=1; \
    touch "/opt/hostedtoolcache/node/${NODE_VERSION}/x64.complete"; \
    chown -R runner:runner /opt/hostedtoolcache

# 3) Pre-seed the action archive cache with the actions our workflows pin.
#    SHAs are the commits the version tags currently resolve to; when a tag
#    moves, the runner just re-downloads until this list is refreshed.
#    Find the SHA either way:
#      - any job log's "Set up job" step prints it:
#        Download action repository 'actions/checkout@v7' (SHA:3d3c42e5...)
#        (on a cache hit that line disappears — seeing it again means the tag
#        moved and this list needs a refresh)
#      - resolve it directly: gh api repos/<owner>/<repo>/commits/<tag> --jq .sha
RUN set -eux; \
    seed() { \
      dir="${ACTIONS_RUNNER_ACTION_ARCHIVE_CACHE}/$(printf '%s' "$1" | tr '/' '_')"; \
      mkdir -p "$dir"; \
      curl -fsSL "https://codeload.github.com/$1/tar.gz/$2" -o "${dir}/$2.tar.gz"; \
    }; \
    # actions/checkout@v7
    seed actions/checkout 3d3c42e5aac5ba805825da76410c181273ba90b1; \
    # actions/setup-go@v7
    seed actions/setup-go b7ad1dad31e06c5925ef5d2fc7ad053ef454303e; \
    # actions/setup-node@v7
    seed actions/setup-node 820762786026740c76f36085b0efc47a31fe5020; \
    # actions/upload-artifact@v7
    seed actions/upload-artifact 043fb46d1a93c77aae656e7c1c64a875d1fc6a0a; \
    # actions/setup-python@v7
    seed actions/setup-python 5fda3b95a4ea91299a34e894583c3862153e4b97; \
    # pnpm/action-setup@v6
    seed pnpm/action-setup 0977fd99725f1db4007ccb2928dbb4e90d06cc86; \
    # golangci/golangci-lint-action@v9
    seed golangci/golangci-lint-action ba0d7d2ec06a0ea1cb5fa41b2e4a3ab91d21278a; \
    # docker/setup-buildx-action@v4
    seed docker/setup-buildx-action 37fe631027851001ddb9b187196cc803df7f5f0e; \
    # docker/login-action@v4
    seed docker/login-action dbcb813823bdd20940b903addbd779551569679f; \
    # docker/metadata-action@v6
    seed docker/metadata-action dc802804100637a589fabce1cb79ff13a1411302; \
    # crazy-max/ghaction-github-runtime@v4 (also SHA-pinned to this commit in ws-management)
    seed crazy-max/ghaction-github-runtime 04d248b84655b509d8c44dc1d6f990c879747487; \
    # actions/delete-package-versions@v5 (Delete-Old-Packages workflow)
    seed actions/delete-package-versions e5bc658cc4c965c472efe991f8beea3981499c55; \
    # rjstone/discord-webhook-notify@v2 (v2.3.0; also SHA-pinned in ws-management)
    seed rjstone/discord-webhook-notify 13eb215cbf853107dd9386c4a2f4035b7988cfc2; \
    chown -R runner:runner "$ACTIONS_RUNNER_ACTION_ARCHIVE_CACHE"

# 4) System libraries for Playwright's chromium, so the consumer e2e jobs can
#    run `playwright install chromium` WITHOUT `--with-deps` (no per-job
#    apt-get; the browser binary itself comes through the in-cluster
#    playwright-cache). The library list is stable across Playwright versions;
#    install-deps re-resolves it at build time. Uses the Node seeded in step 2.
RUN set -eux; \
    export PATH="/opt/hostedtoolcache/node/${NODE_VERSION}/x64/bin:${PATH}"; \
    export DEBIAN_FRONTEND=noninteractive; \
    apt-get update; \
    npx --yes playwright install-deps chromium; \
    rm -rf /var/lib/apt/lists/* /root/.npm

USER runner
