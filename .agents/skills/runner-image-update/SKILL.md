---
name: runner-image-update
description: >-
  Rebuild the self-hosted GitHub Actions runner image with the latest stable Go, Node,
  pnpm and golangci-lint, refresh the pre-seeded GitHub Actions archive cache to the
  commits the consumer workflows resolve to, and roll the new tag out through
  values.yaml + helm. Invoke whenever the user asks to update, bump, rebuild or refresh
  the runner image or its toolchain, mentions setup-go resolving a stale Go patch,
  "Download action repository" lines reappearing in job logs, or uses
  /runner-image-update. Keywords: "更新 runner image", "升級 Go/Node/pnpm", "action 版本",
  "runner cache", "rebuild runner", "bump toolchain".
---

# Runner Image Update

> This skill is version-controlled in `.agents/skills/runner-image-update/SKILL.md`
> (`.claude/skills` and `.codex/skills` are symlinks to `.agents/skills`).
> When the image layout, build host, or consumer repo list changes, update this file
> and `references/commands.md` in the same PR.

Consumer repos whose workflows run on this runner: `admission`, `intranet`, `theses`,
`print`, `ws-management` (all under `NTUIM-IMTA`). Build host: `imta@10.1.106.110`
(`~/gha-runner`, docker + buildx, logged in to GHCR for push; **no GitHub push
credentials** — push git commits from a machine with `gh auth`).

**Actions:**

1. 🔴 **Resolve target versions** (commands in `references/commands.md` §1):
   - Go: newest stable on go.dev that also exists in `actions/go-versions`.
   - Node: newest patch of the even (LTS-track) major the consumers pin (`node-version`).
   - pnpm: `npm view pnpm version`. golangci-lint: latest release, confirm its notes
     mention support for the chosen Go minor before bumping Go.
2. 🔴 **Resolve every action the consumers use** (§2): collect all `uses:` across the
   five repos' `.github/workflows/*.y*ml`, compute the latest major tag and the commit
   SHA it points to, and read each major bump's release notes for runner requirements
   (Node 24 actions need runner ≥ 2.327.1; base image is `actions-runner:latest`).
3. 🔴 **Edit the Dockerfile**: if the Go or Node major changed, `git mv` the folder to
   `go<X.Y>-node<Z>/` and update every `go1.NN-node26` reference in `README.md`,
   `BUILD.md`, `values.yaml`. Set `ARG GO_VERSION` / `ARG NODE_VERSION`, and rewrite the
   step-3 `seed` list so it contains exactly the actions found in step 2 with their
   resolved SHAs (one `# owner/repo@vN` comment per entry; drop unused actions).
4. 🔴 **Build and push on the build host** (§3) with both the floating tag
   (`go<X.Y>-node<Z>`) and the immutable tag (`go<X.Y.Z>-node<A.B.C>`); the host's
   clone must be at `origin/main` first. Verify with `docker buildx imagetools inspect`.
5. 🔴 **Roll out**: if the floating tag changed, update `values.yaml` and run the
   `helm upgrade` from README "Tuning knobs". If only the content changed, nothing to do
   (`imagePullPolicy: Always`, ephemeral pods).
6. 🔴 **Update the consumers** (§4): in each repo set `go-version`, `node-version`,
   pnpm `version`, golangci-lint `version`, and every `uses:` to the values from steps
   1–2 (refresh SHA pins and their `# vX.Y.Z` comments); bump `go.mod`, Dockerfile base
   images, `packageManager`, docs. Run each repo's `make lint && make test`, commit,
   push.
7. 🔴 **Verify CI**: every consumer's CI run on the new commit is green, and its
   "Set up job" step shows no `Download action repository` line for seeded actions.
8. 🟡 **Commit gha-runner**: `Dockerfile`, `values.yaml`, `README.md`, `BUILD.md`; push
   from a machine with GitHub credentials (§5).

**Pass criteria:** `imagetools inspect` shows the new digest on both tags; the
AutoscalingRunnerSet image equals the floating tag in `values.yaml`; all five consumer
CI runs are green with cache hits for every seeded action; gha-runner `main` contains
the Dockerfile with `ARG` values and a seed list matching what the consumers pin.

## References

- `references/commands.md` — copy-paste commands for each step.
- `BUILD.md` — image build background and the verdaccio-s3 image.
