# MKMBA setup-go wrapper

This action wraps the top-level actions/setup-go GitHub action to provide the following functionality:

1) A default MKMBA production go language version that is used by default.
2) The ability for a caller to pass the location of a Dockerfile from which a go language version should be extracted
3) Access to private Go modules from later steps in the same job, with no credential wired in by the caller.

Once the action has found a go language version to use, it calls the standard setup-go action to install it, and then
sets GOTOOLCHAIN=local into the environment for later steps so that the installed version is used without any further
changes.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `go-version` | no | `1.26`, unless `dockerfile` supplies one | Override the default or Dockerfile-extracted Go version. Last resort. |
| `dockerfile` | no | none | Path to a Dockerfile; the Go version is taken from its `FROM golang:` line. |
| `module-prefix` | no | `""`, meaning `github.com/<calling repository's owner>` | Git URL prefix the token may authenticate, e.g. `github.com/example-org`. Vanity module paths resolve to it via go-import metadata. |
| `goprivate` | no | `mkmba.nz,oncall-optimizer.com` | Module path patterns for `GOPRIVATE`, comma separated and prefix matched. |

## Fetching private modules

Private module access is set up on every call, so later steps in the job (`go
build`, `go test`, `go mod download`, `govulncheck`) can reach a module in a
private repository without anything further. **The job needs `id-token:
write`** — this action mints the credential from the job's OIDC token, and
without that permission it has nothing to mint from and fails:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: mkmba-nz/github-infra/actions/setup-go@main
        with:
          dockerfile: Dockerfile

      # GOPRIVATE and the rewrite are already in the job environment.
      - run: go test ./...
```

No secret is passed, and none can be: composite actions read neither the
`secrets` nor the `vars` context. The action calls
[`actions/github-org-token`](../github-org-token/README.md#usage) instead, which
trades the job's OIDC token for a short-lived org-wide read-only token and
exports it as `MKMBA_GITHUB_TOKEN`. Granting `id-token: write` is the whole
migration for a consumer repository; see that README for the trap in adding it
to a job that already has a `permissions:` block.

Set `module-prefix` only when the private modules live under a different owner
than the calling repository, e.g. `module-prefix: github.com/mkmba-nz`.

Vanity import paths such as `mkmba.nz/x` work unchanged, because their
go-import metadata points at `github.com/mkmba-nz/...`, which the rewrite
covers, while `GOPRIVATE` must name the vanity domain, which the default
already does. Modules imported directly as `github.com/mkmba-nz/...` need
`goprivate` extended to include `github.com/mkmba-nz`.

### Limits worth knowing

- The token is written as an `url.<base>.insteadOf` rewrite into a `0600` file
  under `$RUNNER_TEMP`, included from `~/.gitconfig`, so later steps and
  anything else in the job can read it — as can any third-party action, since
  `$MKMBA_GITHUB_TOKEN` is job-wide. Do not call this action from a job running
  untrusted code.
- GitHub-hosted runners clear `$RUNNER_TEMP` between jobs; self-hosted runners
  must do so themselves.
- The token lasts an hour. A job still running Go commands after that needs a
  second `setup-go` call with the **same** `module-prefix`; the include file is
  keyed by prefix, so a different one writes a second rewrite beside the stale
  first, and calling `github-org-token` alone refreshes the variable but not the
  rewrite built from it.
- `GOPRIVATE` is appended to, never replaced.
- **Fork pull requests fail here.** A fork's job gets no OIDC token whatever
  permissions the workflow declares, so there is nothing to mint from. A
  repository taking fork contributions needs its Go jobs arranged not to call
  this action on a fork's ref.
