# Contributing

## Bumping the pinned gibo version

When Renovate (or a manual PR) bumps the `version` in `action.yml`, the two
SHA256 anchor values must be updated in the same commit.

`action.yml` pins:

```sh
# renovate: datasource=github-releases depName=simonwhitaker/gibo
version=v3.0.22
checksums_txt_sha256=<SHA256 of upstream checksums.txt for v3.0.22>
checksums_windows_txt_sha256=<SHA256 of upstream checksums.windows.txt for v3.0.22>
```

Renovate's custom manager updates only the `version` line. The SHA256 values
must be updated manually to match the new release.

### Step-by-step

Replace `<new-version>` with the target release tag (e.g. `v3.0.23`).

```sh
new_version=<new-version>
base_url="https://github.com/simonwhitaker/gibo/releases/download/${new_version}"

# Download the new checksums files
curl -fsSLO "${base_url}/checksums.txt"
curl -fsSLO "${base_url}/checksums.windows.txt"

# Compute the anchor SHA256 values
sha256sum checksums.txt        # → new checksums_txt_sha256
sha256sum checksums.windows.txt  # → new checksums_windows_txt_sha256
```

On macOS `sha256sum` is not installed by default; use
`shasum -a 256 checksums.txt` instead, or install `coreutils` via Homebrew.

Update `action.yml` with the three new values:

```sh
# version line (already done by Renovate if this is a Renovate PR):
version=<new-version>

# anchor SHA256 values (must be updated manually):
checksums_txt_sha256=<output of sha256sum checksums.txt, first field only>
checksums_windows_txt_sha256=<output of sha256sum checksums.windows.txt, first field only>
```

The SHA256 output from `sha256sum` looks like:
`abcdef1234...  checksums.txt` — copy only the hash (first field, before the
two spaces).

### Why these values matter

See [README.md — Security model](README.md#security-model). The anchor values
move the trust anchor from the upstream release URL to this repository's git
history. Leaving them stale after a version bump causes the action's anchor
verification step to fail on the first CI run after the bump.

### Verifying the update locally

After editing `action.yml`, confirm the new anchor matches:

```sh
echo "<new-checksums_txt_sha256>  checksums.txt" | sha256sum --check -
echo "<new-checksums_windows_txt_sha256>  checksums.windows.txt" | sha256sum --check -
```

Both commands must print `checksums*.txt: OK`.
