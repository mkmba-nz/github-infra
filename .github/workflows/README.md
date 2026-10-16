# `review-agent.yml`

Reusable workflow that runs Claude Code over a pull request and posts a
review (approve / request-changes).

A single workflow with two modes:

| Mode (input)   | Typical trigger (in consumer) | What it does                                                                                  |
|----------------|-------------------------------|-----------------------------------------------------------------------------------------------|
| `auto` (default) | `pull_request`              | Reviews on open/synchronize/etc. Skips silently if the PR is merged or closed, if the head commit is already approved, or if the diff was already reviewed. |
| `requested`    | `pull_request_review_comment` | Re-reviews on demand. Skips dedupe, always runs, embeds the triggering comment body as reviewer special instructions. |

Both modes share the same concurrency group (`pr-review-<number>`) with
`cancel-in-progress: true`, so:

- a new commit during an in-flight review or re-review supersedes it;
- a rapid second @-mention cancels the first re-review (the latest
  reviewer instructions win).

## Required variables

These are provisioned as org vars in mkmba-nz, so no action is required when used within that org, 
but if this is being reused outside the org, then you must supply as `vars`:

| Variable                     | Purpose                                                  |
|------------------------------|----------------------------------------------------------|
| `TS_OAUTH_CLIENT_ID`         | Tailscale OAuth client - used to reach the llmux gateway |
| `TS_AUDIENCE`                | Tailscale OIDC audience                                  |
| `REVIEW_AGENT_APP_ID`        | GitHub App ID for the bot that posts the review          |
| `MKMBA_ANTHROPIC_BASE_URL`   | URL of the claude proxy to use                           |

## Required secrets

Pass via `secrets: inherit` if the names match in the consumer repo,
otherwise enumerate.

| Secret                       | Purpose                                                  |
|------------------------------|----------------------------------------------------------|
| `REVIEW_AGENT_PRIVATE_KEY`   | GitHub App private key matching REVIEW_AGENT_APP_ID      |

## Conventions

- **Bot login.** The review bot is the org-wide `mkmba-review-agent` App,
  hardcoded in the workflow. GraphQL `author.login` omits the `[bot]`
  suffix for App actors, so the workflow finds its own prior comments by
  the bare slug (`BOT_SLUG`) and passes the `[bot]`-suffixed login to the
  review action as `bot_name`. Reviews are matched by state, not by login:
  who approved does not affect the decision.
- **Approval is judged against the head SHA.** An approval counts only
  while GitHub still reports it (not dismissed) *and* it was submitted
  against the commit being reviewed. A new head is therefore never treated
  as approved, whether or not the repo dismisses stale approvals — though
  a push that leaves the diff unchanged can still be skipped by the
  diff-hash dedupe below.
- **Gateway ready** Sets up a Tailscale connection and sets environment
  variables in anticipation of the caller overriding the base URL to route
  reviews through a custom LLM gateway before Anthropic.  

## Inputs

| Input                     | Required | Default                       | Description                                                                                  |
|---------------------------|----------|-------------------------------|----------------------------------------------------------------------------------------------|
| `mode`                    | no       | `auto`                        | `auto` or `requested` - see modes table above                                                |
| `extra-instructions`      | no       | `""`                          | Repo-specific text appended to the end of the prompt                                         |

## Usage

### Auto mode (pull_request)

```yaml
# .github/workflows/review-agent.yml in the consumer repo
name: Review Agent

on:
  pull_request:
    types: [opened, synchronize, ready_for_review, reopened]

jobs:
  review:
    uses: mkmba-nz/github-infra/.github/workflows/review-agent.yml@v1
    secrets: inherit
```

That's the entire consumer-side workflow. The defaults handle the rest.

### Requested mode (re-review on @-mention)

```yaml
# .github/workflows/review-agent-rereview.yml in the consumer repo
name: Review Agent (Re-review)

on:
  pull_request_review_comment:
    types: [created]

jobs:
  rereview:
    if: |
      !contains(github.event.pull_request.labels.*.name, 'skip-workflows') &&
      (contains(github.event.comment.body, '@mkmba-review-agent') ||
       contains(github.event.comment.body, '@review-agent')) &&
       contains(fromJSON('["OWNER", "MEMBER", "COLLABORATOR"]'), github.event.comment.author_association)
    uses: mkmba-nz/github-infra/.github/workflows/review-agent.yml@v1
    secrets: inherit
    with:
      mode: requested
```

The @-mention filter lives in the consumer's `if:` rather than inside the
shared workflow so that each repo can choose its own handles (or add more).

The `author_association` check restricts re-review triggering to repo
members and collaborators. **This matters most for public repos**: any
logged-in GitHub user can post a comment on a public PR, and that comment
body is embedded directly in the prompt. Without the filter, an outsider
could inject arbitrary instructions into your review bot. In a private
repo the filter is effectively a no-op (everyone with PR-comment access
is at least a collaborator) but it doesn't hurt to include.

### Adding repo-specific guidance

```yaml
    with:
      extra-instructions: |
        Pay extra attention to schema migrations - this repo runs them
        automatically on deploy, so a broken migration ships immediately.
```

The text is appended verbatim to the end of the prompt. Combine with
`mode: requested` if needed.

## Review output

The prompt holds the agent to a one-report contract, so the same finding is
never published twice:

- **Findings appear exactly once.** A line-specific problem that needs
  changing goes in an inline comment; everything else goes in the summary
  comment. The summary must not restate anything already raised inline — at
  most it points at those comments or gives a count.
- **The summary comment is a verdict line plus at most five bullets.** No
  compliments, no description of the PR, no area-by-area narration. On a
  re-review it covers only what changed since the previous review, and does
  not restate earlier findings or approvals.
- **The review body is empty on approve, one line on request-changes.**
  `gh pr review --approve` is submitted with no body; `--request-changes`
  carries a single line naming the verdict and deferring to the summary
  comment, because GitHub rejects a request-changes review with an empty
  body.

The summary comment is the tracking comment that `track_progress: true`
publishes on the agent's behalf — the agent is told not to post a second one
of its own, and to end that comment with the `diff-hash` marker described
under [Skip / dedupe behaviour](#skip--dedupe-behaviour-auto-mode-only).

## Resolving review threads

The workflow runs [`actions/review-threads`](../../actions/review-threads),
which publishes two narrow commands the agent can use to list the review
threads on the PR it is reviewing and to resolve one of them. The agent is
asked to close only its own earlier findings that it now judges addressed or
no longer applicable, stating the reason — which the command posts as a reply
on the thread before resolving it, so a closure is always attributable. Those
rules are given to the agent on every review, not only on a re-review, so
whichever route triggers a review the guardrails come with it. This is a
capability consumers get automatically; there is no input to opt in or out.

Three consequences worth knowing:

- **The agent can resolve any thread on the PR, including a human's.**
  GitHub scopes thread resolution by write access, not by authorship, so the
  "only your own findings" rule is prompt guidance rather than a hard limit.
  Where a repo enables *Require conversation resolution before merging*, a
  wrongly closed thread unblocks a merge — the mandatory reason reply is what
  makes that visible and reversible.
- **The step is fail-closed.** If the action cannot publish a usable command
  path it fails the job and no review is posted, because the alternative is a
  run that looks healthy while every command is silently denied.
- **Reply-then-resolve is not atomic.** The concurrency group cancels a running
  review when a newer one starts, so a cancelled run can leave a reason reply
  on a thread that is still open, and the retry posts a second one. A duplicate
  reply is the accepted cost of never resolving a thread silently.

The commands themselves are documented in the action's
[README](../../actions/review-threads/README.md).

## Skip / dedupe behaviour (auto mode only)

1. **PR is merged or closed** → skip entirely. The review would land after
   the decision it was meant to inform. The state is read once, when the
   job starts: a merge that happens mid-review does not interrupt it.
2. **The head SHA already has an undismissed approval, from anyone** →
   skip entirely. Nothing a fresh review says takes that approval away, so
   it can only add noise, or race a merge the approval has already cleared.
   A reviewer's effective state is their latest `APPROVED` /
   `CHANGES_REQUESTED` / `DISMISSED` review; `COMMENTED` reviews are ignored
   and never revoke an approval. One approval is enough to skip, even where
   another reviewer's requested changes are still holding up the merge.
3. **Diff hash matches a prior review comment** → skip. The same SHA was
   reviewed; no work to do.
4. **A prior review exists with a different diff hash** → re-review, but
   in "focus on what's changed since the previous review" mode.
5. **No prior review** → first-pass review.

Every review summary comment ends with a `diff-hash:<sha256>` marker that
the dedupe logic reads on subsequent runs. Requested-mode re-reviews also
emit this marker, so a subsequent auto-mode run on the same diff will
correctly skip.

All five cases are decided from one `gh pr view` fetch, and that fetch is
deliberately fail-closed: if it fails (e.g. a GitHub API blip) the job fails
with an `::error::` annotation and no review is posted. Without the data we
cannot tell case 3 from case 5, and treating that as "no prior review" is
what silently disabled the dedupe entirely for the workflow's first year.
Re-run the job to recover from a transient failure.

In `requested` mode none of the above applies — the workflow always
proceeds to the review step.

# `build-and-push.yml`

Reusable workflow that builds a Docker image with Buildx and pushes it to
Amazon ECR, tagged with the triggering commit SHA.

AWS credentials come from an OIDC role assumption (`role-to-assume`), so the
consumer needs no static AWS keys. Layers are cached in the GitHub Actions
cache, scoped by `repo-name`.

## Required secrets

This workflow needs no secret. It declares no `workflow_call.secrets`, so a caller
that passes secrets for other reasons must use `secrets: inherit` — naming a
secret explicitly is rejected as an undefined input.

## Inputs

| Input            | Required | Default         | Description                                                                 |
|------------------|----------|-----------------|-----------------------------------------------------------------------------|
| `role-to-assume` | yes      | -               | ARN of the IAM role to assume via OIDC for the ECR push                     |
| `region`         | yes      | -               | AWS region of the target registry                                           |
| `registries`     | no       | (none)          | Registry IDs to log in to; unset means the credentials' own account         |
| `repo-name`      | yes      | -               | ECR repository name; also the layer-cache scope                             |
| `context`        | yes      | -               | Docker build context path                                                   |
| `file`           | no       | (none)          | Path to the Dockerfile relative to the workspace root; unset builds `<context>/Dockerfile` |
| `target`         | no       | (none)          | Build stage to stop at; unset builds the final stage                        |
| `build-args`     | no       | (none)          | Extra `NAME=value` build arguments, one per line                            |
| `platforms`      | no       | `linux/amd64`   | Target platforms to build                                                   |
| `runs-on`        | no       | from `platforms` | Runner label. Unset, it is derived from `platforms`: `ubuntu-26.04-arm` for exactly `linux/arm64` (a native build rather than QEMU), `ubuntu-26.04` otherwise. A label set here is always used; one that emulates a single-platform build logs a warning |
| `private-modules`| no       | `false`         | Mint a short-lived org-wide read-only token for the build and mount it as the BuildKit secret `github_token`; unset, no credential is mounted — see [Fetching private modules](#fetching-private-modules) |

Only `platforms` and `private-modules` declare a default, and `runs-on` is
derived from `platforms` when unset; the rest are simply unset, and the
behaviour listed above is what the underlying actions do with an empty value.

## Usage

The caller must grant `id-token: write` — the reusable workflow cannot hold a
permission the caller does not have, and without it the OIDC role assumption
fails.

```yaml
# .github/workflows/build.yml in the consumer repo
name: Build
permissions:
  id-token: write
  contents: read

on:
  push:
    branches: [main]

jobs:
  build:
    uses: mkmba-nz/github-infra/.github/workflows/build-and-push.yml@main
    secrets: inherit
    with:
      region: ap-southeast-2
      role-to-assume: arn:aws:iam::${{ vars.build_account_id }}:role/push-my-service
      registries: ${{ vars.images_account_id }}
      repo-name: my-service
      context: .
```

`github-infra` carries no tags, so consumers pin `@main`, as the example does.

## Building a Go image

Every image holding a Go binary is built one way, bar the
[exceptions](#exceptions) below: in Docker, with the layer cache as its only
cache. Copy this Dockerfile and change the names, the Go version and the
runtime stage:

```dockerfile
# syntax=docker/dockerfile:1
FROM golang:1.26-alpine AS build
RUN apk add --no-cache git
ENV GOPRIVATE=mkmba.nz,oncall-optimizer.com
WORKDIR /src
COPY go.mod go.sum ./
RUN --mount=type=secret,id=github_token,required=true \
    GIT_CONFIG_COUNT=1 \
    GIT_CONFIG_KEY_0="url.https://x-access-token:$(cat /run/secrets/github_token)@github.com/mkmba-nz/.insteadOf" \
    GIT_CONFIG_VALUE_0="https://github.com/mkmba-nz/" \
    go mod download
COPY . .
RUN git config --global --add safe.directory /src
RUN --mount=type=cache,target=/root/.cache/go-build \
    CGO_ENABLED=0 go build -buildvcs=true -o /out/my-service .

FROM alpine:3.22
COPY --from=build /out/my-service /usr/local/bin/my-service
ENTRYPOINT ["/usr/local/bin/my-service"]
```

A repository whose `go.mod` names no private module drops the secret mount,
the `GIT_CONFIG_*` assignments and `GOPRIVATE`; everything else stays the same.
The template declares no `ARG` at all. In particular there is no `ARG GITHUB_PAT`,
which would capture any `GITHUB_PAT` value a caller passes via `inputs.build-args`, and no `ARG` for the Go version.

- The `FROM golang:` tag is written literally, at the Go minor version (with a
  variant such as `-alpine` or `-bookworm` if wanted), because
  [`actions/setup-go`](../../actions/setup-go/README.md) reads the version from
  the first `FROM golang:` line, and a build argument there gives it nothing to
  read.
- `git` is installed where the variant ships none (alpine), because both the
  private-module fetch and `-buildvcs` need it.
- `GOPRIVATE` names the module path prefixes (the vanity domains, matching
  `actions/setup-go`'s default), not the `github.com/mkmba-nz` URL the
  modules are fetched from, which a vanity import path never matches.
- Both stages float on version tags (`golang:<major.minor>[-<variant>]` and
  `alpine:3.22` or a `distroless` image), so an ordinary rebuild picks up Go
  patch releases and OS fixes.
- **Modules live in the `go mod download` layer**, so whenever that step is
  served from the layer cache its modules come with it, and `go build` can never
  find them missing the way it does when they sit in a cache mount that starts
  empty.
- **No module cache mount on the build `RUN`** (nor on the download `RUN`):
  a mount at the module cache shadows the layer's modules, so `go build`
  downloads them again, which fails for a private module because the build
  `RUN` has no credential.
- **The build cache mount is cold every CI run and nothing depends on it**: it
  exists only to keep compiled objects out of the builder layer that the
  workflow's `cache-to: type=gha,mode=max` exports to the Actions cache.
- The official `golang` image sets `GOTOOLCHAIN=local` — do not unset it — so a
  `go.mod` requiring a newer Go than the `FROM` tag fails at the download step
  (and whoever raises the `go` directive raises the tag in the same change)
  rather than fetching a toolchain that leaves the `FROM` line, which
  `actions/setup-go` reads, naming a version the image no longer builds with.
- `.dockerignore` must keep `.git` and must exclude no tracked file (check it
  against `git ls-files`), because `-buildvcs` stamps the binary `(modified)`
  when a tracked file is missing from the copied tree.
- `safe.directory` carries no credential, and it keeps git working when the
  copied tree's owner differs from the build user.
- Cache mounts for non-Go steps (an `apt` cache, say) are allowed on the same
  terms: cold every run, with nothing depending on them.

A build whose context is a subdirectory of the repository rather than its root
has no `.git` to stamp from, so it sets `-buildvcs=false` explicitly instead.
`-buildvcs=true` is the template's default.

The caller's side:

```yaml
    with:
      # ...the inputs from Usage above, plus:
      private-modules: true      # only when go.mod names a private module
      platforms: linux/arm64     # only for an arm64 image
```

Leave `runs-on` unset so the workflow picks a native runner for a
single-platform build (see the `build-and-push.yml` Inputs table above).

### Exceptions

These do not follow the template:

- **cgo builds** (such as a SQLite-linking stage built inside an image) link
  libc, so they use a Debian-variant `golang` builder and static `-extldflags`.
- **Images of an upstream project with its own complex build** keep building
  on the runner through the upstream Makefile, with the Dockerfile only copying
  the output in.
- **Go builds that are not images** (such as the GitHub token broker Lambda)
  build on the runner with `actions/setup-go` on a native runner, with
  `CGO_ENABLED=0`, `-trimpath` and an explicit `GOARCH`.

### Verifying a build

In a correct run, the build step's log shows no `go: downloading` lines under
`go build` (every module came from the download layer). The run that proves it
is one where the `go mod download` step is reported `CACHED` but `go build`
still runs. The clean-checkout check
is the local build from [Outside CI](#outside-ci), run from a fresh clone
(without the `--secret` for a repository that fetches no private module):

```bash
docker build --secret id=github_token,env=GITHUB_PAT -t my-service .
```

## Fetching private modules

This section covers fetching private modules **inside an image build**. A job
that runs `go build`, `go test` or `govulncheck` directly on the runner gets
this from [`actions/setup-go`](../../actions/setup-go/README.md) instead, which
wires up an `insteadOf` rewrite and `GOPRIVATE` for the rest of the job on every
call — that job needs `id-token: write`. The notes below on
`x-access-token:`, org scoping and `GOPRIVATE` apply to both.

A credential is mounted into the build as the BuildKit secret `github_token`,
readable at BuildKit's default target `/run/secrets/github_token` for the
duration of the single `RUN` that mounts it. Which credential depends on
`private-modules`: set, the workflow mints a short-lived org-wide read-only
token per run through
[`actions/github-org-token`](../../actions/github-org-token/README.md), needing
no secret from the caller and no permission beyond the `id-token: write`
[Usage](#usage) already requires; unset, no credential is mounted.

Keeping it there is the consumer's job: **the token must not be written
anywhere that survives the `RUN`.** Supply the URL rewrite to the module fetch
itself, as shell-local environment on the same `RUN` that mounts the secret,
exactly as the `go mod download` step of the
[Go image template](#building-a-go-image) does.

- **Do not use `git config --global` to install the rewrite.** That writes the
  token in cleartext to `/root/.gitconfig`, which is part of the layer's
  filesystem — and this workflow's `cache-to: type=gha,mode=max` exports that
  layer to the GitHub Actions cache, where anyone with access to the
  repository's Actions environment can recover it. Mounting the secret and then
  doing `git config --global` closes nothing.
- **The workspace checkout holds no git credential.** The workflow checks out
  with `persist-credentials: false`, so the
  `http."https://github.com/".extraheader` carrying the job's `GITHUB_TOKEN` is
  removed before the checkout step finishes, not left in `.git/config` for the
  rest of the job. Without that, a build context including `.git` (as a Go build
  needs, to stamp VCS info) copies the token into a layer — exported to the
  Actions cache by the route above, and into the pushed image too where the copy
  lands in the final stage. So a `git` operation inside the build inherits no
  credential from the copied work tree: every authenticated fetch must use the
  `github_token` secret above.
- The `GIT_CONFIG_*` assignments are shell-local, not `ENV`, so they are not
  recorded in the image config either. Settings that carry no credential (say
  `safe.directory`) are still fine as an ordinary `git config --global`.
- **`x-access-token:` is load-bearing.** `https://<token>@github.com/` puts the
  token in the username with no password, which git treats as incomplete
  credentials: it tries to prompt, and `go mod download` sets
  `GIT_TERMINAL_PROMPT=0`, so the fetch fails rather than authenticating.
- The rewrite is scoped to the `mkmba-nz` prefix rather than all of
  `https://github.com/`, so the token is only ever attached to requests for
  this org's repositories.
- `# syntax=docker/dockerfile:1` selects the BuildKit frontend; under the
  classic builder `--mount` is a parse error. `GOPRIVATE` stops go resolving
  these modules through the public proxy and checksum database, which it would
  otherwise do without ever invoking git.
- BuildKit mounts the secret `mode=0400` owned by root. A stage that has
  switched to a non-root `USER` needs `--mount=type=secret,id=github_token,uid=<uid>`
  or it gets a permission denied on the `cat`.

### Outside CI

The same Dockerfile builds locally and on fly; only the way the secret is
supplied changes. Both need `GITHUB_PAT` set to a read-only PAT in the calling
shell, and `docker build` needs BuildKit (the default from Docker 23):

```bash
# local
docker build --secret id=github_token,env=GITHUB_PAT -t my-service .

# fly
fly deploy --build-secret github_token="$GITHUB_PAT"
```

The `id=…,env=…` form is `docker build` syntax; flyctl's `--build-secret`
takes plain `NAME=VALUE` pairs.

Note that the rewritten URL contains the token, so git prints it in full in
authentication error messages and under `GIT_TRACE` / `go mod download -x`.
Actions masks registered secrets in workflow logs; a local or fly build has no
such masking, so do not paste failing build output around.

# `self-test-actions.yml`

Not called by a consumer: it exercises the org-token path here rather than in
somebody else's build. It mints against the deployed `github-token-broker`
through [`actions/github-org-token`](../../actions/github-org-token/README.md)
and checks that a second call in the same job reuses that token, that
[`actions/setup-go`](../../actions/setup-go/README.md) wires it into git, and
that a job without `id-token: write` fails rather than degrading. Runs on
pushes to `main` under `actions/`, on a weekday morning schedule, and on
demand; its header comment says why not on a pull request.
