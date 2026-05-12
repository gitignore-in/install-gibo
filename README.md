# GitHub Action for Install gibo

## Usage

```yaml
steps:
- uses: actions/checkout@v4
- uses: gitignore-in/install-gibo@main
```

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
  uses: gitignore-in/install-gibo@main
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
  uses: gitignore-in/install-gibo@main
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
