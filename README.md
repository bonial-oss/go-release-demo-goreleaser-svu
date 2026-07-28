<!--
SPDX-FileCopyrightText: 2026 Bonial International GmbH
SPDX-License-Identifier: Apache-2.0
-->

# go-release-demo-goreleaser-svu

**Combo 2** reference implementation of immutable Go releases, using
**GoReleaser + svu + git-cliff**. Part of the
[go-release-demo](https://github.com/bonial-oss/go-release-demo)
evaluation project comparing three release toolchains.

## What this demonstrates

Cutting a versioned Go release where:

- The git tag cannot be moved, deleted, or re-created (tag ruleset).
- The release assets cannot be edited after publication (immutable-releases setting).
- The artifacts are signed by a verifiable workflow identity via **cosign keyless** (Sigstore, OIDC).
- The build has **SLSA L3 provenance attestations** proving how it was built.
- The build is **reproducible**: the same commit produces byte-identical binaries, testable via a rebuild-and-diff CI job.
- The release is **atomic**: the tag only exists if the full build, sign, and publish succeeded — no partial state ever reaches a consumer.

## Toolchain

| Concern | Tool |
|---|---|
| Next version from Conventional Commits | [`svu`](https://github.com/caarlos0/svu) |
| Release notes from Conventional Commits | [`git-cliff`](https://git-cliff.org) |
| Build, sign, SBOM, release | [`goreleaser`](https://goreleaser.com) |
| Keyless signing | [`cosign`](https://docs.sigstore.dev/cosign/) |
| Build provenance | [`slsa-github-generator`](https://github.com/slsa-framework/slsa-github-generator) |
| SBOM | [`syft`](https://github.com/anchore/syft) (invoked by GoReleaser) |

## Using the released binary

Download for your platform from the [latest release](https://github.com/bonial-oss/go-release-demo-goreleaser-svu/releases/latest):

```bash
# Example: linux/amd64
curl -fsSL -o demo.tar.gz \
  https://github.com/bonial-oss/go-release-demo-goreleaser-svu/releases/download/v0.1.0/demo_0.1.0_linux_amd64.tar.gz
tar xzf demo.tar.gz
./demo version
./demo verify
```

## Verifying the release

See [`docs/verification.md`](docs/verification.md) for cut-and-paste commands
that check the signature, provenance, and reproducibility of any downloaded
archive.

The bundled `demo verify` subcommand does the same check in-binary — the
binary you downloaded proves to you that it's the binary the release
workflow produced.

## How releases are cut

1. Merge a PR to `main` containing at least one `feat:` or `fix:` commit.
2. The `Release` workflow runs `svu next` against the Conventional Commit
   history since the last tag. If a bump is warranted, it proceeds to the
   atomic release pipeline.
3. Three sequential jobs: **goreleaser** (build, sign, SBOM, create draft
   release, push tag as the last action) → **slsa** (attach SLSA L3
   provenance to the draft) → **promote** (`gh release edit --draft=false`
   — the only write to a published release).
4. If any step fails, no tag exists, no release is visible to consumers.
   Re-dispatching the workflow retries cleanly.

A manual `workflow_dispatch` path is supported for dry-run previews — see
[`.github/workflows/release.yaml`](.github/workflows/release.yaml).

## Development

```bash
make install-tools   # install reuse into .venv-reuse
make lint            # reuse-lint + md-lint + go-lint
make test            # go test ./... -race -cover
make build           # go build ./...
```

Individual commits must be [Conventional Commits](https://www.conventionalcommits.org/); this is enforced by CI (`.github/workflows/commitlint.yaml`). The `main` branch protects against direct pushes; merge via PR.

## License

Apache 2.0. See [LICENSE](LICENSE) and [REUSE.toml](REUSE.toml).
