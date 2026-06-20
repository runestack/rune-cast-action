# `runestack/rune-cast-action`

Deploy services to a [Rune](https://github.com/runestack/rune) cluster
from GitHub Actions. A faithful 1:1 wrapper around `rune cast`.

## Quick start

```yaml
- uses: runestack/rune-cast-action@v1
  with:
    server:    ${{ vars.RUNED_HOST }}
    token:     ${{ secrets.RUNE_TOKEN }}
    source:    infra/runeset/casts/external-service.yaml
    values:    infra/runeset/values/stg.yaml
    namespace: stg
```

The `token` should be a [scoped service-account
token](https://github.com/runestack/rune/blob/main/_docs/rune-github-action.md),
not a root admin token. Mint one with:

```sh
rune admin service create ci-stg --namespace stg --permissions cast
```

### Deploy a named release

Cast a runeset as a named [release](https://docs.runestack.io) (a persisted,
diffable, roll-back-able record). Use `release` to pin the release name and
`yes` so a prune/adopt plan doesn't block on a confirmation prompt that has no
TTY in CI:

```yaml
- uses: runestack/rune-cast-action@v1
  with:
    server:  ${{ vars.RUNED_HOST }}
    token:   ${{ secrets.RUNE_TOKEN }}
    source:  https://example.com/runesets/api-1.4.2.runeset.tgz
    release: api          # overrides the runeset manifest name
    values:  infra/values/prod.yaml
    namespace: prod
    atomic:  true         # roll back if the deploy fails
    yes:     true         # auto-confirm prune/adopt (no TTY in CI)
```

To gate a deploy on the computed plan, render JSON and read the `result` output:

```yaml
- id: plan
  uses: runestack/rune-cast-action@v1
  with:
    server: ${{ vars.RUNED_HOST }}
    token:  ${{ secrets.RUNE_TOKEN }}
    source: ./runeset
    release: api
    dry-run: true
    output:  json
- run: echo '${{ steps.plan.outputs.result }}' | jq '.plan'
```

## Design principle

Every input maps **1:1** to a `rune cast` flag (or to a `rune login`
auth flag). No subset, no opinionated shortcuts. If the CLI doesn't
accept it, this action doesn't either; if the CLI grows a flag, this
action grows the matching input. Source of truth: `rune cast --help`.

## Inputs

### Auth (required)

| Input    | Description                                                                  |
|----------|------------------------------------------------------------------------------|
| `server` | Rune gRPC server address (`host:port`). Maps to `rune login --server`.       |
| `token`  | Bearer token. Piped over stdin so it never appears on argv. Mask as a secret.|

### What to cast

| Input    | Description                                                                  |
|----------|------------------------------------------------------------------------------|
| `source` | Required. Path / directory / glob / URL passed as the positional arg of `rune cast <source>`. |

### `rune cast` flags (all optional, all 1:1)

| Input              | CLI flag                          | Notes                                                       |
|--------------------|-----------------------------------|-------------------------------------------------------------|
| `namespace`        | `-n / --namespace`                |                                                             |
| `values`           | `-f / --values` (repeatable)      | Multi-line input → one `-f` per non-blank line.             |
| `set`              | `--set` (repeatable)              | Multi-line input → one `--set` per non-blank line.          |
| `tag`              | `--tag`                           |                                                             |
| `release`          | `--release`                       | Override the release name from the runeset manifest.        |
| `recursive`        | `-r / --recursive`                | For directory sources.                                      |
| `create-namespace` | `--create-namespace`              |                                                             |
| `force`            | `--force`                         | Force generation increment.                                 |
| `dry-run`          | `--dry-run`                       |                                                             |
| `render`           | `--render`                        | Print rendered manifests, do not apply.                     |
| `detach`           | `--detach`                        | Return immediately without waiting.                         |
| `timeout`          | `--timeout`                       | Default `5m`.                                               |
| `adopt`            | `--adopt`                         | Take over resources that are unmanaged or owned by another release. |
| `atomic`           | `--atomic`                        | Roll this cast's changes back on failure (verify timeout also triggers rollback). |
| `yes`              | `--yes`                           | Skip the confirmation prompt for plans that **prune or adopt**. Required in CI for such plans — without it the CLI blocks on a TTY that does not exist. |
| `output`           | `-o / --output`                   | `json` for a machine-readable plan + result. Also exposed as the `result` output. |

Boolean inputs accept `true` / `false` (and `1` / `0` / `yes` / `no`).

### Action-only

| Input          | Default                | Description                                            |
|----------------|------------------------|--------------------------------------------------------|
| `version`      | `latest`               | Rune CLI version (e.g. `v0.0.1-dev.26`) — passed to `runestack/rune-setup`. |
| `github-token` | `${{ github.token }}`  | Token for the CLI download.                            |

## Outputs

| Output    | Description                                       |
|-----------|---------------------------------------------------|
| `version` | The resolved Rune CLI version that was installed. |
| `result`  | The cast command's stdout. Most useful with `output: json`, where it is the structured plan + result for a downstream step to parse. |

## Examples

```yaml
# Single file — the 90% case.
- uses: runestack/rune-cast-action@v1
  with:
    server:    ${{ vars.RUNED_HOST }}
    token:     ${{ secrets.RUNE_TOKEN }}
    source:    infra/runeset/casts/external-service.yaml
    values:    infra/runeset/values/stg.yaml
    namespace: stg

# Full runeset directory.
- uses: runestack/rune-cast-action@v1
  with:
    server:    ${{ vars.RUNED_HOST }}
    token:     ${{ secrets.RUNE_TOKEN }}
    source:    infra/runeset/
    recursive: true
    values: |
      infra/runeset/values/base.yaml
      infra/runeset/values/prod.yaml
    namespace:        prod
    create-namespace: true

# Override values from the workflow context.
- uses: runestack/rune-cast-action@v1
  with:
    server:    ${{ vars.RUNED_HOST }}
    token:     ${{ secrets.RUNE_TOKEN }}
    source:    infra/runeset/
    values:    infra/runeset/values/stg.yaml
    set: |
      app.tag=${{ github.sha }}
      app.replicas=3
    namespace: stg

# Dry-run on PRs.
- uses: runestack/rune-cast-action@v1
  if: github.event_name == 'pull_request'
  with:
    server:    ${{ vars.RUNED_HOST }}
    token:     ${{ secrets.RUNE_TOKEN }}
    source:    infra/runeset/casts/external-service.yaml
    values:    infra/runeset/values/stg.yaml
    dry-run:   true
```

## Token hygiene

- Use [`rune admin service create`](https://github.com/runestack/rune)
  to mint a scoped service-account token instead of using the root
  bootstrap token.
- Use [GitHub Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)
  to scope `RUNE_TOKEN` per environment so the `stg` job can never
  see the `prod` token.
- The action calls `::add-mask::` on the token and pipes it over
  stdin via `rune login --token-stdin`, so it never appears on
  argv (`/proc/<pid>/cmdline`, `ps`, shell history) or in step logs.

## Companion: `runestack/rune-setup-action`

If you only want the CLI installed (e.g. to run `rune get`, `rune
logs`), use [`runestack/rune-setup-action`](https://github.com/runestack/rune-setup-action)
directly.

## License

MIT — see [LICENSE](LICENSE).
