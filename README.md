# Vanilla OS Desktop Image Merge

Composite GitHub Action that turns per-architecture image digests into a single multi-arch manifest list.

It downloads the `digests-*` artifacts uploaded by the build jobs, creates a manifest list for each tag, pushes it, and attests the result. It must run **after** all build jobs, on a runner with Buildah installed (e.g. `ubuntu-24.04`).

## Usage

```yaml
jobs:
  build:
    name: Build ${{ matrix.arch }}
    strategy:
      fail-fast: false
      matrix:
        include:
          - runner: ubuntu-24.04
          - runner: ubuntu-24.04-arm
    runs-on: ${{ matrix.runner }}
    steps:
      - uses: actions/checkout@v7

      - uses: vanilla-os/builder-gh-action@v1
        with:
          image: ghcr.io/${{ github.repository_owner }}/gnome
          base-image: ghcr.io/Vanilla-OS/core

  merge:
    name: Merge
    needs: build
    if: github.event_name != 'pull_request'
    runs-on: ubuntu-24.04
    permissions:
      contents: read
      packages: write
      id-token: write
      attestations: write
    steps:
      - uses: vanilla-os/merge-gh-action@v1
        with:
          image: ghcr.io/${{ github.repository_owner }}/gnome
```

## Inputs

Expects one empty file per architecture inside the downloaded digests directory, named after the image digest (as produced by the build action, in the `digests-<arch>` artifacts).

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `image` | ✅ | — | Target image name **without** a tag, e.g. `ghcr.io/owner/my-image`. Lowercased per the OCI spec. |
| `legacy-image` | | — | Optional legacy image reference including a tag (e.g. `ghcr.io/owner/desktop:main`). When set, the manifest list is also pushed there. |
| `registry-username` | | `github.repository_owner` | Registry username. Only needed for non-GHCR registries. |
| `registry-password` | | `github.token` | Registry password or token. Only needed for non-GHCR registries. |

## Outputs

| Output | Description |
| --- | --- |
| `digest` | Digest of the merged manifest list. |
| `image-url` | Target image URL without a tag. |

## Tags

Tags are derived from the git ref by [`docker/metadata-action`](https://github.com/docker/metadata-action):

| Ref | Tags |
| --- | --- |
| Tag `v1.2.3` | `1.2.3`, `1.2`, `v1.2.3`, `v1`, `latest` |
| Branch `dev` | `dev` |
| Other refs | none — the manifest cannot be pushed, so run the merge only on branches and tags |

## Required permissions

- `packages: write` — push the manifest list to GHCR.
- `attestations: write` — create the build attestation.
- `id-token: write` — required by `actions/attest`.
