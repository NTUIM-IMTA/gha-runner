# Custom ARC runner image
# - base: ghcr.io/actions/actions-runner (bare Ubuntu)
# - adds build-essential so `go test -race` (cgo) works
# - pre-seeds /opt/hostedtoolcache so actions/setup-go and actions/setup-node
#   hit "Found in cache" instead of downloading every run
FROM ghcr.io/actions/actions-runner:latest

# Bump these when workflows pin a newer minor/patch.
# Must be a full semver that satisfies the workflow's `go-version: "1.26"`
# and `node-version: 24` constraints.
ARG GO_VERSION=1.26.0
ARG NODE_VERSION=24.10.0

USER root

# 1) C toolchain + common build deps
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      build-essential \
      ca-certificates \
      curl \
      git \
      jq \
      pkg-config \
      unzip \
 && rm -rf /var/lib/apt/lists/*

# 2) Pre-seed toolcache for setup-go / setup-node
RUN set -eux; \
    arch="$(dpkg --print-architecture)"; \
    case "$arch" in \
      amd64) GO_ARCH=amd64; NODE_ARCH=x64 ;; \
      arm64) GO_ARCH=arm64; NODE_ARCH=arm64 ;; \
      *) echo "unsupported arch $arch"; exit 1 ;; \
    esac; \
    # Go
    mkdir -p "/opt/hostedtoolcache/go/${GO_VERSION}/${GO_ARCH}"; \
    curl -fsSL "https://go.dev/dl/go${GO_VERSION}.linux-${GO_ARCH}.tar.gz" \
      | tar -xz -C "/opt/hostedtoolcache/go/${GO_VERSION}/${GO_ARCH}" --strip-components=1; \
    touch "/opt/hostedtoolcache/go/${GO_VERSION}/${GO_ARCH}.complete"; \
    # Node
    mkdir -p "/opt/hostedtoolcache/node/${NODE_VERSION}/${NODE_ARCH}"; \
    curl -fsSL "https://nodejs.org/dist/v${NODE_VERSION}/node-v${NODE_VERSION}-linux-${NODE_ARCH}.tar.xz" \
      | tar -xJ -C "/opt/hostedtoolcache/node/${NODE_VERSION}/${NODE_ARCH}" --strip-components=1; \
    touch "/opt/hostedtoolcache/node/${NODE_VERSION}/${NODE_ARCH}.complete"; \
    chown -R runner:runner /opt/hostedtoolcache

USER runner
