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
# workflow says `go-version: "1.26"` / `node-version: 26`.
# Check the latest published patch at:
#   https://raw.githubusercontent.com/actions/go-versions/main/versions-manifest.json
#   https://raw.githubusercontent.com/actions/node-versions/main/versions-manifest.json
ARG GO_VERSION=1.26.4
ARG NODE_VERSION=26.3.0

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
#        Download action repository 'actions/checkout@v6' (SHA:df4cb1c0...)
#        (on a cache hit that line disappears — seeing it again means the tag
#        moved and this list needs a refresh)
#      - resolve it directly: gh api repos/<owner>/<repo>/commits/<tag> --jq .sha
RUN set -eux; \
    seed() { \
      dir="${ACTIONS_RUNNER_ACTION_ARCHIVE_CACHE}/$(printf '%s' "$1" | tr '/' '_')"; \
      mkdir -p "$dir"; \
      curl -fsSL "https://codeload.github.com/$1/tar.gz/$2" -o "${dir}/$2.tar.gz"; \
    }; \
    # actions/checkout@v6
    seed actions/checkout df4cb1c069e1874edd31b4311f1884172cec0e10; \
    # docker/setup-buildx-action@v4
    seed docker/setup-buildx-action d7f5e7f509e45cec5c76c4d5afdd7de93d0b3df5; \
    # docker/login-action@v4
    seed docker/login-action 650006c6eb7dba73a995cc03b0b2d7f5ca915bee; \
    # docker/metadata-action@v6
    seed docker/metadata-action 80c7e94dd9b9319bd5eb7a0e0fe9291e23a2a2e9; \
    # docker/build-push-action@v7
    seed docker/build-push-action f9f3042f7e2789586610d6e8b85c8f03e5195baf; \
    # rjstone/discord-webhook-notify@v2
    seed rjstone/discord-webhook-notify 510c47fe8d6583687298dc08f8d0306957b82a9b; \
    # actions/checkout@v4 (README workflow conventions, consumer CI jobs)
    seed actions/checkout 34e114876b0b11c390a56381ad16ebd13914f8d5; \
    # actions/setup-go@v5
    seed actions/setup-go 40f1582b2485089dde7abd97c1529aa768e1baff; \
    # actions/setup-node@v4
    seed actions/setup-node 49933ea5288caeca8642d1e84afbd3f7d6820020; \
    # pnpm/action-setup@v4
    seed pnpm/action-setup b906affcce14559ad1aafd4ab0e942779e9f58b1; \
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
