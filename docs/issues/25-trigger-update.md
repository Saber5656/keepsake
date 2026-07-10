# Title

`keepsake trigger update`: refresh vendored binary and workflows after upgrade

## Summary

Implement `keepsake trigger update`: re-vendor the pinned binary at `trigger.binary_version`, regenerate workflow files and `keepsake.yaml` from current vault config, while preserving state and refusing to run over uncommitted local changes.

## Context

Binary upgrades are a deliberate owner action (ADR-003: pinned vendored binary; abuse A8: no auto-update channel). The update path must keep config/workflow drift impossible (regeneration from vault config = single source).

## Scope

- `internal/trigger/update.go` (+ tests), command wiring.

## Detailed Requirements

1. Preconditions: `trigger.local_path` is a git repo with clean tree for generated paths (`keepsake.yaml`, `.github/workflows/*`, `bin/*`) [E-TRG-DIRTY]; `state/` is never touched.
2. Steps: `dist.Ensure(trigger.binary_version)` (or `--tools-dir`) → replace `bin/keepsake-linux-amd64` + `.sha256` → regenerate `keepsake.yaml` + workflows via the same generator as 24 (guaranteeing parity) → print diff summary (`git -C ... diff --stat`) → instruct owner to review, commit, push (commands shown; `--commit` flag optionally performs `git add`+commit+push with message `keepsake trigger update to <version>`).
3. Version sanity: if the new binary's `keepsake version --json` (executed from the downloaded artifact for the HOST platform, not the linux one, via dist cache) reports a version != `trigger.binary_version` → E-TRG-VERMATCH abort. (Requires dist.Ensure to fetch host-platform binary alongside linux; extend 13's `Ensure` with a platforms parameter — coordinate: this issue owns the extension.)
4. Downgrade protection: semver comparison; downgrades require `--allow-downgrade`.
5. After update, print reminder: "run `keepsake drill` after pushing" (F4 mitigation).

## Acceptance Criteria

- [ ] Update on a scaffold fixture swaps binary + regenerates files; `state/` byte-identical; diff summary shown.
- [ ] Dirty-tree refusal; `--commit` path produces exactly one commit with expected message.
- [ ] Version-mismatch and downgrade guards tested with fixture artifacts.
- [ ] Regeneration parity test: `trigger init` output == `trigger update` output for same inputs (tree hash compare, excluding state/).

## Validation

CI unit tests with httptest dist fixture; manual real-repo update exercised before v1 tag (30's docs QA).

## Dependencies

24 (13's dist for the platforms extension).

## Non-goals

Auto-update, workflow customization preservation (hand-edits to generated files are overwritten by design — documented in README-switch.md), state migration (schema stable in v1).

## Design References

DESIGN §8 (trigger update row), §8.1, §13.1 F4; ADR-003.
