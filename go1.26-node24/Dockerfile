# Custom ARC runner image
# - base: ghcr.io/actions/actions-runner (bare Ubuntu)
# - adds build-essential so `go test -race` (cgo) works
# - pre-seeds /opt/hostedtoolcache so actions/setup-go and actions/setup-node
#   hit "Found in cache" instead of downloading every run
FROM ghcr.io/actions/actions-runner:latest

# Link the GHCR package to this repo so it shows up under
# https://github.com/NTUIM-IMTA/gha-runner/pkgs/container/gha-runner
LABEL org.opencontainers.image.source=https://github.com/NTUIM-IMTA/gha-runner

# Must match the exact version setup-go / setup-node resolve to when the
# workflow says `go-version: "1.26"` / `node-version: 24`.
# Check the latest published patch at:
#   https://raw.githubusercontent.com/actions/go-versions/main/versions-manifest.json
#   https://raw.githubusercontent.com/actions/node-versions/main/versions-manifest.json
ARG GO_VERSION=1.26.3
ARG NODE_VERSION=24.16.0

USER root

# 1) C toolchain + common build deps + docker CLI (the daemon lives in a
#    dind sidecar; this image only needs the client side)
RUN set -eux; \
    apt-get update; \
    apt-get install -y --no-install-recommends \
      build-essential \
      ca-certificates \
      curl \
      git \
      gnupg \
      jq \
      pkg-config \
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
    rm -rf /var/lib/apt/lists/*

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

USER runner
