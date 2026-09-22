# `github-org-token`

Composite action that trades this job's GitHub Actions OIDC token for a
short-lived, org-wide, **read-only** GitHub token, and exports it as
`MKMBA_GITHUB_TOKEN` for the rest of the job.

The job's own `GITHUB_TOKEN` cannot read other private repositories in the org,
and **a composite action can read neither the `secrets` nor the `vars`
context** — so an org-standard action could only be handed a credential by
every workflow wiring one in. The OIDC request variables *are* plain job
environment, so a broker that trades one for a GitHub App installation token
needs no secret in any workflow. The broker is `github-token-broker`, an AWS
Lambda maintained alongside the org's other infrastructure.

## Usage

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      id-token: write   # required — see below
      contents: read
    steps:
      - uses: mkmba-nz/github-infra/actions/github-org-token@main

      # A later step. $GITHUB_ENV is not readable by the step that writes it.
      - run: gh repo view mkmba-nz/some-private-repo
        env:
          GH_TOKEN: ${{ env.MKMBA_GITHUB_TOKEN }}
```

`github-infra` carries no tags, so consumers pin `@main`, as the example does.

**The job needs `id-token: write`.** Without it GitHub injects no OIDC request
variables and the action fails the job with an `::error::` naming that fix. Note
that a job-level `permissions:` block *replaces* the workflow-level one, so a
job that has one must list every permission it relies on — `contents: read` for
`actions/checkout`, and so on. Forks get no OIDC token and are expected to fail
here.

Most Go repositories do not need this action directly:
[`actions/setup-go`](../setup-go/README.md) calls the same implementation on
every run, and
[`build-and-push.yml`](../../.github/workflows/README.md#build-and-pushyml) does
when its `private-modules` input is set.

## The token

| | |
|---|---|
| Scope | Every repository in the org, public and private |
| Permissions | `contents: read`, `metadata: read` |
| Lifetime | One hour, GitHub's installation-token default |

It is deliberately not narrowed to a repository allowlist: the requirement is
that any repo's build can fetch a Go module from any private repo.

| Exported | Value |
|----------|-------|
| `MKMBA_GITHUB_TOKEN` | The token, registered with `::add-mask::` before it is written anywhere |
| `MKMBA_GITHUB_TOKEN_EXPIRES_AT` | GitHub's expiry for it, unmodified |

Called a second time in the same job it re-registers the mask and stops, so
calling it twice costs one mint. A token within five minutes of expiry, with an
unparseable expiry, or carrying whitespace is replaced rather than reused — but
otherwise the reuse path trusts `MKMBA_GITHUB_TOKEN` as it finds it, so do not
set that name yourself in a workflow's `env:`.

It is a **job-wide credential**: `$GITHUB_ENV` is readable by every later step,
including third-party actions. Read-only and one hour bound the blast radius;
they do not remove it. Do not add this action to a job that runs untrusted code.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `broker-url` | no | the deployed Function URL, in `bin/mint-org-token` | Override to point a test run at another deployment |
| `audience` | no | `github-token-broker.mkmba.nz` | Override only alongside `broker-url` |

Both are non-secret, and empty means "use the default". The defaults live in
`bin/mint-org-token` rather than in `action.yml` because `actions/setup-go`
calls that script directly, and a default written in two places can disagree
with itself. `action.yml` passes them as positional arguments, so the script
never consults the ambient environment for either — which is what stops a
workflow-level `env:` redirecting where a job's OIDC token is sent. Setting
`broker-url` forces a fresh mint against it rather than reusing a token from
the default broker.

`actions/setup-go` takes no override of its own: call this action first with
`broker-url` set, and `setup-go` reuses what it minted.

## Why the implementation is a script

`bin/mint-org-token` holds the whole implementation and `action.yml` is a
wrapper over it, because `actions/setup-go` also needs it and **a composite
action cannot `uses:` a sibling action by relative path** — a relative `uses:`
resolves against `$GITHUB_WORKSPACE`, the caller's checkout, not the action's
own directory. Naming
`mkmba-nz/github-infra/actions/github-org-token@<ref>` from inside `setup-go`
would pin a ref independent of the one the caller pinned for `setup-go`;
calling the sibling's script through `$GITHUB_ACTION_PATH` gets it at exactly
the SHA the caller asked for. The script refuses to run from inside
`$GITHUB_WORKSPACE`, which is where a relative `uses:` would have put it and
where the code is the PR author's.
