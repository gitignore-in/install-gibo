# GitHub Action for Install gibo

## Usage

```yaml
steps:
- uses: actions/checkout@v4
- uses: gitignore-in/install-gibo@16003b5d1278dde15995459d9b4a01b146ec7798  # v0.1.0+
```

> **Pinning**: Pin to a specific commit SHA for reproducible, supply-chain-safe usage.
> `@main` is a mutable ref and may change without notice.
> Check the [releases page](https://github.com/gitignore-in/install-gibo/releases) for the
> latest tagged release, or use a commit SHA pointing to the tip of `main`.

## Prerequisites

The action runs as a composite `shell: bash` step. The runner must provide:

| Tool | Required when | Notes |
| --- | --- | --- |
| `bash` | always | Used as the step shell |
| `curl` | always | Downloads gibo release archive and checksums file from `github.com` |
| `sha256sum` | always | Verifies archive integrity; must be GNU coreutils `sha256sum` (not macOS `shasum`) |
| `tar` | Linux / macOS | Extracts the `.tar.gz` archive |
| `unzip` | Windows | Extracts the `.zip` archive |
| `git` | `update: 'true'` or `boilerplates-ref` set | `gibo update` and `boilerplates-ref` checkout both invoke `git` |

**Network**: the action contacts `github.com` at two points — to download the gibo
release archive from `simonwhitaker/gibo` and (when `update: 'true'`) to clone or
pull `simonwhitaker/gitignore-boilerplates`. Both require outbound HTTPS and git
transport to `github.com`. Set `update: 'false'` and restore the boilerplates
database from `actions/cache` to eliminate the second network call.

**Self-hosted runners**: `gibo update` writes `$HOME/.gitignore-boilerplates/`.
On self-hosted runners the `$HOME` directory persists across jobs; see
[Parallel jobs on self-hosted runners](#parallel-jobs-on-self-hosted-runners)
for the lock-based serialization this action provides and the recommended
caching strategy.

GitHub-hosted runners satisfy all of the above requirements out of the box.

## Inputs

| Input | Default | Description |
| --- | --- | --- |
| `version` | (action pin) | gibo release tag (e.g. `v3.0.22`). Leave empty to use the version pinned by the action. |
| `update` | `'true'` | Run `gibo update` after install. Set to `'false'` to skip the unconditional database refresh, e.g. when caching the boilerplates database externally. |
| `boilerplates-ref` | `''` | Optional git ref to check out in the boilerplates database after `gibo update`. Use a commit SHA to make later `gibo dump` output reproducible. |

## Outputs

| Output | Description |
| --- | --- |
| `version` | gibo release tag that was installed. |
| `bin-dir` | Directory containing the gibo executable (already on `PATH`). |
| `boilerplates-dir` | Directory where gibo stores its boilerplates database, as reported by `gibo root`. |
| `boilerplates-commit` | Resolved commit of the boilerplates database, when present. |

## Pinning the boilerplates database

By default, `gibo update` follows the current upstream boilerplates database
HEAD. Pin `boilerplates-ref` to a commit SHA when downstream steps need
byte-reproducible `gibo dump` output:

```yaml
steps:
- uses: actions/checkout@v4
- id: gibo
  uses: gitignore-in/install-gibo@16003b5d1278dde15995459d9b4a01b146ec7798  # v0.1.0+
  with:
    boilerplates-ref: 0123456789abcdef0123456789abcdef01234567

- run: |
    echo "database commit: ${{ steps.gibo.outputs.boilerplates-commit }}"
    gibo dump Python > .gitignore
  shell: bash
```

## Caching the boilerplates database

`gibo update` clones / pulls the upstream boilerplates database on every run.
Pair the action with `actions/cache` and `update: 'false'` to keep that
database across runs:

```yaml
steps:
- uses: actions/checkout@v4

- id: gibo
  uses: gitignore-in/install-gibo@16003b5d1278dde15995459d9b4a01b146ec7798  # v0.1.0+
  with:
    update: 'false'

- id: gibo-cache
  uses: actions/cache@v4
  with:
    path: ${{ steps.gibo.outputs.boilerplates-dir }}
    key: gibo-boilerplates-${{ steps.gibo.outputs.version }}-${{ github.run_id }}
    restore-keys: |
      gibo-boilerplates-${{ steps.gibo.outputs.version }}-

- if: steps.gibo-cache.outputs.cache-hit != 'true'
  run: gibo update
  shell: bash
```

## Concurrency notes

### Multiple calls in the same job

Calling this action twice in the same job (e.g., to install two different versions
side-by-side) is supported. Each version is installed into its own directory
(`$RUNNER_TEMP/gibo/<version>/bin`), so installations do not overwrite each other.
Both versions are added to `PATH`; the first entry wins for unqualified `gibo` calls.
Use `outputs.bin-dir` from each step to invoke a specific version explicitly.

### Parallel jobs on self-hosted runners

When multiple jobs run concurrently on a self-hosted runner **sharing the same user
account**, they share `$HOME/.gitignore-boilerplates`. If two jobs both call this
action with `update: 'true'` at the same time, this action serializes shared
boilerplates database writes with an atomic lock directory next to that database.
The lock protects `gibo update` and the optional `boilerplates-ref` checkout from
concurrent `git clone` / `git pull` / `git checkout` writers. If another process
holds the lock for too long, the action exits with a timeout instead of running an
unsafe concurrent update.

**Recommended mitigation for self-hosted runners:**

- Cache the boilerplates database and set `update: 'false'` (see [Caching](#caching-the-boilerplates-database) above).
  Only one job — the cache populator — runs `gibo update`, and subsequent jobs
  restore from cache without writing to the shared directory.
- Or use [GitHub-hosted runners](https://docs.github.com/en/actions/using-github-hosted-runners/), where each job gets an isolated VM.

## Security model

The action downloads release artifacts from upstream
[`simonwhitaker/gibo`](https://github.com/simonwhitaker/gibo), which does not
publish cosign/SLSA signatures. To avoid a self-referential trust chain
(where both the archive and its checksums file come from the same release
URL and can be swapped together), the action pins the SHA256 of upstream's
`checksums.txt` / `checksums.windows.txt` for the **default** `version`
directly in `action.yml`. The trust anchor is therefore this repository's
git history (commit review, branch protection) rather than the upstream
release URL.

What this defends against, for the default `version`:

- Upstream tag rewrite or release artifact replacement after publish
- TLS / CDN MITM that rewrites both archive and checksums in flight

What it does **not** defend against:

- Upstream compromise that occurs **before** the pinned anchor was committed
  to this repository
- A future `version` bump (with matching anchor update) merged into this
  repository under a compromised review process

When you set `inputs.version` to anything other than the action's pinned
default, the action falls back to the original self-referential verification
(both files trusted because they were fetched from the same release URL),
and prints a warning on stderr. Pinning a non-default `version` does **not**
disable archive verification — `sha256sum --check` still runs against the
downloaded `checksums` file — but the anchor step is skipped.

## License

MIT
