# Title

`keepsake trigger update`: refresh vendored binary and workflows after upgrade

## Summary

Implement `keepsake trigger update` in `internal/trigger`: re-vendor the pinned binary at `trigger.binary_version`, regenerate ALL generated files via the same generator as `trigger init` (parity guaranteed), with an explicit conflict policy for hand-edited `keepsake.yaml`, dirty-tree refusal, version sanity, and a guarded optional `--commit`.

## Context

Binary upgrades are a deliberate owner action (ADR-003: no auto-update). Regeneration from vault config keeps config/workflow drift impossible; the conflict policy makes hand-edits explicit instead of silently overwritten.

## Scope

- `internal/trigger/update.go` (+ tests), command wiring. (DESIGN §8's `trigger update` row is amended by this issue's conflict policy — reflected there.)

## Detailed Requirements

1. Generated path set (single source shared with 24): `keepsake.yaml`, `.github/workflows/monitor.yml`, `.github/workflows/checkin-button.yml`, `bin/keepsake-linux-amd64`(+`.sha256`), `.gitignore`, `.keepsake-forbidden`, `README-switch.md`. `state/` is never touched.
2. Preconditions: git worktree with `origin` matching `trigger.repo` (issue 18's normalization reused) [`E-TRG-ORIGIN`]; current branch is the repo's default branch with upstream [`E-TRG-BRANCH`]; `git status --porcelain -- <generated paths>` empty — any staged/unstaged/untracked/renamed/conflicted entry counts [`E-TRG-DIRTY`].
3. Config conflict policy (normative): regenerate `keepsake.yaml` from vault config; if the existing repo file differs from what regeneration WOULD have produced for the OLD version (i.e. hand-edits exist), refuse with a field-level diff [`E-TRG-CONFLICT`] unless `--take-vault` (vault config wins) or `--keep-repo` (repo timing/recipients/mail preserved into the regenerated file; binary/version fields still updated). Workflows and README are always regenerated (hand-edits to them are unsupported — stated in README-switch.md).
4. Binary: `dist.Ensure(version, [{linux,amd64}, hostPlatform])` (13's API); version sanity = execute the HOST-platform binary from the dist cache with `version --json` and require `.version == trigger.binary_version` [`E-TRG-VERMATCH`]. Supported hosts: darwin-arm64/darwin-amd64/linux-amd64 (windows host unsupported for owner-side commands in v1 — clear error). `--tools-dir` must then contain BOTH the linux binary and a host binary.
5. Downgrade guard: compare via `golang.org/x/mod/semver` (allowlisted); downgrades need `--allow-downgrade`.
6. Output: `git diff --stat HEAD -- <generated paths>` summary (worktree vs HEAD after regeneration) + next steps. `--commit`: stages EXACTLY the generated paths, commits `keepsake trigger update to <version>`, pushes to the default branch upstream; refuses if the forbidden-file scan (23's rule) matches anything in the tree.
7. Post-update reminder: "push (if not --commit), then run `keepsake drill`" (F4 mitigation).

## Acceptance Criteria

- [ ] Update on a scaffold fixture: binary + `.sha256` + `keepsake.yaml` version fields updated; `state/` byte-identical; parity test `init`-tree == `update`-tree for identical inputs (hash compare over generated set).
- [ ] Conflict matrix: hand-edited timing → `E-TRG-CONFLICT` with field diff; `--take-vault` and `--keep-repo` both produce the documented results.
- [ ] Dirty variants (staged, untracked, deleted file in generated paths) each → `E-TRG-DIRTY`.
- [ ] Version-mismatch fixture binary → `E-TRG-VERMATCH`; downgrade guard; prerelease ordering via semver tested (`v0.2.0-rc.1 < v0.2.0`).
- [ ] `--commit` stages only generated paths (fixture with an unrelated dirty file outside the set: refused by dirty check? — outside-set dirt is ALLOWED and must remain unstaged; asserted), correct message, push observed on bare remote; forbidden-file fixture refuses.
- [ ] Wrong-origin and non-default-branch fixtures → specified errors.

## Validation

CI with httptest dist fixture + bare-repo remotes; real-repo update exercised before v1 tag (30's QA).

## Dependencies

24 (generator + path set; 13's dist and 18's origin normalization arrive transitively).

## Non-goals

Auto-update, state migration (schema stable in v1), preserving hand-edited workflows (unsupported by design).

## Design References

DESIGN §8 (trigger update row — conflict policy), §8.1, §10.6 (x/mod), §13.1 F4; ADR-003.
