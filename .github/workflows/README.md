# `review-agent.yml`

Reusable workflow that runs Claude Code over a pull request and posts a
review (approve / request-changes).

A single workflow with two modes:

| Mode (input)   | Typical trigger (in consumer) | What it does                                                                                  |
|----------------|-------------------------------|-----------------------------------------------------------------------------------------------|
| `auto` (default) | `pull_request`              | Reviews on open/synchronize/etc. Skips silently if the diff was already reviewed, or if a human has stepped in on a bot-authored PR. |
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
  suffix for App actors, so the workflow matches reviews and comments
  against the bare slug (`BOT_SLUG`) and passes the `[bot]`-suffixed login
  to the review action as `bot_name`.
- **Human vs agent.** The "is this a human reviewer?" check (auto mode
  only) excludes any reviewer whose login ends in `-agent`. Org-wide
  convention: every automated review-posting bot's slug ends in `-agent`.
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

## Skip / dedupe behaviour (auto mode only)

1. **PR opened by a bot AND already approved by us AND a human has
   reviewed since that approval** → skip entirely. Avoids tail-chasing on
   agent-authored PRs that humans are actively reviewing.
2. **Diff hash matches a prior review comment** → skip. The same SHA was
   reviewed; no work to do.
3. **A prior review exists with a different diff hash** → re-review, but
   in "focus on what's changed since the previous review" mode.
4. **No prior review** → first-pass review.
5. **The comment fetch itself fails** (e.g. a GitHub API blip) → the job
   fails with an `::error::` annotation, and no review is posted.

Every review summary comment ends with a `diff-hash:<sha256>` marker that
the dedupe logic reads on subsequent runs. Requested-mode re-reviews also
emit this marker, so a subsequent auto-mode run on the same diff will
correctly skip.

Case 5 is deliberately fail-closed: if we cannot read the prior comments we
cannot tell case 2 from case 4, and treating that as "no prior review" is
what silently disabled the dedupe entirely for the workflow's first year.
Re-run the job to recover from a transient failure.

In `requested` mode none of the above applies — the workflow always
proceeds to the review step.
