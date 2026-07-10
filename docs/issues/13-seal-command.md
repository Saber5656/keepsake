# Title

`keepsake seal`: end-to-end kit sealing pipeline with verified binary provisioning

## Summary

Implement `keepsake seal`: validate config → scan payload → deterministic tar → age-encrypt with a fresh KEK → split shares → build bundle → write seal-manifest → print share_G once with GitHub-secret instructions. Includes the verified GitHub-Releases download used to populate `tools/` (DESIGN §8.1).

## Context

This is the owner's core workflow tying together issues 05–07, 11, 12. Its output side-channel discipline (share_G printed exactly once, nothing logged) is a §10.3 requirement.

## Scope

- `internal/vault/seal.go` (+ `_test.go`), `internal/dist/download.go` (+ `_test.go`), command wiring.

## Detailed Requirements

1. Pipeline (each step's failure aborts atomically — `out/.tmp-*` cleaned):
   1. Load+validate config (print warnings; errors → exit 3).
   2. If a previous seal-manifest exists: show sealed_at + kek_fingerprint and require confirm (or `--yes`); message reminds that old bundles/Recovery Sheets become stale (rotation semantics — KEK always fresh per DESIGN §14 "Re-seal").
   3. `payload.Scan` (warnings shown; errors exit 3).
   4. Resolve tools: `--tools-dir DIR` (must contain the 4 platform binaries; names per §6.3) OR `dist.Ensure(version)` download path.
   5. Generate KEK; tar-stream directly into `agefile.Encrypt` writing `kit.age` (no plaintext tar ever on disk).
   6. Split shares; build bundle (12); write seal-manifest.
   7. Print final report: bundle path, kit hash/size, share_R CHECK, share_G CHECK, then the share_G block: framed, once, with instructions — set repo secret `KEEPSAKE_SHARE_G` manually (`gh secret set` command line shown as copy-paste but NOT executed), then "clear your terminal / do not screenshot" (Printer messages, both languages).
7. `internal/dist`: `Ensure(version string) (dir string, err error)` — download `keepsake_<ver>_<os>_<arch>[.exe]` + `checksums.txt` from `https://github.com/Saber5656/keepsake/releases/download/<ver>/`, verify each binary's SHA-256 against checksums.txt BEFORE moving into `~/.cache/keepsake/dist/<ver>/`, atomic rename, 0755; offline/failed → E-DIST-NET with `--tools-dir` hint (exit 5). No proxy config in v1 (system env honored by net/http).
8. `--dry-run`: run steps 1–3 only + report what would happen.
9. Timing: print elapsed; large payloads stream (no buffering whole tar).
10. Secrets hygiene: share_G/passphrase never in errors, logs, or `--json` output; `--json` emits report WITHOUT share_G (explicit field `share_g: "<printed-to-tty-only>"`).

## Acceptance Criteria

- [ ] Full seal on the fixture payload: bundle passes `bundle.Check`, kit decrypts with combined shares (test uses keys pkg), seal-manifest correct.
- [ ] Plaintext tar bytes never hit disk (code review + no temp tar file assertion during a monitored run).
- [ ] Previous-seal confirm flow; `--yes` bypass; abort leaves old `out/bundle` untouched.
- [ ] dist.Ensure: happy path against a local httptest server fixture; checksum mismatch → E-DIST-SUM and nothing cached; `--tools-dir` path works offline.
- [ ] share_G appears exactly once in captured stdout and nowhere in stderr/json.

## Validation

Unit + integration tests with httptest releases fixture; manual real-download smoke once a real release exists (29) — until then `--tools-dir` documented as the path (note in README development section).

## Dependencies

07, 12 (and transitively 05, 06, 11).

## Non-goals

Guide generation (16 — `seal` prints a reminder to run `keepsake guide`), trigger repo (24), re-distribution logistics (30).

## Design References

DESIGN §8 (seal row), §8.1, §5.2, §10.3 rules 1/5, §14.
