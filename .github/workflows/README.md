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

## Required secrets

Pass via `secrets: inherit` if the names match in the consumer repo,
otherwise enumerate.

| Secret                       | Purpose                                                  |
|------------------------------|----------------------------------------------------------|
| `TS_OAUTH_CLIENT_ID`         | Tailscale OAuth client - used to reach the llmux gateway |
| `TS_AUDIENCE`                | Tailscale OIDC audience                                  |
| `REVIEW_AGENT_APP_ID`        | GitHub App ID for the bot that posts the review          |
| `REVIEW_AGENT_PRIVATE_KEY`   | GitHub App private key for the same App                  |

## Conventions

- **Bot login.** `bot-name` takes the `[bot]`-suffixed user login
  (e.g. `mkmba-review-agent[bot]`). The workflow strips the suffix when
  matching against GraphQL `author.login` values, which omit `[bot]` for
  App actors.
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
| `bot-id`                  | no       | `2798759`                     | Numeric GitHub App user ID. Defaults to the org-wide review App.                             |
| `bot-name`                | no       | `mkmba-review-agent[bot]`     | App user login with `[bot]` suffix. Defaults to the org-wide review App.                     |
| `claude-code-action-ref`  | no       | `v1.0.112`                    | Pinned ref of `anthropics/claude-code-action`                                                |
| `anthropic-base-url`      | no       | `""` (action default)         | Override the Claude API base URL. Blank = whatever `claude-code-action` defaults to. Set to llmux URL to route through the gateway. |
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

## Skip / dedupe behaviour (auto mode only)

1. **PR opened by a bot AND already approved by us AND a human has
   reviewed since that approval** → skip entirely. Avoids tail-chasing on
   agent-authored PRs that humans are actively reviewing.
2. **Diff hash matches a prior review comment** → skip. The same SHA was
   reviewed; no work to do.
3. **A prior review exists with a different diff hash** → re-review, but
   in "focus on what's changed since the previous review" mode.
4. **No prior review** → first-pass review.

Every review summary comment ends with a `diff-hash:<sha256>` marker that
the dedupe logic reads on subsequent runs. Requested-mode re-reviews also
emit this marker, so a subsequent auto-mode run on the same diff will
correctly skip.

In `requested` mode none of the above applies — the workflow always
proceeds to the review step.
