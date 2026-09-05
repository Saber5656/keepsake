# Title

`keepsake init`: vault creation with safety checks

## Summary

Implement `keepsake init` in `internal/vault`: create the local vault skeleton (DESIGN §6.1) with payload templates, a commented example config, exact permissions, git-worktree rejection, and cloud-sync-path warnings.

## Context

First command every owner runs. Safety checks here (permissions, sync-dir warning, no version control of plaintext) establish the trusted-machine boundary of DESIGN §10.2.

## Scope

- `internal/vault/init.go` (+ `_test.go`), command wiring; `internal/vault/syncpath.go` (cloud-sync detector, exported for reuse by 15).

## Detailed Requirements

1. Validation order (first failure of a hard rule aborts; codes exact):
   1. Target exists as non-directory or broken symlink → `E-INIT-NOTDIR` (exit 3).
   2. Target (after `filepath.Abs` + best-effort `filepath.EvalSymlinks`) is inside a git worktree: walk target and every ancestor for `.git` (directory OR file — worktrees use a file) → `E-INIT-GIT` (exit 3). Plaintext payload must never be version-controlled.
   3. Target exists non-empty and no `--force` → `E-INIT-NONEMPTY` (exit 3), nothing written.
   4. Cloud-sync check (warning): match resolved path segments case-insensitively against: `Dropbox`, `Google Drive`, `GoogleDrive`, `OneDrive`, `Box Sync`, `Nextcloud`, and the consecutive pair (`Library`,`Mobile Documents`). Match → prompt `Confirm` (default No). Interactive No or non-TTY without `--yes` → exit 3 with `W-INIT-1` explanation; `--yes` proceeds.
2. Creation (all paths, exact modes):
   | Path | Mode |
   |---|---|
   | `<vault>/` , `payload/`, `payload/20-credentials/`, `payload/40-documents/`, `out/`, `state/` | 0700 |
   | `config.yaml`, every template file | 0600 |
   Template mapping: `internal/content/payload-templates/<rel>` → `<vault>/payload/<rel>` (prefix stripped), byte-identical.
3. `config.yaml` generation: emitted from an embedded commented TEMPLATE (in `internal/content`, added by issue 09 as `config-template.yaml` — coordinate; this issue owns requesting it) with `REPLACE_ME` placeholders matching `config.DefaultVaultConfig()`; every field carries a preceding Japanese+English comment line. Golden fixture committed.
4. `--force` merge policy (per path): create if missing; NEVER overwrite or chmod existing files/dirs; report existing-but-different-mode as warning `W-INIT-2`. Extra files are left untouched.
5. Output: created-path list + next steps (`payload/ を編集 → keepsake seal`) via Printer; `--json` → `{"created":["payload/…"],"skipped":[...],"warnings":[{"code":"W-INIT-1","path":"..."}]}` (paths vault-relative, sorted).
6. Non-interactive matrix is normative: (`--json` implies no prompts — combined with sync-warning and no `--yes` → exit 3.)

## Acceptance Criteria

- [ ] Fresh init produces the exact §6.1 tree; modes asserted (POSIX only; test skipped on Windows with note).
- [ ] Ancestor-git fixture (vault under a repo subdir) and `.git`-file worktree fixture both → `E-INIT-GIT`.
- [ ] Non-empty without `--force` → exit 3 and target hash-identical afterwards.
- [ ] `--force` on partial vault: fills gaps, leaves existing files byte-identical, W-INIT-2 on mode drift.
- [ ] Sync-path table tests: each listed provider segment + the two-segment iCloud pair + a negative control; prompt/`--yes`/non-TTY behaviors asserted.
- [ ] Golden `config.yaml` fixture equality; generated config fails validation with only `E-CFG-PLACEHOLDER` errors (proves edit-me markers work).
- [ ] `--json` golden outputs for success and warning cases.

## Validation

Unit tests with t.TempDir(); the sync detector is reused by issue 15 (shared function noted in code).

## Dependencies

08, 09.

## Non-goals

seal/verify logic, trigger repo scaffold (24), config editing UX (owner edits YAML directly in v1).

## Design References

DESIGN §6.1, §6.4, §8 (init row), §10.2.
