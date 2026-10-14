# Go Image Build Standard - PRD

Tracking Issue: [FZ-2527](https://fizzy.rohu-bass.ts.net/1/cards/2527)

Research: [RESEARCH.md](RESEARCH.md) (terminology and measurements are carried forward from
there and not restated).

## Introduction

Twelve Go image builds across the organisation use five different arrangements of download/build
split, cache mounts and credential placement, and four images persist their Go cache mounts
through cache-dance. The research measured that cache-dance saves 0-55 s of a 75-400 s job at
best, that QEMU emulation (not caching) is the large cost on three images, and that putting
modules in a cache mount is what made tokenizer's build fail
([FZ-2522](https://fizzy.rohu-bass.ts.net/1/cards/2522)).

This feature adopts research option A - **build in Docker with the layer cache only** - as the one
build method for every present and future MKMBA image holding a Go binary: modules in the
`go mod download` layer, the private-module secret on that one `RUN`, native builds for arm64
images, and cache-dance retired. It also settles the option's open design choices: the builder
stage is pinned by digest and the export mode stays `mode=max`. The approved scope decisions are
1A (design from the recorded runs, no live experiments), 2A (a standalone `docker build --secret`
from a clean checkout is a requirement), 3A (container images; traefik is an exception, the Lambda
follows the on-runner shape) and 4A (the `go mod download` layer may be exported to the Actions
cache).

## Goals

1. One documented Dockerfile pattern and one `build-and-push.yml` usage for every Go image, so a
   new repository copies the pattern rather than making the six decisions the research lists.
2. A build's result does not depend on cache state: no step depends on a cache mount an earlier
   step populated, and a run whose `go mod download` layer is served from the layer cache still
   has every module at `go build`.
3. The three emulated images (claude-proxy, fizzy-gh-webhook, corp/localoidc) build natively:
   job time from 7-19 minutes to about 3 minutes.
4. No Go build runs cache-dance; the `cache-mounts` input is removed from `build-and-push.yml`.
5. Credential properties from FZ-2425/2426/2433/2434 are preserved: the token reaches only the
   `RUN` that fetches modules and never a layer, the layer cache, the image config or the workspace.
6. The same Dockerfile builds unchanged on GitHub Actions, locally and on fly.

## User Stories

Stories US-003 to US-007 change other repositories. The implementation agent requests each as a
`change` task from that project (agent-tasks skill) once US-001 and US-002 have merged, with a
prompt containing that story's acceptance criteria and a link to the merged README standard; the
story is complete when the task closes with success naming a merged pull request and the
implementation agent has read the runs named in Testing strategy. The five tasks are independent
of each other and may run concurrently. Every migration also removes any `ARG GITHUB_PAT` the
Dockerfile still declares (the workflow passes that build argument on every build, and a declared
`ARG` records it in exported layer history); each migration story carries this as its own
criterion so it is checked at the pull request that changes the Dockerfile.

#### US-001: `build-and-push.yml` picks the native runner for an arm64 build

**Description:** As a workflow caller, I want an arm64 image to build on an arm runner without
remembering a second input, so that no future repository falls into a 15-minute emulated build.

**Acceptance Criteria:**
- [ ] A caller setting `platforms: linux/arm64` and no `runs-on` runs the job on
      `ubuntu-24.04-arm`; a caller setting neither still runs on `ubuntu-latest`; an explicit
      `runs-on` is always honoured. Any other `platforms` value (multi-platform, amd64) keeps
      today's default. Trap: a `workflow_call` input default cannot reference another input, so
      `runs-on` needs an empty default and the job's `runs-on` expression does the selection.
- [ ] When `platforms` is exactly `linux/arm64` on an `X64` runner or exactly `linux/amd64` on an
      `ARM64` runner (`runner.arch`), the job emits a `::warning::` naming the emulation before
      the build step and still builds (an explicit override is deliberate). Any other
      `platforms` value produces no warning.
- [ ] The README input table and the `runs-on` description say the default is derived from
      `platforms`, and the README sentence listing which inputs declare a default is corrected.
- [ ] The three callers that build `linux/arm64` without `runs-on` today (claude-proxy,
      dev-infra gh-webhook, oncall localoidc) move to native builds at their next run once this
      merges, without a change in their repositories. The implementation agent states this effect
      in the PR description so the human reviewer approves it knowingly.
- [ ] The implementation agent reads the first arm64 consumer run after merge and confirms its
      "Set up job" log names an ARM64 runner and the buildx log reports a native `linux/arm64`
      worker.

#### US-002: Document the Go image build standard

**Description:** As a maintainer of any MKMBA repository shipping a Go binary in an image, I want
one documented build pattern so that I can copy it rather than design my own.

**Acceptance Criteria:**
- [ ] `.github/workflows/README.md` gains a "Building a Go image" section under `build-and-push.yml`
      that presents one complete Dockerfile template and the caller's workflow inputs. The existing
      "Fetching private modules" section keeps the credential rules and its example `RUN` becomes a
      reference to the template rather than a second copy.
- [ ] The template builder stage, in order: a `golang` image pinned by digest with the full tag
      kept for readability (`FROM golang:<major.minor.patch>-<variant>@sha256:... AS build`); a
      `git` install where the variant ships none (`apk add --no-cache git` on alpine - both the
      private-module fetch and the VCS stamp need it); `ENV GOPRIVATE=mkmba.nz,oncall-optimizer.com`
      (the module path prefixes, matching `actions/setup-go`'s default - not the `github.com/...`
      URL the current README example shows, which the vanity paths never match); `COPY go.mod go.sum`
      then `go mod download` on the one `RUN` that mounts the secret with `required=true`, with
      **no cache mount** on that `RUN`; then `COPY . .`, `safe.directory` for the work directory,
      and `go build` with `CGO_ENABLED=0` and `-buildvcs=true` on a `RUN` whose only mount is a
      cache mount at `/root/.cache/go-build` (the build cache). The template declares no
      `ARG GITHUB_PAT` and no `ARG GOPRIVATE`. A repository whose `go.mod` names no private
      module omits the secret mount and `GOPRIVATE`; everything else is identical.
- [ ] The template runtime stage is a version-tagged base (`alpine:3.22`, `distroless/...`), not a
      digest, so OS fixes reach shipped images on every rebuild; the section states the pinning
      rationale from Technical Considerations ("Base image pinning") in two or three sentences,
      including who bumps the pin and when.
- [ ] The section states why each of these is load-bearing, each as one sentence: modules in the
      `go mod download` layer (the cold-mount failure is impossible); no module cache mount on
      the build `RUN` (a mount shadows the layer's modules and forces a re-download); the build
      cache mount is cold every run and nothing depends on it (it exists to keep compiled objects
      out of the layer that `mode=max` exports); the official `golang` image sets
      `GOTOOLCHAIN=local`, so a `go.mod` newer than the pinned image fails at the download step
      instead of fetching a toolchain that silently defeats the pin - do not unset it; `.dockerignore`
      must keep `.git` and must exclude no tracked file, because `-buildvcs` stamps `(modified)`
      for every tracked file missing from the copied tree; `safe.directory` carries no credential
      and keeps git working when the tree's owner differs from the build user. Cache mounts for
      non-Go steps (solver's apt mounts) are allowed on the same terms: cold every run.
- [ ] A build whose context lacks the repository root (a monorepo subdirectory) is told to set
      `-buildvcs=false` explicitly, since the stamp is impossible there; `-buildvcs=true` is the
      default of the template.
- [ ] The caller's side is stated as: `private-modules: true` when `go.mod` names a private
      module; `platforms: linux/arm64` for an arm64 image, with the runner chosen by the workflow
      (US-001); no `cache-mounts`.
- [ ] Exceptions are listed with their reason: cgo builds (the litestream stages in oncall) use a
      Debian-variant builder, static `-extldflags`, and their builder pin must also move on OS
      rebuilds because libc is linked into the binary; traefik keeps its on-runner upstream
      Makefile build; non-image Go builds (the github-token-broker Lambda in cloud-infra) use
      `actions/setup-go` on a native runner with `CGO_ENABLED=0`, `-trimpath` and an explicit
      `GOARCH`, which cloud-infra's `github-token-broker/build.sh` already does (verified during
      design; not in RESEARCH.md).
- [ ] A "Verifying a build" paragraph says what a correct run looks like (the builder stage's
      `FROM` resolves to the pinned digest, `go build` prints no `go: downloading` lines, the job
      has no cache-mount steps) and repeats the "Outside CI" local build command as the
      clean-checkout check.
- [ ] The `cache-mounts` input documentation stays untouched in this story (US-008 removes it);
      the new section says the input is not used for Go images.

#### US-003: tokenizer follows the standard

**Description:** As the tokenizer maintainer, I want its Dockerfile and both deploy workflows on
the standard so that the FZ-2522 failure class cannot recur.

**Acceptance Criteria:**
- [ ] The implementation agent requests this task only after
      [tokenizer PR #13](https://github.com/mkmba-nz/tokenizer/pull/13) (FZ-2522) is merged or
      closed, and names it in the task prompt so the task invocation builds on it rather than
      conflicting with it.
- [ ] `Dockerfile` matches the US-002 template: digest-pinned builder, modules in the
      `go mod download` layer, secret on the download `RUN` only, build cache mount only on the
      build `RUN`, `COPY . .` replacing the selective source copies. The existing `-ldflags`
      (`main.Version` from `VERSION`, `main.FilteredHeaders`) and the post-build VCS stamp check
      stay.
- [ ] `.dockerignore` excludes no tracked file (checked against `git ls-files`), so the stamp is
      clean - today's selective copy always stamped `(modified)`.
- [ ] No stage of any Dockerfile this story changes declares `ARG GITHUB_PAT` after the change.
- [ ] `deploy-staging.yml` no longer passes `cache-mounts`; `deploy-dev.yml` already sets
      `private-modules: true` and needs no change.
- [ ] The runs named in Testing strategy succeed for both workflows, the binary's stamp reports the
      commit without `(modified)`, and the staging job is no slower than its cold cache-dance
      runs in RESEARCH.md (86-111 s).
- [ ] The implementation agent runs the README's local build command from a clean checkout of the
      merged commit where it has a Docker daemon and a read-only PAT; otherwise it asks the project owner on
      the card to run it. Either way the result is recorded on the card. tokenizer is the one
      representative check for Goal 6 (private modules, VCS stamp check).

#### US-004: ssokenizer follows the standard

**Description:** As the ssokenizer maintainer, I want its Dockerfile and both deploy workflows on
the standard.

**Acceptance Criteria:**
- [ ] `Dockerfile` matches the US-002 template: the single `COPY . .` + one `RUN` arrangement is
      split into download and build steps, the secret moves from the build `RUN` to the download
      `RUN`, the module cache mount is removed, the builder is pinned by digest.
- [ ] No stage of any Dockerfile this story changes declares `ARG GITHUB_PAT` after the change.
- [ ] `deploy-staging.yml` no longer passes `cache-mounts`; the `.dockerignore` comment that
      references cache-mounts is removed or corrected.
- [ ] The runs named in Testing strategy succeed for both workflows, and the staging job is no
      slower than its cold cache-dance runs in RESEARCH.md (87-97 s).

#### US-005: claude-proxy follows the standard

**Description:** As the claude-proxy maintainer, I want the image built natively on arm so that a
deploy takes minutes rather than a quarter of an hour.

**Acceptance Criteria:**
- [ ] `Dockerfile` matches the US-002 template: the module cache mounts on both `RUN`s are
      removed, the repeated secret and `GIT_CONFIG_*` on the build `RUN` are removed (the comment
      explaining why they were repeated goes with them), the builder is pinned by digest. The
      existing `-trimpath -buildvcs=true -ldflags="-s -w"` and distroless runtime stage stay.
- [ ] No stage of any Dockerfile this story changes declares `ARG GITHUB_PAT` after the change.
- [ ] `deploy-claude-proxy.yml` passes no `cache-mounts` and no `runs-on` (US-001 selects the
      runner).
- [ ] The runs named in Testing strategy succeed natively; the UI still shows the VCS revision;
      the job total is about 3 minutes (the native figure is inferred, not measured - a slower
      native run is recorded on the card as a measurement, not a failure).

#### US-006: fizzy-gh-webhook (dev-infra) follows the standard

**Description:** As the dev-infra maintainer, I want gh-webhook built natively on arm with a
`go mod download` layer that survives source changes.

**Acceptance Criteria:**
- [ ] `fizzy/gh-webhook/Dockerfile` matches the US-002 template with the context staying the
      repository root (`context: .`, which `-buildvcs` needs): `COPY fizzy/gh-webhook/go.mod
      fizzy/gh-webhook/go.sum` and `go mod download` precede `COPY . .` (today the source copy
      comes first, so every commit invalidates the download layer); a build cache mount is added
      to the build `RUN` (today compiled objects land in the exported layer); the builder is
      pinned by digest, replacing `ARG GO_VERSION`; `ARG GOPRIVATE` becomes the template's `ENV`;
      `-buildvcs=true` is explicit and `safe.directory` is set.
- [ ] No stage of any Dockerfile this story changes declares `ARG GITHUB_PAT` after the change.
- [ ] `deploy-gh-webhook.yml` passes no `cache-mounts` and no `runs-on` (US-001 selects the
      runner).
- [ ] The runs named in Testing strategy succeed natively and the job total is about 3 minutes
      (same caveat as US-005).

#### US-007: oncall images follow the standard

**Description:** As the oncall maintainer, I want solver, backups, front and localoidc built one
way, with the repository's Actions cache no longer filled by cache-dance archives.

**Acceptance Criteria:**
- [ ] `Dockerfile.solver` and `localoidc/Dockerfile` match the US-002 template (download/build
      split, secret on the download `RUN`, module cache mounts removed, digest-pinned builder).
      Solver's apt cache mounts may stay, cold every run.
- [ ] No stage of any Dockerfile this story changes declares `ARG GITHUB_PAT` after the change.
- [ ] The litestream stages in `Dockerfile` and `front/Dockerfile` follow the cgo exception: a
      digest-pinned Debian-variant `golang` builder; the pinned-commit `git fetch` `RUN` keeps the
      secret mount (the fork is private); the build `RUN` keeps only a build cache mount. Trap:
      there is no `go.mod` in the context before the clone, so these stages have no separate
      download layer; modules download inside the build `RUN`, which needs no credential because
      the fork's `go.mod` names no private module. No other stage of `Dockerfile` or
      `front/Dockerfile` changes.
- [ ] No oncall deploy workflow passes `cache-mounts` - six do today (solver dev and staging,
      backups staging, the three front workflows); the task invocation searches the repository
      rather than trusting that list. `deploy-localoidc.yml` passes no `runs-on` (US-001 selects
      the runner).
- [ ] The runs named in Testing strategy succeed for every affected workflow; localoidc builds
      natively in about 3 minutes (same caveat as US-005); the solver and backups staging jobs are
      no slower than their cold cache-dance runs in RESEARCH.md (171 s and 219-245 s).

#### US-008: Retire `cache-mounts` from `build-and-push.yml`

**Description:** As the workflow maintainer, I want cache-dance gone so that the build path has one
caching mechanism to explain.

**Acceptance Criteria:**
- [ ] The implementation agent confirms, by searching every workflow in the `mkmba-nz`
      organisation that calls `build-and-push.yml`, that no caller passes `cache-mounts` before
      opening the PR, and records the search in the PR description. Trap: a caller passing an
      input the called workflow no longer declares fails at workflow validation, so the
      implementation agent opens this PR only after the US-003, US-004 and US-007 tasks have
      closed with success.
- [ ] The `cache-mounts` input, its three conditional steps (resolve, restore, inject) and the
      comments that exist only for them are removed from `build-and-push.yml`; the checkout,
      secret, dirt-warning and build steps are unchanged.
- [ ] The README loses the `cache-mounts` row, the "keeps nothing in the workspace" paragraph and
      every other reference to cache-dance; the "Building a Go image" section's note that the
      input is not used for Go images goes too.
- [ ] The implementation agent reads the first consumer run after merge and confirms the build
      step still imports and exports the layer cache (the `type=gha` lines are untouched).

## Non-Goals

- Changing the layer-cache export mode (`mode=max` stays; see Open Questions).
- Finishing the FZ-2425 migration on the workflow side (the `GITHUB_PAT` build-argument line and
  the `GIT_PAT` Buildx token stay), or any change to `actions/setup-go`.
- Moving traefik off its on-runner build, or any change to the github-token-broker Lambda, the
  litestream fork's own release workflow, dev-test's harness, or tsserve.
- Live GHA experiments (1A): the design rests on the recorded runs in RESEARCH.md.
- Automated digest bumps (Dependabot `docker` updates): see "Base image pinning".
- Purging existing Actions cache entries; cache-dance archives age out under the 7-day rule.

## Design Considerations

No user interface. The one human-facing artefact is the README section (US-002), which follows
the README's existing voice: a complete copyable example first, then one paragraph per
load-bearing choice saying why it matters.

## Technical Considerations

### Phasing

The implementation agent lands US-001 and US-002 first, in this repository, as two PRs (the
workflow change, then the docs that describe it). US-003 to US-007 are `change` tasks in five
projects, requested concurrently once both have merged; the implementation agent requests US-003
only once tokenizer PR #13 is merged or closed. The implementation agent opens US-008 only after
the US-003, US-004 and US-007 tasks have closed with success - those are the three repositories
that pass `cache-mounts`.

### Base image pinning

Answers the card question on tag drift. The research found `golang:1.26-alpine` resolving to four
digests across the sampled runs, invalidating the builder stage almost every run. The gain from
pinning is modest - the base pull and the module download, since the build `RUN` is invalidated by
every commit regardless - and the main value is determinism: the same commit builds against the
same toolchain in dev and staging, and the image-rebuild convention already requires images to
pin their base.

- **Builder: pin by digest, not by patch tag.** Docker official image patch tags (`1.26.0`) are
  rebuilt for OS updates until the next patch supersedes them, so a `-0` tag is only frozen after
  `-1` exists; a digest is frozen from the start and is what BuildKit keys the cache on anyway.
  For a `CGO_ENABLED=0` binary nothing in the builder's OS layer reaches the output, so OS
  rebuilds of the builder can be ignored - the card's intuition holds for that case.
- **What it overlooks: Go patch releases.** Standard-library security fixes (net/http, crypto)
  are compiled into every binary, so the builder pin must move on Go patch releases, not stay at
  the first revision of a minor. Two paths move it: Inspector flags the stdlib CVE in the binary
  and the vulnerability-manager's image-rebuild task "updates the base image version", which for a
  pinned builder means tag and digest together (Open Question 2 confirms this path); and a
  dependency update that raises `go.mod`'s `go` directive past the pinned patch fails the image
  build at the download step (`GOTOOLCHAIN=local`), which the test job on `check-latest: true`
  would not catch, so the task making that update bumps the pin in the same PR. Staleness
  rebuilds move it between those. Dependabot digest updates are not adopted: a PR per upstream
  rebuild is the churn the pin exists to remove.
- **cgo exception.** The litestream stages link libc statically, so builder OS content does reach
  their binaries; their pin moves on OS rebuilds as well (same path - a finding against the
  backups or front image).
- **Runtime stage: version tag, not digest.** The final stage is one `COPY`, so its invalidation is
  free, and a floating tag is how OS fixes reach shipped images on every rebuild.
- `actions/setup-go`'s version extraction (`^FROM golang:([0-9]+\.[0-9]+)`) reads `1.26` from the
  pinned form, so test jobs keep tracking the Dockerfile's minor; they run the latest patch
  (`check-latest: true`) while the image runs the pinned one, which is accepted.

### Export mode and the build cache

`cache-to: type=gha,mode=max` is unchanged (4A: the `go mod download` layer, private source
included, may be exported). Because the mode exports every builder layer, the build `RUN` keeps
one cache mount for the build cache (`/root/.cache/go-build`) purely to keep compiled objects
(tens to hundreds of MB per run) out of the exported layer; the mount is cold every run and
nothing depends on it. This is the only cache mount the standard allows on a Go step.

### Testing strategy

This repository has no automated test of `build-and-push.yml` (`self-test-actions.yml` covers the
actions only), and the workflow always pushes, so there is no dry run. For each migrated image
the implementation agent reads two runs from the logs: the **first run after merge**, whose
builder stage rebuilds from scratch because its instructions changed (the cold case), and the
**next run whose change touches neither `go.mod` nor `go.sum`**, where the `go mod download`
layer is reported `CACHED` and `go build` must still print no `go: downloading` lines (the warm
layer, formerly cold mount, case that broke tokenizer). Both must succeed; the second may be the
next routine deploy rather than a run made for the purpose.

### Integration points

- tokenizer PR #13 (FZ-2522): US-003 builds on it.
- The image-rebuild skill (agent-manager, outside this repository) says "update the base image
  version"; with a digest pin that means tag and digest. Open Question 2.
- oncall's Actions cache (9.3 GB of 10 GB): cache-dance archives (375 MB each for solver) stop
  being written at US-007; nothing is purged.

## Success Metrics

- claude-proxy, fizzy-gh-webhook and corp/localoidc jobs about 3 minutes (from 7-19).
- No Go image job runs a cache-dance step; `cache-mounts` absent from the organisation.
- tokenizer, ssokenizer, solver and backups staging jobs no slower than their cold runs in
  RESEARCH.md.
- One Dockerfile arrangement across the twelve builds, with the documented exceptions.
- The clean-checkout local build of tokenizer (US-003) succeeds.

## Open Questions

1. `mode=min` would stop exporting the builder stage (no private source in the cache, 10-57 s less
   export per run) at the cost of re-running the whole builder stage every run - base pull, git
   install and module download - which the research did not measure. Not changed here because
   the setting serves 18 call sites including non-Go images; a follow-up card can measure it on
   one migrated image.
2. Is the vulnerability-manager's image-rebuild path the intended mechanism for moving the builder
   pin on Go patch releases, and does the image-rebuild skill (outside this repository) need a
   sentence saying a digest pin is bumped as tag and digest together? If so, the project owner opens that
   card in the agent-manager project; nothing in this PRD depends on it.
