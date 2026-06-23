# Contributing

## Bumping the pinned gibo version

When manually bumping the `version` in `action.yml`, the two SHA256 anchor
values must be updated in the same commit. The gibo version is intentionally
not Renovate-managed: updating the version and both checksum anchors in the
same reviewed commit is a security requirement (see `README.md` — Security
model and `.github/renovate.json5`).

`action.yml` pins:

```sh
version=v3.0.22
checksums_txt_sha256=<SHA256 of upstream checksums.txt for v3.0.22>
checksums_windows_txt_sha256=<SHA256 of upstream checksums.windows.txt for v3.0.22>
```

All three values (`version`, `checksums_txt_sha256`, `checksums_windows_txt_sha256`)
must be updated manually in the same commit.

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
version=<new-version>
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
