# Custom Verdaccio image with the AWS S3 storage plugin baked in, so package
# metadata and tarballs live in the in-cluster MinIO (atomic object writes)
# instead of on the non-atomic local disk that produced 0-byte package.json
# poison-cache files. See README "Verdaccio storage: MinIO".
#
# The maintained verdaccio-aws-s3-storage (>= 12) requires Verdaccio >= 7, which
# at the time of writing only ships as a beta. Node >= 24 is also required (the
# base image already satisfies it).
#
# Build & push (one-time per version) — see BUILD.md.

# 1) Vendor the plugin and its full dependency tree (flattened into node_modules
#    by --install-strategy=shallow).
FROM node:lts-alpine AS builder
RUN mkdir -p /verdaccio/plugins \
 && cd /verdaccio/plugins \
 && npm install --install-strategy=shallow --no-bin-links --omit=optional \
      verdaccio-aws-s3-storage@^12

# 2) Bake it into Verdaccio. The plugin loader looks under /verdaccio/plugins
#    (the default `plugins` dir) for a folder named verdaccio-<store-key>, so the
#    plugin must sit there directly. Its dependencies (@aws-sdk/*, …) ride along
#    as a sibling node_modules so the plugin's require() calls resolve.
FROM verdaccio/verdaccio:7.0.0-beta.4
# Links the GHCR package to this repo (same convention as the runner image).
LABEL org.opencontainers.image.source=https://github.com/NTUIM-IMTA/gha-runner
COPY --chown=10001:root --from=builder \
  /verdaccio/plugins/node_modules /verdaccio/plugins/node_modules
COPY --chown=10001:root --from=builder \
  /verdaccio/plugins/node_modules/verdaccio-aws-s3-storage \
  /verdaccio/plugins/verdaccio-aws-s3-storage
