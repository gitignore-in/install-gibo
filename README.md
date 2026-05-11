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
| `update` | `'true'` | Run `gibo update` after install. Set to `'false'` to skip the unconditional database refresh, e.g. when caching `~/.gitignore-boilerplates` externally. |

## Outputs

| Output | Description |
| --- | --- |
| `version` | gibo release tag that was installed. |
| `bin-dir` | Directory containing the gibo executable (already on `PATH`). |
| `boilerplates-dir` | Directory where gibo stores its boilerplates database (e.g. `$HOME/.gitignore-boilerplates`). |

## Caching the boilerplates database

`gibo update` clones / pulls `simonwhitaker/gitignore-boilerplates` on every run. Pair the action with `actions/cache` and `update: 'false'` to keep that database across runs:

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

## License

MIT
