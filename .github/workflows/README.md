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
