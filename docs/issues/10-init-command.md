# Title

`keepsake init`: vault creation with safety checks

## Summary

Implement `keepsake init`: create the local vault directory skeleton (DESIGN §6.1) with payload templates, an example config, correct permissions, and cloud-sync-path warnings.

## Context

First command every owner runs. Safety checks here (permissions, sync-dir warning) establish the trusted-machine boundary assumptions of DESIGN §10.2.

## Scope

- `internal/trigger` NOT touched; new `internal/vault/init.go` (+ `_test.go`) and wiring the `init` command in `cmd/keepsake`.

## Detailed Requirements

1. Behavior: create `<vault>/` with `payload/` (templates copied from `internal/content`), `out/`, `state/` (0700), and `config.yaml` generated from `config.DefaultVaultConfig()` with commented placeholders (YAML comments explaining each field, Japanese + English).
2. Refuse if target exists and is non-empty unless `--force`; `--force` never overwrites existing `payload/` files or `config.yaml` (only fills missing pieces) — destructive overwrite is out of scope by design.
3. Permissions: vault dir 0700; `state/` 0700; files 0600 (config may hold PII); `payload/` files 0600.
4. Cloud-sync warning W-INIT-1 (proceed with prompt, abort without `--yes`… default answer No) when the resolved absolute path contains any of: `Dropbox`, `Library/Mobile Documents`, `Google Drive`, `GoogleDrive`, `OneDrive`, `Box Sync`, `Nextcloud` (case-insensitive match on path segments where feasible).
5. Not a git repo: if `<vault>/.git` exists → hard error E-INIT-GIT with explanation (plaintext payload must not be version-controlled).
6. Output: summary of created paths + explicit next steps (edit payload → `keepsake seal`), via Printer.
7. `--json` prints `{created:[], warnings:[]}`.

## Acceptance Criteria

- [ ] Fresh init produces the exact §6.1 tree with asserted modes (skip mode asserts on Windows CI note).
- [ ] Non-empty dir without `--force` → exit 3, nothing written.
- [ ] `--force` on partial vault fills gaps, never rewrites existing files (content hash unchanged test).
- [ ] Sync-path fixtures trigger W-INIT-1; `.git` fixture → E-INIT-GIT.
- [ ] Templates byte-identical to `internal/content` embeds.

## Validation

Unit tests with t.TempDir(); manual macOS smoke: `keepsake init` under `~/Library/Mobile Documents/...` shows the warning (screenshot in PR).

## Dependencies

08, 09.

## Non-goals

seal/verify logic, trigger repo scaffold (24), config editing UX (owner edits YAML directly in v1).

## Design References

DESIGN §6.1, §6.4, §8 (init row), §10.2.
