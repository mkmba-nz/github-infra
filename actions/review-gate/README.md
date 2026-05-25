# `review-gate`

Composite action that short-circuits a check when it has already completed
successfully for the current pull-request HEAD commit.

The motivating case is a check that runs alongside an automated PR reviewer.
When a human asks the reviewer for a second look on an unchanged HEAD, the
check shouldn't re-run - its previous result on the same SHA is still valid.

## Usage

```yaml
jobs:
  tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: mkmba-nz/github-infra/review-gate@v1
        id: gate
      - if: steps.gate.outputs.should_skip != 'true'
        run: ./bin/run-tests
```

Place the gate as the **first** step (or just after checkout) so that, when
it skips, the rest of the job exits quickly.

## How it decides

It queries the GitHub Checks API for `github.event.pull_request.head.sha`,
filters to runs whose name equals `check-name`, and counts how many have
already finished with `status=completed` and `conclusion=success`. If at
least one exists, the gate emits `should_skip=true`.

The currently-running job is `in_progress` rather than `completed`, so it
doesn't match itself.

## Inputs

| Input        | Required | Default              | Description                                                                                          |
|--------------|----------|----------------------|------------------------------------------------------------------------------------------------------|
| `check-name` | no       | `$GITHUB_JOB`        | The check name to look for. Defaults to the current job name, which is the right answer in most cases. Pass an explicit value when the gating job's name differs from the check it's gating. |

## Outputs

| Output        | Description                                                          |
|---------------|----------------------------------------------------------------------|
| `should_skip` | `'true'` if a successful run already exists for this SHA, else unset |

## Requirements

- The calling workflow must be triggered by a `pull_request` event - the
  action reads `github.event.pull_request.head.sha`.
- `GITHUB_TOKEN` is used via `gh api`; no extra permissions are needed
  beyond the default `contents: read`.
