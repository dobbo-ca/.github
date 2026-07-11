# editorlint consolidation — migration plan (dobbo-ca/.github#2)

**Status:** APPROVED (2026-07-10) as the plan of record — decisions D1–D7 below are locked.
Execution deferred to a future session; each org transfer requires explicit per-step ok.
**Goal:** bring `editorlint/editorlint` (CLI) + `editorlint/action` (Action) back into `dobbo-ca`, collapsed to ONE repo, with homebrew/chocolatey/secrets pointed at dobbo-ca.

## Context — this REVERSES a prior transfer

`editorlint` (CLI) and `chocolatey-packages` originally lived under **dobbo-ca**, were
transferred OUT to a new `editorlint` org, and never fully cleaned up:
- `dobbo-ca/editorlint` and `dobbo-ca/chocolatey-packages` still **301-redirect** to `editorlint/*`.
- `editorlint/editorlint`'s `go.mod` is **still `module github.com/dobbo-ca/editorlint`** (never re-bumped).

So this migration is a transfer *back*, and several of issue #2's tasks are already partly true.

## Current state (researched 2026-07-10)

| Piece | State |
|---|---|
| CLI `go.mod` | already `github.com/dobbo-ca/editorlint` → **issue #2 task 4 (module path) is a no-op** |
| CLI `release.yml` | uses `dobbo-ca/.github/.github/workflows/release-go.yml@v1`; secrets `AUTOMATIONS_APP_ID/_PRIVATE_KEY`; dispatch to `${{ github.repository_owner }}/homebrew-taps` + `/chocolatey-packages`, `event_type: update_editorlint` (**owner-scoped → auto-repoints on transfer**) |
| CLI tags | bare (`0.3.0` final, `0.4.0-beta.37`) |
| CLI root `action.yml` | composite (inline bash) — **diverged** from the `editorlint/action` repo |
| `editorlint/action` | composite; downloads prebuilt binary via `install.sh` (hardcodes `editorlint/editorlint` release URLs) + `run.sh` + optional PR comment; extra `args` input; refactored (the more advanced of the two) |
| action `release.yml` | `semver-bump@v1` composite; secrets `AUTOMATIONS_APP_*`; **own regression guard** (commit `0b9e384`, redundant with central); **v0/v1 aliases via hand-rolled `git tag -f && push --force`** step |
| action redirect | `editorlint/editorlint-action` → `editorlint/action` (rename) confirmed |
| Consumers | **zero** in dobbo-ca + editorlint orgs; only `proscia-techops/github-actions-workflows@v1` (your work org — cross-org, unverified) |
| `editorlint/homebrew-taps` | `Formula/editorlint.rb` (URLs stale-point at dobbo-ca/editorlint); receiver `update-formulas.yml`, `types: [update_editorlint]`, payload `version` + `{darwin,linux}_{arm64,amd64}_sha256` |
| `dobbo-ca/homebrew-taps` | live (graphify-go/jt/az-go/autoresearch); generic receiver `types: [update-formula]`, payload needs a `formula` field + `{...}_sha` (**no `256`**); asset names hyphenated (`f-version-goos-goarch`) vs editorlint's underscored `editorlint_vVERSION_goos_goarch` |
| Chocolatey | `editorlint/chocolatey-packages` exists (nuspec + receiver `update-packages.yml`, `types:[update_editorlint]`, self-heals owner URLs). **`dobbo-ca/chocolatey-packages` does NOT exist** (redirect only) |
| GH App | editorlint side uses `AUTOMATIONS_APP_*`; dobbo-ca/homebrew-taps trusts `GH_PUB_APP_*` |

## Decisions — APPROVED (recommendations in **bold** are the locked choices)

- **D1 — Merge target repo:** transfer `editorlint/editorlint` → `dobbo-ca` and fold the Action into it. **Recommend: one repo `dobbo-ca/editorlint`, `action.yml` at the ROOT** (so `uses: dobbo-ca/editorlint@v1` works), Action scripts in `action/` or root. Archive `editorlint/action` (leaves a redirect).
- **D2 — Which action.yml is canonical?** The root (CLI) one and the `editorlint/action` one have diverged. **Recommend: keep the `editorlint/action` version** (external `install.sh`/`run.sh`, `args` input — more advanced), drop the CLI-root inline one. `install.sh` must be repointed `editorlint/editorlint` → `dobbo-ca/editorlint` release URLs.
- **D3 — Chocolatey:** `dobbo-ca/chocolatey-packages` doesn't exist. **Recommend: transfer `editorlint/chocolatey-packages` → dobbo-ca** (it exists + works, self-heals owner URLs) rather than recreate.
- **D4 — Homebrew:** **Recommend: fold the editorlint formula into `dobbo-ca/homebrew-taps`** (one tap, matches graphify-go/jt/az-go) rather than transfer `editorlint/homebrew-taps`. Cost: reshape the release dispatch to dobbo-ca's generic receiver — add `formula: editorlint`, rename `_sha256`→`_sha`, and align asset filenames to the hyphenated convention (or teach the receiver editorlint's underscored names). This is the biggest single reconciliation.
- **D5 — Secrets:** switch `AUTOMATIONS_APP_*` → `GH_PUB_APP_CLIENT_ID`/`GH_PUB_APP_PEM`; ensure the dobbo-ca GH App is installed on `dobbo-ca/editorlint` + `dobbo-ca/homebrew-taps` + `dobbo-ca/chocolatey-packages`.
- **D6 — Proscia consumer:** `proscia-techops/github-actions-workflows@v1` is the only known consumer (your work org). Does it still use the action, and should it repoint to `dobbo-ca/editorlint@v1`? Your call — it's cross-org and I can't see it fully.
- **D7 — task #4 (regression guard):** drop the action repo's per-repo guard (commit `0b9e384`); gate downstream on `steps.bump.outputs.changed`. Folds naturally into this work.

## Ordered sequence (minimize breakage)

The release dispatch is **owner-scoped**, so the instant the CLI repo lands in dobbo-ca its next
release publishes to `dobbo-ca/homebrew-taps` (no `editorlint.rb`) and `dobbo-ca/chocolatey-packages`
(doesn't exist). Therefore **prepare the dobbo-ca targets BEFORE transferring.**

1. **Prep dobbo-ca targets (non-breaking, no transfer yet):**
   - Create/populate `dobbo-ca/homebrew-taps/Formula/editorlint.rb` (adapt from editorlint's, fix URLs to `dobbo-ca/editorlint`).
   - Stand up `dobbo-ca/chocolatey-packages` (D3: transfer or recreate) with the `update-packages.yml` receiver.
   - Install the dobbo-ca GH App on those repos.
2. **Prep the consolidated repo content on a branch (in editorlint/editorlint, pre-transfer):**
   - Fold in the canonical `action.yml` (D2) + `install.sh`/`run.sh`, repoint release URLs to `dobbo-ca/editorlint`.
   - Reshape `release.yml`: dispatch payload/event to match dobbo-ca receivers (D4), swap secrets (D5), drop the guard (D7). Keep the v0/v1 alias step.
   - Fix README (broken `cdobbyn/taps`, add choco + `uses:` docs).
3. **Transfer (IRREVERSIBLE org steps — your explicit ok each):**
   - GitHub-transfer `editorlint/editorlint` → `dobbo-ca` (redirect stub resolves).
   - Merge the branch; archive `editorlint/action` (redirect remains).
4. **Verify:** merge→beta publishes to `dobbo-ca/homebrew-taps` + `dobbo-ca/chocolatey-packages`; a real tag cuts a full release (this also becomes the choco leg of the untested-release path from task #1). Update the Proscia consumer (D6). Confirm v0/v1 aliases still move.

## Risks / notes

- **Publishing breakage window** if transfer precedes target prep — sequence above avoids it.
- **Asset-name mismatch** (underscored vs hyphenated) is the subtlest reconciliation — must be consistent across release.yml build step ↔ homebrew receiver ↔ formula download URLs.
- **App-install access**: dispatches fail silently if the GH App isn't installed on the receiving repos.
- **Proscia cross-org** consumer can't be fully verified from here.
- Bare tag format retained (matches editorlint today).
