# `review-threads`

Composite action that publishes two narrow commands - `list-review-threads` and
`resolve-thread` - for an LLM step that is reviewing a pull request.

The motivating case is the review agent: it is told, on a re-review, to mark
comments it now judges addressed as resolved, but thread resolution exists only
in GraphQL and granting the model `gh api graphql` would hand it every query and
mutation its token can perform. These two commands reach exactly two
capabilities - listing the review threads on one pull request, and resolving one
of those threads - against exactly one pull request in one repository.

## Usage

```yaml
jobs:
  review:
    runs-on: ubuntu-latest
    steps:
      - uses: mkmba-nz/github-infra/actions/review-threads@main
        id: review-threads
        with:
          repository: ${{ github.repository }}
          pull-request-number: ${{ github.event.pull_request.number }}

      - uses: anthropics/claude-code-action@v1
        with:
          github_token: ${{ steps.app-token.outputs.token }}  # your App-token step
          prompt: ${{ format('Run {0}/list-review-threads to see the review threads on this PR, and {0}/resolve-thread <THREAD_ID> "<reason>" to resolve one.', steps.review-threads.outputs.commands_path) }}
          claude_args: |
            --allowedTools "${{ steps.review-threads.outputs.allowed_tools }},Bash(gh pr review:*)"
```

Place the action **before** the step that runs the model, and skip it on the
same condition that step skips on.

**The allowlist rule and the invocation the prompt asks for must share a
byte-identical prefix** - the absolute path in `commands_path`, with the rule
carrying the wildcard and the invocation carrying the arguments. A rule anchored
on one spelling matches no other: an absolute-path rule does not match a bare
name, and nothing matches `bash <command>`, a pipeline, or a leading
`VAR=value`. The failure is silent - with no interactive user an unmatched
command is denied and the run continues - so it presents as "the model resolved
nothing", never as an error.

Use the `allowed_tools` output rather than writing the rules out by hand: it is
built from the same value as `commands_path`, so that half of the match cannot
drift. It carries only these two commands - comma-join it with the entries the
caller already has rather than replacing them. The prompt's invocation still has
to be written out with `format(…, steps.<id>.outputs.commands_path)`, because a
`${{ }}` cannot be nested inside another `${{ }}` expression.

`github-infra` carries no tags, so consumers pin `@main`, as the example does.

## Where the commands live, and why

The commands are served from the action's own checkout
(`$GITHUB_ACTION_PATH/bin`), never from `$GITHUB_WORKSPACE`. A review job checks
out the pull request head, so everything in the workspace is content the PR
author controls; a wrapper sourced from there would be attacker-supplied code
running with the reviewing App's token. The model's working directory *is* the
workspace, which is why a relative invocation would be unsafe as well as
unstable.

If the action cannot publish a usable path - because the commands are missing,
are not executable, `GITHUB_ACTION_PATH` is unset, or the path it names turns out
to be inside the workspace - it fails the job. An empty path would turn every
allowlist rule into one that matches nothing, and an unmatched command is denied
silently.

## The commands

Both read the repository and pull request they act on from the environment this
action publishes (`REVIEW_THREADS_REPOSITORY`, `REVIEW_THREADS_PR_NUMBER`),
never from an argument the model supplies. Both write failures to stderr and
exit non-zero rather than exiting zero on a no-op: a non-zero exit is what lets
the model notice and report a failure, and the model's account is what a reader
of its summary will trust.

Neither needs a credential the review job does not already supply.
`claude-code-action` sets `GH_TOKEN` and `GITHUB_TOKEN` in the model's shell
environment to the value of its `github_token` input, which is the same route
that makes an allowlisted `gh pr review` work.

### `list-review-threads`

```
list-review-threads
```

Takes no arguments. Prints every thread on the pull request - thread ID,
resolution state, outdated state, file and line, and each comment's author,
timestamp and body - as plain text, followed by a count of threads and how many
are unresolved.

Threads are paginated in full. Comments are capped at 100 per thread, and a
thread that hits the cap says so in its own output rather than dropping comments
silently. The cap keeps the 100 *most recent* comments, not the first 100: the
newest are the ones that say whether a finding has been addressed, which is the
question the command exists to answer. Comments are numbered by their real
position in the thread, so a truncated thread does not renumber its tail from
one.

The output is deliberately plain text rather than JSON: Claude Code requires
each subcommand of a pipeline to match an allow rule independently, so the
model's natural `list-review-threads | jq …` would be denied, and this output
has to be readable exactly as printed.

Comment bodies are reproduced verbatim, indented four spaces so that no body can
forge a thread header. They are still content written by whoever commented,
including the author of the pull request under review, and the model reading them
holds the capability to resolve - a prompt should treat them as data rather than
as instructions.

### `resolve-thread`

```
resolve-thread <THREAD_ID> "<reason>"
```

`THREAD_ID` is the `thread-id` from `list-review-threads` output (a `PRRT_…`
node ID). The reason is required: it is posted as a reply on the thread, and the
thread is resolved only once that reply has succeeded. If the reply fails the
thread is left open and the command exits non-zero - a silently closed thread is
the outcome that ordering prevents. An already-resolved thread posts no reply,
mutates nothing, says so, and exits zero.

The command confirms the thread belongs to the pull request under review **in
the repository under review**, and exits non-zero without mutating when it does
not. Thread node IDs are globally addressable and pull request numbers repeat
across repositories, so a number-only comparison would leave the capability
nearly as broad as the general API access this action exists to avoid.

This is the same interface as the `resolve-thread` development agents use, in
`dev-infra`'s `framework/skills/address-review-feedback/scripts/` - the
two-argument, reply-then-resolve form that repository's own change to the script
introduces. The two copies cannot share a delivery path (one is bind-mounted
into containers from a private repo, the other fetched onto runners from this
public one), so presenting the same interface is the only defence available
against drift.

## Inputs

| Input                 | Required | Description                                                                                                    |
|-----------------------|----------|----------------------------------------------------------------------------------------------------------------|
| `repository`          | yes      | `OWNER/REPO` holding the pull request under review, e.g. `${{ github.repository }}`.                             |
| `pull-request-number` | yes      | Number of the pull request under review. Not a standard runner variable - it lives in the event payload.         |

## Outputs

| Output          | Description                                                                                  |
|-----------------|-----------------------------------------------------------------------------------------------|
| `commands_path` | Absolute path of the directory holding the two commands. Write it out in the prompt's invocation. |
| `allowed_tools` | `--allowedTools` entries for the two commands, built from `commands_path`. Comma-join with the caller's other entries. |

The path is an output rather than an environment variable on purpose: an
invocation written as `$SOME_VAR/resolve-thread` does not match an allowlist
rule written as an absolute path, and would be denied.

## Requirements

- A `gh`-authenticated environment in the step that runs the commands, with a
  token holding write access to pull requests in the repository under review.
- `jq`, used internally, and `bash` 4 or newer. Both, and `gh`, are present on
  GitHub-hosted runners.
