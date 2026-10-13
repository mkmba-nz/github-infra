# Go Image Build Caching Research

Tracking Issue: [FZ-2527](https://fizzy.rohu-bass.ts.net/1/cards/2527)

## Overview

Every MKMBA container image holding a Go binary except traefik is built through
`build-and-push.yml` in this repository. The workflow offers two caching layers: BuildKit's
`type=gha` layer cache (always on) and, since June 2026, opt-in persistence of
`RUN --mount=type=cache` directories through `buildkit-cache-dance` (the `cache-mounts`
input, "cache-dance" below). A build failure on 2026-10-13
([FZ-2522](https://fizzy.rohu-bass.ts.net/1/cards/2522)) showed that a Dockerfile written
for one layer can fail under the other, and that the combination is hard to reason about.

This document inventories every Go image build in the `mkmba-nz` organisation, sizes each
build, measures what the current caching actually delivers from the recorded workflow runs,
and ranks the four build methods the card names. The measurements come from existing runs
rather than new experiments; whether live experiments are still wanted is the first scope
question.

*Omitted: User Stories (restates the card); Target/Destination (the shared workflow is
described under Current System Analysis instead).*

## Problem Statement

Builds fail or run slowly for reasons that are not visible in the Dockerfile or the calling
workflow:

1. **Latent failure.** A `go mod download` step that writes into a cache mount leaves nothing
   in its layer. When the layer cache hits but the mount is cold (no `cache-mounts`, or an
   evicted entry), `go build` re-downloads every module - and fails on private modules
   because the credential was only mounted on the download step. Run
   [tokenizer 38369691373](https://github.com/mkmba-nz/tokenizer/actions/runs/38369691373)
   shows exactly this: steps 2-6 `CACHED`, then `go build` printing `go: downloading ...`
   for every module and dying on `mkmba.nz/x` with
   `fatal: could not read Username for 'https://github.com'`.
2. **Hidden cost.** Cache-dance adds five conditional steps (resolve, restore, inject, and
   extract + save as post-steps) whose cost is not attributed to the build step. Measured
   below, a warm cache shortens the job by at most tens of seconds, and never below the
   no-dance build of the same image.
3. **Inconsistency.** Twelve Dockerfiles use five different arrangements of download/build
   split, cache mounts and credential placement, so each repository has to be reasoned about
   on its own.

The card's goal is one simple, efficient build method for present and future repositories
building images with Go binaries, including private modules.

## Domain Model and Terminology

Existing terms, as used in `.github/workflows/README.md`:

- **Layer cache** - BuildKit's instruction cache, exported to and imported from the GitHub
  Actions cache with `cache-from/to: type=gha` (`mode=max` exports every stage's layers).
- **Cache mount** - a `RUN --mount=type=cache,target=...` directory. Lives in the builder's
  local storage, is not part of any layer, and is empty on a fresh runner unless something
  refills it. Not covered by the layer cache.
- **cache-dance** - `reproducible-containers/buildkit-cache-dance`, driven by the
  `cache-mounts` input: restores host directories with `actions/cache`, *injects* them into
  the cache mounts by running a throwaway `busybox` build per mount that `cp -R`s the
  directory in, and after the image build *extracts* them the same way so `actions/cache`
  can save them.
- **Module cache** - Go's downloaded modules, `$GOMODCACHE` (`/go/pkg/mod` in the official
  image). Several Dockerfiles mount `/go/pkg` instead, which contains it.
- **Build cache** - Go's compiled-package cache, `$GOCACHE` (`/root/.cache/go-build`).
- **Private module** - a module under `mkmba.nz/`, `oncall-optimizer.com/` or
  `github.com/mkmba-nz/`, fetched with git using the `github_token` BuildKit secret.
- **Emulated build** - `platforms: linux/arm64` on the default `ubuntu-latest` (x64) runner;
  BuildKit runs the arm64 stages under QEMU. **Native build** - the same with
  `runs-on: ubuntu-24.04-arm`.

The card's four **build methods**, numbered as it numbers them:

| # | Method | Where Go runs | Caches in play |
|---|--------|---------------|----------------|
| 1 | Build on the runner, `COPY` the binary into the image | runner (`setup-go`) | `setup-go`'s module + build cache (`actions/cache`) |
| 2 | Build in Docker, layer cache only | builder stage | layer cache; cache mounts cold every run |
| 3 | Build in Docker, cache-dance for the build cache only; the token goes to the build step, which fetches modules | builder stage | layer cache + persisted `go-build`; module cache cold every run |
| 4 | Build in Docker, full cache-dance | builder stage | layer cache + persisted `go-build` and module cache |

Method 3 is as the card defines it. A variant - modules kept in a `go mod download` layer,
token on that step, cache-dance for `go-build` only - is what
[tokenizer PR #13](https://github.com/mkmba-nz/tokenizer/pull/13) moves to; it is treated as
"3 (layer variant)" below. No glossary changes are proposed.

## Requirements

### Functional Requirements

- One documented build arrangement (workflow inputs + Dockerfile pattern) that every
  present and future image with a Go binary follows.
- Private modules fetch successfully on every run regardless of cache state - no step may
  depend on a cache mount having been populated by an earlier step.
- Credential handling preserves the properties established by
  [FZ-2425](https://fizzy.rohu-bass.ts.net/1/cards/2425),
  [FZ-2426](https://fizzy.rohu-bass.ts.net/1/cards/2426),
  [FZ-2433](https://fizzy.rohu-bass.ts.net/1/cards/2433) and
  [FZ-2434](https://fizzy.rohu-bass.ts.net/1/cards/2434): the token reaches only the `RUN`
  that needs it, never a layer, the layer cache, the image config or the workspace.
- `-buildvcs` stamping keeps working where an image relies on it (tokenizer fails its build
  without the stamp; claude-proxy shows it in its UI).

### Non-Functional Requirements

- Simplicity is the primary axis (card); performance and reproducibility outside GHA are
  the secondary axes that may justify complexity.
- A cold run must succeed with the same result as a warm run - caching is an optimisation,
  never a correctness dependency.
- Stay within the 10 GB per-repository Actions cache budget; `oncall` is at 9.3 GB today.

### Out of Scope

- Go binaries not shipped in images: the `github-token-broker` Lambda (built on the runner,
  zipped) and the Rails/Ruby stages of `front`. Listed in the inventory for completeness.
- The `goproxy-prod` Athens proxy: a LAN service for developer machines, not reachable from
  Actions runners, so it does not change how CI fetches private modules.
- Retiring the transitional `GITHUB_PAT` build argument (tracked by FZ-2425's migration
  status in the README).

## Current System Analysis

### Shared infrastructure

- [`.github/workflows/build-and-push.yml`](../../../.github/workflows/build-and-push.yml) -
  the image-build path for every image but traefik (18 call sites in the organisation: the
  14 Go-image sites below plus grafana, prom, oncall's prom and fizzy). Always
  `cache-from/to: type=gha,mode=max,scope=<repo-name>`; always `push: true`; mounts the
  `github_token` secret (minted when `private-modules: true`, else `GIT_PAT`); `cache-mounts`
  adds the four cache-dance steps. Introduced in commit `0679213` (2026-06-22) for "a Go
  build invalidated on every source change"; commit `c131a30` (2026-09-24) moved the host
  directories out of the workspace after they dirtied `-buildvcs` stamps.
- [`actions/setup-go`](../../../actions/setup-go/action.yml) - on-runner Go with
  `actions/setup-go` caching (keyed on `go.sum`) and private-module access via a minted
  token. Already the method-1 building block; used by every repo's test jobs.
- [`.github/workflows/README.md`](../../../.github/workflows/README.md) "Fetching private
  modules" - the documented Dockerfile pattern: secret on the `go mod download` `RUN` with
  shell-local `GIT_CONFIG_*`. The documented example has **no** cache mount on the download
  step, so modules land in the layer; tokenizer and claude-proxy diverged from it by adding
  one.

### Inventory of Go image builds

Every `Dockerfile` in the organisation with a `FROM golang` stage, and every workflow that
builds it. "Split" = separate `go mod download` and `go build` steps with only `go.mod`/
`go.sum` copied before the download. "Token on" = which `RUN` mounts the secret.

| Image (workflow) | Dockerfile | Runner → platform | `cache-mounts` | Split | Go cache mounts | Token on | Private deps |
|---|---|---|---|---|---|---|---|
| tokenizer dev (`deploy-dev.yml`) | `Dockerfile` | x64 → amd64 | none | yes | mod + build on both steps | download only | `mkmba.nz/x` |
| tokenizer staging (`deploy-staging.yml`) | same | arm native | go-build, go-mod | yes | same | download only | same |
| ssokenizer dev (`deploy-dev-ssokenizer.yml`) | `Dockerfile` | x64 → amd64 | none | no (`COPY . .`, one `RUN`) | mod + build | build | `mkmba.nz/x` |
| ssokenizer staging (`deploy-staging.yml`) | same | arm native | go-build, go-mod | no | same | build | same |
| claude-proxy (`deploy-claude-proxy.yml`) | `Dockerfile` | x64 → **arm64 emulated** | none | yes | mod on download; mod + build on build | **both** (repeated because of this very problem) | `mkmba.nz/x` |
| fizzy-gh-webhook (dev-infra `deploy-gh-webhook.yml`) | `fizzy/gh-webhook/Dockerfile` | x64 → **arm64 emulated** | none | partial (`COPY . .` precedes download) | none | download only; modules stay in the layer | `mkmba.nz/x` |
| ocopt-solver dev (oncall `deploy-dev-solver.yml`) | `Dockerfile.solver` | x64 → amd64 | apt ×2, go-build, go-mod | no | mod + build | build | `mkmba.nz/x`, `oncall-optimizer.com/{api,rotation,storage}` |
| ocopt/solver staging (`deploy-staging-solver.yml`) | same | arm native | same | no | same | build | same |
| backups dev (oncall `deploy-dev-backups.yml`, target `backups`) | `Dockerfile` litestream stage | x64 → amd64 | none | n/a (clones `mkmba-nz/litestream` at a pinned commit, then builds) | mod + build on build | the `git fetch` step | none in `go.mod`; the repo itself is private |
| ocopt/backups staging (`deploy-staging-backups.yml`) | same | arm native | go-build, go-mod | n/a | same | fetch | same |
| ocopt-front / ocopt/front / ocopt_staging/front (three oncall `*front.yml`) | `front/Dockerfile` litestream stage inside the Rails image | x64 → amd64; arm native ×2 | apt ×2 - **but `front/Dockerfile` has no apt cache mounts**, so the entries are 0 MB | n/a | mod + build on build | fetch | same as backups |
| corp/localoidc (oncall `deploy-localoidc.yml`) | `localoidc/Dockerfile` | x64 → **arm64 emulated** | none | no | mod + build | build | `mkmba.nz/x` |

Go builds that do not go through `build-and-push.yml`:

| Build | Method today | Notes |
|---|---|---|
| traefik (oncall `deploy-dev-traefik.yml`) | **method 1**: `setup-go`, `make binary-linux-amd64` on the runner, then `docker/build-push-action` on `infra/traefik/Dockerfile`, which only `COPY`s `dist/$TARGETPLATFORM/traefik` | The largest Go build in the org (3,141 `go.sum` lines, 89 k LOC); jobs 259-744 s. Last run 2026-05-01. |
| github-token-broker (cloud-infra) | `setup-go` on `ubuntu-24.04-arm`, `build.sh` → Lambda zip | 17-32 s build step. Not an image. |
| dev-test `Dockerfile.gotest` | built locally per its README with `--build-arg GITHUB_PAT` and `git config --global` insteadOf | Test harness; the pre-FZ-2425 pattern the README now forbids. |
| dev-test `gotest.yml` | `setup-go` then `go build` on the runner, no image | Test harness for private-module access from `setup-go`. |
| litestream `release.docker.yml` | upstream Docker Hub release workflow carried by the fork | Last two runs failed (2026-10-08); the org builds litestream from source inside oncall's images instead. |
| tsserve `Dockerfile` | no workflow builds it | Public repo, no private modules, builds with plain `docker build`. |

### Size of each Go build

Counted from the clones on 2026-10-13. "Cold native build" is `go build` with empty caches
on the architecture the image targets, read from the run logs cited in the next section.

| Module | Direct deps | `go.sum` lines | Non-test LOC | cgo | Cold native `go build` |
|---|---|---|---|---|---|
| tokenizer | 14 | 90 | 4,100 | no | 20-31 s (x64) |
| ssokenizer | 10 | 91 | 2,100 | no | ~40-48 s incl. download (x64) |
| claude-proxy | 4 | 237 | 11,100 | no (`CGO_ENABLED=0`, modernc sqlite) | not measured natively; 975 s emulated |
| fizzy/gh-webhook | 5 | 64 | 2,000 | no | not measured natively; 698 s emulated |
| oncall/localoidc | 8 | 95 | 1,200 | no | not measured natively; 474 s emulated |
| oncall/solver | 17 | 276 | 7,100 | no (`CGO_ENABLED=0`) | 76 s (x64) |
| litestream (fork, built inside backups/front) | 18 | 286 | 8,600 | **yes** (`mattn/go-sqlite3`, static link) | 115-141 s (arm native, incl. clone) |
| traefik | 124 | 3,141 | 88,600 | no | several minutes (on-runner, with `setup-go` cache) |

Module downloads are small: tokenizer's full `go mod download` takes 3-5 s natively and
claude-proxy's 50 s only because it is emulated. The cache-dance archives (module + build
cache, `actions/cache` entries as of 2026-10-13) are 66 MB for tokenizer, 71 MB for
ssokenizer, 310 MB for backups (litestream) and ~375 MB for solver (which also holds two apt
caches).

### Measured behaviour of the current caching

All figures are from the `build-and-push` jobs of the most recent 3-10 runs per workflow
(`gh api repos/mkmba-nz/<repo>/actions/runs/<id>/jobs`), 2026-06 to 2026-10. "Build step"
is `Build, tag, and push`; "dance" is the sum of restore + inject + extract + save. The
sample straddles commit `c131a30` (2026-09-24), whose `v2` key marker retired every
cache-dance entry, so "dance cold" rows include that reset as well as the 7-day unused rule.
Per-layer timings come from the logs of these runs: tokenizer dev
[37885683024](https://github.com/mkmba-nz/tokenizer/actions/runs/37885683024),
[36062107781](https://github.com/mkmba-nz/tokenizer/actions/runs/36062107781),
[34441274685](https://github.com/mkmba-nz/tokenizer/actions/runs/34441274685),
[38190249798](https://github.com/mkmba-nz/tokenizer/actions/runs/38190249798); tokenizer
staging [38369691442](https://github.com/mkmba-nz/tokenizer/actions/runs/38369691442) (warm),
[37885683118](https://github.com/mkmba-nz/tokenizer/actions/runs/37885683118) (cold); solver
dev [36651409400](https://github.com/mkmba-nz/oncall/actions/runs/36651409400) (cold), solver
staging [35792730254](https://github.com/mkmba-nz/oncall/actions/runs/35792730254) (warm);
claude-proxy [37698244928](https://github.com/mkmba-nz/claude-proxy/actions/runs/37698244928);
gh-webhook [35795192730](https://github.com/mkmba-nz/dev-infra/actions/runs/35795192730);
localoidc [38095326184](https://github.com/mkmba-nz/oncall/actions/runs/38095326184). Dev
workflows run on x64 runners and staging workflows on native arm runners, so a dev/staging
pair is not a like-for-like comparison; the dance comparison below is warm vs cold within
one workflow.

| Workflow | Job total | Build step | Dance overhead | Notes |
|---|---|---|---|---|
| tokenizer dev (no dance) | 58-77 s | 43-53 s | - | builder stage rebuilt from scratch in 4 of 4 sampled runs |
| tokenizer staging, dance warm | 82-83 s | 21-22 s | 37-38 s | `go build` 1.1 s |
| tokenizer staging, dance cold | 86-111 s | 41-55 s | ~25 s | |
| ssokenizer dev (no dance) | 70-86 s | 46-63 s | - | |
| ssokenizer staging, dance warm | 73-75 s | 15-16 s | ~35 s | |
| ssokenizer staging, dance cold | 87-97 s | 41-48 s | ~25 s | |
| solver dev, dance warm (4 mounts) | 232-404 s | 72-121 s | 120-217 s | extract alone 71-129 s |
| solver dev, dance cold | 244 s | 121 s | ~80 s | `go build` 76 s |
| solver staging, dance warm | 201-253 s | 54-87 s | 110-140 s | `go build` 3.3 s |
| solver staging, dance cold | 171 s | 86 s | ~60 s | |
| backups dev (no dance) | 172-235 s | 155-207 s | - | |
| backups staging, dance warm | 188 s (one run) | 34 s | ~130 s | |
| backups staging, dance cold | 219-245 s | 115-141 s | ~85 s | |
| front (dance, apt only, 0 MB) | 81-123 s | 54-83 s | 3-5 s | Rails image; litestream stage inside |
| claude-proxy (**emulated** arm64) | 648-1134 s | 615-1102 s | - | `go build` 975 s, download 50 s, cache export 57 s |
| fizzy-gh-webhook (**emulated**) | 525-829 s | 505-799 s | - | `go build` 698 s |
| corp/localoidc (**emulated**) | 434-695 s | 395-671 s | - | `go build` 474 s |

Findings:

1. **Emulation is the dominant cost in the organisation.** The three emulated images spend
   8-16 minutes in `go build` alone (474 s, 698 s, 975 s; jobs of 7-19 minutes). The builder
   in those runs reports only `linux/amd64*` platforms, so the arm64 stage runs under QEMU.
   None of the three has a native measurement; the inference is from the measured images
   of the same size class (1-11 k LOC, 64-276 `go.sum` lines), whose cold native `go build`
   is 20-76 s. The staging workflows in oncall, tokenizer and ssokenizer already use
   `runs-on: ubuntu-24.04-arm` from private repositories; GitHub's billing page lists arm64
   Linux at $0.005/min against $0.006/min for x64 (read 2026-10-13).
2. **A warm cache-dance saves little or nothing.** Within one workflow, warm vs cold: tokenizer
   staging 82-83 s vs 86-111 s, ssokenizer staging 73-75 s vs 87-97 s, solver dev 232-404 s
   vs 244 s, solver staging 201-253 s vs 171 s, backups staging 188 s vs 219-245 s. For
   solver the relationship inverts: every warm staging run in the sample is slower than the
   one cold run, and the warm dev runs bracket the cold one, because the dance overhead on a
   375 MB archive exceeds the 73 s of compile it saves. The warm
   run cuts `go build` from 20-76 s to 1-3 s, but the dance steps cost 35-40 s on a 66-71 MB
   archive and 110-220 s on a 310-375 MB one, which is where the saving goes. The no-dance
   dev jobs (58-86 s for tokenizer and ssokenizer on x64) are not longer than the warm
   staging jobs on arm. Extraction is the expensive half: a throwaway build per mount
   copying the whole directory out, then a tarball upload.
3. **The layer cache rarely helps the Go stage.** `golang:1.26-alpine` resolved to four
   different digests across the sampled tokenizer runs, invalidating the whole builder stage
   each time the tag moved; only same-day re-runs hit. The digest churn was measured on
   tokenizer only, but every Dockerfile in the inventory references a floating tag
   (`golang:1.26`, `golang:1.26-bookworm`, `golang:${GO_VERSION}-bookworm`, `golang:1.23`,
   plus `alpine:latest` / `debian:bookworm` runtimes) that moves on the same upstream
   cadence, so the finding applies to all of them. Exporting with `mode=max` costs
   10-57 s per run and is where most of the cache budget goes: oncall holds 264 BuildKit
   blobs (5.6 GB of its 9.3 GB; `gh api repos/mkmba-nz/oncall/actions/caches`, 2026-10-13).
4. **Entries are being lost.** oncall is at 9.3 GB of the 10 GB limit. Its solver cache-dance
   entries (375 MB each, one per commit SHA because the key rolls on `github.sha`; last saved
   2026-09-30) no longer exist on 2026-10-13 - gone to the 7-day unused rule, budget
   eviction, or both - so the next solver run pays the dance overhead for nothing. The front
   workflows persist 0 MB entries (22 of them) for apt mounts `front/Dockerfile` does not
   have.
5. **The latent failure is a Dockerfile property, not a workflow one.** Exposed today:
   tokenizer (download step writes into a mount; build step has no token) - the open
   [tokenizer PR #13](https://github.com/mkmba-nz/tokenizer/pull/13) for FZ-2522 removes the
   module mounts so modules live in the layer. Not exposed: gh-webhook (same fix already in
   place), claude-proxy (repeats the secret on the build step instead), ssokenizer, solver,
   localoidc (single `RUN` with the secret), backups/front (no private modules in `go.mod`).
6. **On-runner compile is comparable to a native Docker build.** tokenizer's `test` job in
   `deploy-dev.yml` (`setup-go` 13-17 s with its cache restored, then `go test ./...`, which
   compiles everything) takes 38-52 s cold and 3-4 s warm across the five most recent
   successful runs; the token-broker build step in cloud-infra's
   `deploy-github-token-broker.yml` on a native arm runner takes 17-32 s. The `setup-go` cache
   is saved by `actions/setup-go` only on a miss, so the build cache it holds goes stale
   between `go.sum` changes.

### Inconsistencies across the twelve Dockerfiles

Each of these is a decision a new repository currently has to make on its own:

- Module-cache mount target `/go/pkg` (tokenizer, ssokenizer, solver, localoidc, oncall and
  front litestream stages, litestream, gotest) vs `/go/pkg/mod` (claude-proxy) vs no module
  mount (gh-webhook, tsserve).
- `GOPRIVATE` as `ARG` (tokenizer, solver, localoidc, gh-webhook, gotest - overridable to
  empty by a build arg) vs `ENV` (claude-proxy, ssokenizer).
- Download/build split (tokenizer, claude-proxy, tsserve) vs `COPY . .` then one `RUN`
  (ssokenizer, solver, localoidc) vs `COPY . .` then download then build (gh-webhook, whose
  download layer is invalidated by every source change).
- `.git` in the build context for `-buildvcs` with four handling styles: whole tree
  (ssokenizer, solver, localoidc, claude-proxy), `COPY .git` after sources (tokenizer, which
  therefore always stamps "modified"), excluded (tsserve, front), stamp check enforced
  (tokenizer only).
- Base image: `golang:1.26-alpine` (tokenizer and ssokenizer with `apk add git`, tsserve
  without) vs `golang:1.26` / `-bookworm` (the rest); runtime `alpine`, `distroless`,
  `debian`.
- `safe.directory` handling: a wildcard `/...` (solver, localoidc, gotest), the exact path
  (tokenizer, ssokenizer, claude-proxy), none (gh-webhook).

## Approach Options

Ranked. For each, "decided by" separates what the measurements settle (system fact), what
is a preference the user owns (product intent), and what the design phase chooses.

### A. Method 2 on native runners - build in Docker, layer cache only, no cache-dance (recommended)

The README's documented "Fetching private modules" pattern as it stands (modules in the
`go mod download` layer, secret on that one `RUN`, no module mount) with no `cache-mounts`,
and the native arm runner the staging workflows already use wherever the image is arm64.

- Measured: the no-dance builds already in this shape (tokenizer and ssokenizer dev,
  gh-webhook apart from emulation) take 45-86 s per job; backups, with its cgo litestream
  build, 172-235 s. The three emulated images have no native measurement (Finding 1); by
  size class they would land in that range, down from 7-19 minute jobs. Modules in the
  layer make the cold-mount failure impossible: the download layer is either rebuilt with
  the token or served complete from the cache.
- Trade-offs: every build pays the full compile when the base digest moves (frequently),
  i.e. the layer cache is mostly a module-download cache. The module layer, including
  private source, is exported to the Actions cache under `mode=max` - the same exposure as
  the cache-dance archives today; FZ-2522's triage report records it as a risk "a reviewer
  should accept knowingly" and the card's owner approved that plan. Keeps
  `docker build --secret ...` reproducible locally and on fly, which ssokenizer,
  claude-proxy and gh-webhook document.
- Decided by: system fact (timings, failure mechanism); design phase (whether to pin base
  image digests, `mode=min` vs `max`, whether `cache-mounts` is removed from the workflow
  or merely undocumented for Go).

### B. Method 1 - compile on the runner, `COPY` the binary

`setup-go` (already mints the token and caches modules + build cache keyed on `go.sum`),
`CGO_ENABLED=0 GOARCH=... go build`, then a Dockerfile that only copies the binary - as
oncall's traefik build does today.

- Measured: comparable to A when caches are cold (tokenizer compile ~20-50 s on the runner);
  faster than A when warm because `setup-go`'s build cache survives between runs at no
  extra step cost, unlike cache-dance. Avoids QEMU entirely by cross-compiling.
- Trade-offs: the Dockerfile no longer builds on its own, so local and fly builds need a
  second documented path; the card lists this reproducibility loss as the argument against
  it. The litestream stage needs cgo, so backups/front would need a cross C toolchain or a
  native-arch runner. Two toolchain versions to keep aligned (`setup-go` reads the version
  from the Dockerfile today, which would no longer have one). `-buildvcs` stamping moves to
  the runner checkout, which is clean, so stamps improve.
- Decided by: product intent (is standalone `docker build` a requirement?); system fact
  (cgo constraint); design phase (how `build-and-push.yml` would accept a prebuilt
  artefact).

### C. Method 3 - cache-dance for the build cache only

As the card defines it (token on the build step, which fetches modules every run), or the
layer variant tokenizer PR #13 moves to (modules in the download layer, token there).

- Measured indirectly: no run uses either form yet. The dance cost is dominated by the fixed
  per-mount inject/extract builds and the upload rather than archive size (tokenizer's 66 MB
  archive still costs ~35 s), so dropping the module half of the archive does not change the
  picture: the warm-run saving of 20-70 s of `go build` is roughly cancelled. The card's form
  also re-downloads modules on every run (3-5 s natively) and puts the credential on the
  step that compiles third-party code.
- Trade-offs: keeps two caching mechanisms to explain, the per-SHA key accumulation and the
  eviction exposure. Only plausibly worthwhile for a build whose compile dwarfs ~40 s of
  fixed overhead - none of the current images; traefik would qualify if it moved into Docker.
- Decided by: system fact (overhead floor); design phase (whether to keep it available as a
  documented exception for very large builds).

### D. Method 4 - full cache-dance (status quo where enabled)

- Measured: a warm cache saves 0-55 s of a 75-400 s job on the four images using it
  (Finding 2); the module mount is what creates the cold-mount failure in the first place;
  its archives are the entries oncall has lost.
- Trade-offs: no case for it at current build sizes.
- Decided by: system fact.

### Recommendation

Option A: build in Docker with the layer cache only, modules in the `go mod download`
layer, the secret on that one `RUN`, and native arm runners for arm64 images; retire
cache-dance for Go builds. Confidence **high** that cache-dance is not earning its cost
(measured on every image using it); **high** that emulation, not caching, is the large
cost, though the native gain for the three emulated images is inferred from same-size
modules rather than measured; **medium** on A over B, which rests on the reproducibility
preference the card raises (scope question 2) rather than on a measured difference. The
card's ordering (1 > 2 > 3 > 4 by simplicity) is agreed for 3 and 4; between 1 and 2, the
measurements do not separate them and 2 has one documented build path instead of two.

## Integration Points

- **FZ-2522 / tokenizer PR #13** (open): moves tokenizer to "modules in the layer" and
  trims its `cache-mounts` to the build cache (option C). Compatible with option A except
  for the remaining `cache-mounts` entry; the two cards should not both edit
  `deploy-staging.yml` without coordination.
- **FZ-2425 migration status** in the README: `GITHUB_PAT` build argument still passed to
  every build; a single documented Dockerfile pattern is the natural point to finish that
  migration, but it is a separate card.
- **`actions/setup-go`**: option B would reuse it unchanged; option A does not touch it.
  Its Dockerfile version extraction assumes a `FROM golang:` line exists.
- **oncall `deploy-dev-traefik.yml`**: the one existing method-1 build; whichever option
  wins, the standard should say whether traefik is an exception or migrates.
- **Actions cache budget**: any change that drops cache-dance archives or `mode=max`
  exports relieves oncall's 9.3 GB; the `v2` key marker from `c131a30` shows prior entries
  can only be actively retired by a key change, and otherwise persist until the 7-day
  unused rule or budget eviction removes them.

## Complexity and Risk Areas

- `build-and-push.yml` hardcodes `push: true`, and the card forbids pushes during
  experiments, so no live experiment can run through the shared workflow as it is.
- Base image tag drift makes the layer cache miss most runs, so the layer cache's value for
  the Go stage depends on how the base image reference is managed.
- `mode=max` exports the module layer (private source) to the Actions cache; the export
  mode and what it exposes are coupled to how much of the builder stage is reused.
- Removing or deprecating `cache-mounts` touches eight oncall/tokenizer/ssokenizer
  workflows across three repositories plus the `.dockerignore` comment in ssokenizer that
  references it; the `apt` mounts in solver are a non-Go use of the same input.
- The litestream stage is cgo + static linking; its build time (2 min native) is the
  largest remaining compile and the only one where method 3 might become attractive.
- `-buildvcs` behaviour differs per Dockerfile today (tokenizer always stamps modified; the
  workflow warns on a dirty context); a single pattern has to choose one behaviour.
- Emulated builds have been the norm for claude-proxy, gh-webhook and localoidc; switching
  those three workflows to arm runners changes their billing SKU and runner availability
  profile.
- The on-runner `setup-go` build cache (option B) is only re-saved when `go.sum` changes,
  so its benefit decays between dependency updates.

## Scope Questions

1. **Live experiments.** The card asked for GHA experiments on a representative sample. The
   recorded runs already give methods 2 and 4 directly (tokenizer, ssokenizer, solver,
   backups, warm and cold) and method 1 by proxy (test jobs, token-broker, traefik); method 3
   is only inferred from the dance overhead floor. Working assumption: this data is
   sufficient to design from, and a live experiment is only needed if the design wants a
   native-arm measurement of the three emulated images before committing. If wanted, the
   plan would be: a temporary branch in tokenizer (small, private modules), oncall solver
   (medium) and claude-proxy (emulated today), each running an inline
   `docker/build-push-action` job with `push: false` in the four arrangements, twice (cold
   then warm), requested as tasks from those projects.
2. **Reproducibility outside GHA.** Is a standalone `docker build --secret ...` from a clean
   checkout a requirement of the standard? Working assumption: yes, per the card's own
   argument and the three repositories that document it; this is what ranks A above B.
3. **Scope of the standard.** Does it cover only images, or also the on-runner builds
   (traefik, the Lambda)? Working assumption: images only; the others are noted as
   exceptions.
4. **Private source in the Actions cache.** Option A keeps exporting the module layer with
   `mode=max`, as cache-dance archives and the current `mode=max` already do. Working
   assumption: acceptable, as the approved FZ-2522 plan records; the design phase should
   confirm it when choosing the export mode.

## References

- buildkit-cache-dance: https://github.com/reproducible-containers/buildkit-cache-dance
- GitHub Actions cache limits and eviction: https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/caching-dependencies-to-speed-up-workflows
- Docker `type=gha` cache backend: https://docs.docker.com/build/cache/backends/gha/
- Dockerfile `RUN --mount=type=cache` / `type=secret`: https://docs.docker.com/reference/dockerfile/
- GitHub Actions runner pricing (x64 vs arm64): https://docs.github.com/en/billing/managing-billing-for-your-products/about-billing-for-github-actions
- `actions/setup-go` caching: https://github.com/actions/setup-go
