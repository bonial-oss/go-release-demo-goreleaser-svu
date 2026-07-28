<!--
SPDX-FileCopyrightText: 2026 Bonial International GmbH
SPDX-License-Identifier: Apache-2.0
-->

# Repository Rulesets

This directory documents the branch and tag rulesets applied to this repo
via `gh api`. GitHub does not natively read rulesets from repo files — the
API/UI is the source of truth. These JSON files exist so a reader can see
the intended state without visiting the web UI.

## Apply

```bash
gh api -X POST repos/bonial-oss/go-release-demo-goreleaser-svu/rulesets \
  --input .rulesets/main.json

gh api -X POST repos/bonial-oss/go-release-demo-goreleaser-svu/rulesets \
  --input .rulesets/tags.json
```

Applied ruleset IDs are recorded in the plan's execution ledger
(`.superpowers/sdd/…/progress.md` in the meta repo).

## `main.json` — branch ruleset

Enforces PR-based changes to `main`:

- Requires a pull request (min 1 approval, dismiss stale on push, resolve
  review threads before merge).
- Required status checks: `ci / test-and-lint`, `commitlint / commitlint`.
- Requires signed commits.
- Blocks force-push (`non_fast_forward`) and branch deletion.
- No bypass actors — protection applies to everyone.

`enforcement: "active"` — real enforcement.

## `tags.json` — tag ruleset (evaluate-mode, provisional)

**Current state: `enforcement: "evaluate"` (log-only, does not block).**

Intent: restrict creation/updates/deletion of `v[0-9]+.[0-9]+.[0-9]+`
tags so only the release workflow can create them. The plan calls for
`enforcement: "active"` with `bypass_actors: [{actor_id: 15368,
actor_type: "Integration", bypass_mode: "always"}]` — the well-known
GitHub Actions integration ID — so `secrets.GITHUB_TOKEN` pushes in the
release workflow bypass the ruleset while human users cannot.

Applying that shape against `bonial-oss/go-release-demo-goreleaser-svu`
fails with:

```
Validation Failed:
"Actor GitHub Actions integration must be part of the ruleset source
 or owner organization"
```

The `bonial-oss` org does not have the built-in GitHub Actions
integration on its ruleset bypass list. It is not something an admin can
freely enable on the org side — GitHub treats it as a distinct app grant.

### Follow-up (proper fix for production)

Create a dedicated GitHub App (working name: `bonial-release`) with
minimal scopes:

- Repository permissions: `contents: write`
- Repository access: this repo only

Install the app at the org level. Then in `tags.json`:

- Replace `actor_id: 15368` with the new app's numeric App ID.
- Replace `actor_type: "Integration"` with `actor_type: "Integration"`
  (unchanged — it's still an Integration bypass, just a different one).
- Set `enforcement: "active"`.

And in `.github/workflows/release.yaml`, replace `secrets.GITHUB_TOKEN`
with a token minted from the new app (via
`actions/create-github-app-token` or similar), so the release workflow
pushes tags under the app's identity — which the ruleset then correctly
allows to bypass.

Until that App exists, this ruleset stays in `evaluate` mode: violations
are recorded in the ruleset's rule-suite log but not blocked. The
`immutable-releases` repo setting (Task 16) is the load-bearing
protection for release-asset immutability regardless; the tag ruleset
adds defense-in-depth on the tag layer.
