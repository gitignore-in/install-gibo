# Release process

This repository distributes a composite GitHub Action. A release is the Git tag
that users reference with `uses: gitignore-in/install-gibo@<ref>` plus the
matching GitHub Release notes. There are no binary assets to upload for normal
releases.

## Version policy

- Use tags in the `vMAJOR.MINOR.PATCH` format.
- Prefer annotated tags for new releases so the release intent is available in
  Git history as well as in the GitHub Release.
- Treat `v0.x` as pre-1.0: user-visible behavior can change, but release notes
  must call out breaking changes and migration steps.
- Keep the README usage example on a commit SHA for reproducibility. Mention
  the latest tag in the release notes rather than replacing the example with a
  mutable branch.

## Release criteria

Create a release when at least one user-visible change is ready, such as:

- a new pinned upstream `gibo` version,
- changed checksum anchors,
- changed default runtime behavior,
- changed inputs, outputs, platform support, or caching behavior,
- security or supply-chain hardening that users should pick up by pinning a new
  commit or tag.

Dependency-only workflow maintenance does not require a release unless it
changes the action behavior seen by users.

## Release checklist

1. Start from a clean `main` branch.
2. Review the diff since the previous tag:

   ```console
   git log --oneline <previous-tag>..main
   ```

3. Confirm that CI is green on `main`.
4. Run local documentation and workflow checks:

   ```console
   git diff --check
   typos README.md docs/release.md action.yml .github/workflows/*.yml
   npx --yes markdownlint-cli2 README.md docs/release.md
   go run github.com/rhysd/actionlint/cmd/actionlint@v1.7.12 -color
   ```

5. Draft release notes that include:
   - user-visible changes,
   - changed inputs or outputs,
   - platform support changes,
   - security or checksum-anchor changes,
   - migration steps for breaking changes,
   - the recommended pinned commit SHA for users that need reproducibility.
6. Create an annotated tag:

   ```console
   git tag -a vMAJOR.MINOR.PATCH -m "Release vMAJOR.MINOR.PATCH"
   git push origin vMAJOR.MINOR.PATCH
   ```

7. Create the GitHub Release for the same tag.
8. After publishing, verify that the release page points to the intended tag and
   that a sample workflow can use the tag or the pinned commit SHA.

## Pre-releases

Use GitHub pre-releases for changes that need validation before becoming the
recommended tag. Pre-releases should still include notes and the exact commit
SHA users can pin for testing.

## Failed release handling

If the release notes or tag point to the wrong commit, do not silently retarget
a published tag. Publish a corrected tag and release instead, then explain the
superseded release in the new release notes.
