# setup-moonlit

Install the [Moonlit](https://github.com/wolfware-labs/moonlit) CLI in a GitHub Actions workflow.

```yaml
- uses: actions/checkout@v5
  with:
    fetch-depth: 0
- uses: wolfware-labs/setup-moonlit@v1
- run: moonlit run
  env:
    GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Works on `ubuntu-*`, `macos-*` and `windows-*` runners.

## Inputs

| Input | Default | Description |
|---|---|---|
| `version` | `latest` | `latest`, or an exact version such as `1.2.0`. Semver ranges are not supported. |
| `cache` | `true` | Cache the Moonlit plugin content directory between runs. |
| `cache-dependency-path` | `release.y*ml` | Glob whose matched files key the plugin cache. |

## Outputs

| Output | Description |
|---|---|
| `version` | The version actually installed. |
| `bin-dir` | The directory added to `PATH`. |
| `cache-dir` | The resolved plugin cache directory for this runner OS. |
| `cache-hit` | `'true'` on an exact cache key match, `'false'` on a partial hit against a `restore-keys` prefix, or `''` when caching was off or skipped. |

## Run a release pipeline

```yaml
name: Release

on:
  push:
    branches: [main]

permissions:
  contents: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
        with:
          fetch-depth: 0
      - uses: wolfware-labs/setup-moonlit@v1
      - run: moonlit run
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

`fetch-depth: 0` matters: pipelines that read commit history to decide a version need the full history, and a shallow clone silently gives them the wrong answer.

## Validate the pipeline on pull requests

```yaml
name: Validate pipeline

on:
  pull_request:
    paths: ['release.yml']

permissions:
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: wolfware-labs/setup-moonlit@v1
      - run: moonlit validate
```

## Private plugins

The action does not log in to a registry. Add a step:

```yaml
- uses: wolfware-labs/setup-moonlit@v1
- run: moonlit login --token "$MOONLIT_TOKEN"
  env:
    MOONLIT_TOKEN: ${{ secrets.MOONLIT_TOKEN }}
```

## How it installs

With the default `version: latest`, the action downloads and runs Moonlit's official
[cargo-dist](https://github.com/axodotdev/cargo-dist) installer script
(`moonlit-installer.sh` / `.ps1`) from the latest GitHub release, piping it into `sh` (or
`Invoke-Expression` on Windows) on the runner, with the job's secrets in scope. The installer
carries per-target sha256 checksums that are embedded into it at release build time, and it
verifies the downloaded archive against them before installing — so the binary that lands on
`PATH` is exactly the one the release produced. Those checksums pin the archive given that
installer script; they do not pin the script itself. Passing an exact `version:` (rather than
`latest`) fetches the installer from that specific tagged release, which pins the installer
script too, instead of tracking whatever `latest` resolves to at run time.

## Caching

The plugin content store is cached automatically, keyed on the runner OS and a hash of the files
matched by `cache-dependency-path`. Put `setup-moonlit` **after** `actions/checkout`; before it,
no config file is visible and caching is skipped with a log line saying so.

Moonlit's plugin store addresses plugin *content* by sha256, so a stale content entry is a cache
miss, never a wrong plugin. That guarantee does not extend to the store's `refs/*.json` files,
which cache a mutable tag's resolution to a digest for 15 minutes. `actions/cache` saves and
restores the whole store directory, so a job that starts within 15 minutes of the run that primed
the cache can resolve a mutable plugin tag to whatever digest was seen back then, instead of
re-querying the registry.

Set `cache: 'false'` to turn it off.

## Versioning

`@v1` tracks the latest v1 release. Pin `@v1.0.0` for byte-exact reproducibility.

## Licence

MIT OR Apache-2.0.
