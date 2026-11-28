---
name: runner-image-update
description: >-
  Refresh the self-hosted GitHub Actions runner image: bump Go, Node, Python, pnpm and
  golangci-lint to the latest stable releases, re-seed the action archive cache with the
  commits the consumer workflows resolve to, let CI rebuild and push the image, and roll
  the new versions out to the five consumer repos. Use when asked to update, rebuild or
  check the runner image or its toolchain, when setup-go resolves a stale Go patch, or
  when "Download action repository" lines show SHAs that are missing from the seed list.
---

# Runner Image Update

Every consumer CI job runs on this image, so a stale toolchain or seed list costs each
job a download from GitHub, which is slow from the runner hosts. The goal is an image
whose toolchain and action cache match exactly what the consumers pin.

This skill lives in `.agents/skills/runner-image-update/` (`.claude/skills` and
`.codex/skills` are symlinks to `.agents/skills`). Update it and
`references/commands.md` in the same commit whenever the image layout, the build
workflow, or the consumer repo list changes.

Consumer repos (all under `NTUIM-IMTA`): `admission`, `intranet`, `theses`, `print`,
`ws-management`.

## How the image gets built

`.github/workflows/build-runner-image.yml` builds on `ubuntu-latest` and pushes both
tags whenever `go<X.Y>-node<Z>/Dockerfile` or the workflow changes on `main`. It can
also be started with `workflow_dispatch`. It reads `GO_VERSION` / `NODE_VERSION` from
the Dockerfile `ARG` lines and passes no build args, so the `ARG` defaults are the
source of truth for the floating tag (`go<X.Y>-node<Z>`) and the immutable tag
(`go<X.Y.Z>-node<A.B.C>`). A manual build on the build host is only a fallback for when
CI is unavailable (`references/commands.md` §3b).

## Steps

1. **Resolve target versions** (§1). Go: newest stable on go.dev that is also in
   `actions/go-versions`. Node: newest patch of the even (LTS-track) major the consumers
   pin. Python: newest patch of the minor in ws-management's ansible job. pnpm: `npm view
   pnpm version`. golangci-lint: latest release. Bump Go to a new minor only after
   golangci-lint's release notes list support for it.
2. **Resolve every action the consumers use** (§2): collect all `uses:` across the five
   repos' workflows, find the latest major tag of each and the commit that major tag
   points to. Read the release notes of any major bump for runner requirements (base
   image is `actions-runner:latest`).
3. **Edit the Dockerfile.** Set the `ARG` defaults and rewrite the step-3 `seed` list so
   it holds exactly the actions from step 2 with their SHAs, one `# owner/repo@vN`
   comment per entry, unused actions dropped. If the Go or Node major/minor changes,
   `git mv` the folder to `go<X.Y>-node<Z>/` and update every reference to the old name:
   the workflow's `paths`, `dockerfile=` and `context:`, plus `README.md`, `BUILD.md`
   and `values.yaml`. Forgetting the workflow means CI never builds the new folder.
4. **Commit and push to `main`** (§3). The push starts the build. Watch the run, then
   confirm both tags on GHCR carry the new digest.
5. **Roll out.** If the floating tag changed, update `values.yaml` and run the
   `helm upgrade` from README "Tuning knobs". If only the content changed there is
   nothing to do: `imagePullPolicy: Always` and ephemeral pods pick it up on the next job.
6. **Update the consumers** (§4): `go-version`, `node-version`, pnpm `version`,
   golangci-lint `version` and every `uses:` from steps 1–2. All five repos pin
   `crazy-max/ghaction-github-runtime` and `rjstone/discord-webhook-notify` by SHA with
   a `# vX.Y.Z — pinned: <reason>` comment; refresh both the SHA and the comment. Also
   bump `go.mod`, Dockerfile base images, `packageManager` and docs, then run each
   repo's own dev-check before committing there.
7. **Verify consumer CI** (§5): every consumer's run on the new commit is green, and
   every `Download action repository ... (SHA:...)` line in "Set up job" names a SHA in
   the seed list.

**Done when** the `Build runner image` run for the pushed commit succeeded, both GHCR
tags show the same new digest, the AutoscalingRunnerSet image equals the floating tag
in `values.yaml`, and the seed list on `main` matches what the consumers pin (all five
consumer CI runs green, every action SHA in their logs present in the seed list).

## References

- `references/commands.md` — copy-paste commands for each step.
- `BUILD.md` — background on the image layout and the verdaccio-s3 image.
