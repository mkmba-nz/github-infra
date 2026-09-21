# MKMBA setup-go wrapper

This action wraps the top-level actions/setup-go GitHub action to provide the following functionality:

1) A default MKMBA production go language version that is used by default.
2) The ability for a caller to pass the location of a Dockerfile from which a go language version should be extracted
3) Optional credentials for fetching private Go modules from later steps in the same job.

Once the action has found a go language version to use, it calls the standard setup-go action to install it, and then
sets GOTOOLCHAIN=local into the environment for later steps so that the installed version is used without any further
changes.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `go-version` | no | `1.26`, unless `dockerfile` supplies one | Override the default or Dockerfile-extracted Go version. Last resort. |
| `dockerfile` | no | none | Path to a Dockerfile; the Go version is taken from its `FROM golang:` line. |
| `github-pat` | no | `""` | Read-only token used by later steps to fetch private Go modules. |
| `module-prefix` | no | `""`, meaning `github.com/<calling repository's owner>` | Git URL prefix the token may authenticate, e.g. `github.com/example-org`. Vanity module paths resolve to it via go-import metadata. |
| `goprivate` | no | `mkmba.nz,oncall-optimizer.com` | Module path patterns for `GOPRIVATE`, comma separated and prefix matched. Always exported, even without `github-pat`. |

## Fetching private modules

Pass `github-pat` when a later step in the job (`go build`, `go test`,
`go mod download`, `govulncheck`) needs a module from a private repository:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: mkmba-nz/github-infra/actions/setup-go@main
        with:
          dockerfile: Dockerfile
          github-pat: ${{ secrets.GIT_PAT }}

      # GOPRIVATE and the rewrite are already in the job environment.
      - run: go test ./...
```

Composite actions cannot read secrets (`secrets: inherit` does not reach them),
so the token must be passed explicitly.

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
  anything else in the job can read it.
- GitHub-hosted runners clear `$RUNNER_TEMP` between jobs; self-hosted runners
  must do so themselves.
- An empty `secrets.GIT_PAT` silently skips the wiring.
- `GOPRIVATE` is appended to, never replaced, and is set regardless of `github-pat`.
