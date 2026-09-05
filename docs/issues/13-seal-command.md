# Title

`keepsake seal`: end-to-end kit sealing pipeline with verified binary provisioning

## Summary

Implement `keepsake seal` in `internal/vault` plus `internal/dist`: validate config → scan payload → stream deterministic tar into age encryption → split shares → build bundle → generate guides and place the bundle README html → write `seal-manifest.json` → print share_G exactly once with the manual-secret checklist. `internal/dist` provides checksum-verified downloads of release artifacts (per-platform), reused by `trigger init/update`.

## Context

The owner's core workflow, tying together issues 05–07, 11, 12, 16. Its secret-output discipline is DESIGN §10.3 rules 1/5; the artifact-name contract binds issue 29.

## Scope

- `internal/vault/seal.go` (+ tests), `internal/dist/dist.go` (+ tests with httptest), command wiring.

## Detailed Requirements

1. Pipeline (numbered; any failure aborts atomically — new `out/.tmp-*` and a temp seal-manifest are discarded; the previous `out/bundle` AND previous `state/seal-manifest.json` remain byte-identical):
   1. Load+validate config (warnings shown; errors → exit 3).
   2. Previous-seal confirm: if `state/seal-manifest.json` exists, show `sealed_at` + `kek_fingerprint` and `Confirm` (default No; `--yes` bypass) with the rotation reminder (old bundles/Recovery Sheets become stale; KEK always rotates — DESIGN §14).
   3. `payload.Scan` (warnings shown; errors → exit 3).
   4. Resolve tools: `--tools-dir DIR` (must contain exactly the four §6.3 basenames as regular files — validated identically to issue 12 §5) OR `dist.Ensure` download.
   5. Generate KEK; `payload.WriteTar` streams through an `io.Pipe` into `agefile.Encrypt` writing `kit.age` in the tmp area (goroutine + `errgroup`-style join; both error legs propagated; no plaintext tar bytes ever on disk).
   6. Split shares; `bundle.Build` (issue 12).
   7. `guide.Generate` (issue 16); write `out/guides/*`; copy recipient guide into the bundle as `KEEPSAKE-README-ja.html`; rewrite bundle manifest `readme_html_sha256`.
   8. Write `state/seal-manifest.json` (0600; dir 0700): full DESIGN §6.6 field set incl. `share_g_text`, `payload_digest`, `mail_auth_code`; atomic tmp+rename with previous rotated to `seal-manifest.prev.json`.
   9. Final report (Printer): bundle path, kit hash/size, CHECK groups, mail-auth code, then the single framed share_G block with the manual checklist: `gh secret set KEEPSAKE_SHARE_G --repo <owner/repo>` shown WITHOUT the value (paste at gh's stdin prompt), then the "clear your terminal / no screenshots" notice.
2. `--json` is NOT supported on seal (usage error, exit 2) — machine output would conflict with single-shot secret printing (DESIGN §8).
3. `--dry-run`: steps 1 and 3 only; never prompts, never writes; prints the would-be plan and previous-seal info.
4. `internal/dist` (exact API):
   ```go
   type Platform struct{ OS, Arch string } // {"linux","amd64"}, {"darwin","arm64"}, {"darwin","amd64"}, {"windows","amd64"}
   func Ensure(version string, platforms []Platform) (dir string, err error)
   ```
   - Base URL: `https://github.com/Saber5656/keepsake/releases/download/<version>/`, overridable via `KEEPSAKE_DIST_BASE_URL` (any https URL; primarily for tests/e2e).
   - Artifact names (the CONTRACT with issue 29): `keepsake_<version>_<os>_<arch>` + `.exe` for windows; plus `checksums.txt` (`<sha256 lowercase>  <filename>` lines).
   - checksums.txt parsing: reject duplicates, malformed hex, unknown filenames (allowlist = the 4 artifact names for that version) [`E-DIST-SUM`].
   - Download policy: HTTPS only, status 200 only, 60s timeout and 200 MiB cap per file, redirects followed (final URL must be https).
   - Verify sha256 BEFORE moving into `~/.cache/keepsake/dist/<version>/` (per-call unique tmp dir + atomic rename; concurrent callers safe). Cache hits are re-verified against the cached checksums.txt on every use.
   - Errors: `E-DIST-SUM` → exit 4 (integrity); network/HTTP → `E-DIST-NET` → exit 5 with the `--tools-dir` hint. A contract test pins the exact artifact name template (issue 29 runs the same test).
5. Secrets hygiene: share_G and passphrase appear ONLY in the step-9 framed block (stdout) and inside `state/seal-manifest.json`; a test runs the full pipeline and greps stdout/stderr/every created file/temp dir for the planted share_G with exactly those two allowed locations.

## Acceptance Criteria

- [ ] Full seal on the fixture payload: bundle passes `bundle.Check`; kit decrypts via combined shares (test uses keys pkg); seal-manifest fields complete; `KEEPSAKE-README-ja.html` present with matching manifest hash.
- [ ] No plaintext tar on disk: instrumented run asserts no temp file contains the fixture payload marker bytes.
- [ ] Abort matrix (failure injected at each step 4–8): previous bundle and seal-manifest byte-identical, no stray tmp files.
- [ ] `--json` rejected (exit 2); `--dry-run` writes nothing and never prompts (tree hash before/after).
- [ ] dist: happy path via httptest (`KEEPSAKE_DIST_BASE_URL`); checksum mismatch → `E-DIST-SUM`, nothing cached; cache-hit re-verification catches a corrupted cached file; unknown/duplicate checksum lines rejected; artifact-name contract test green.
- [ ] tools-dir validation matrix (missing/extra/symlink/wrong-name → error, exit 3).
- [ ] Secret-hygiene grep test per requirement 5; state modes asserted (0700/0600).

## Validation

Unit + integration with httptest dist fixture; real-release download exercised in issue 29's validation.

## Dependencies

07, 12. (Pipeline step 7 calls guide generation through an internal interface `guideGen` that THIS issue defines and stubs — the stub writes a placeholder README html and marks the seal report "guides pending". Issue 16 implements the interface and removes the stub; execution order 13 → 16. This keeps the dependency graph acyclic.)

## Non-goals

Trigger repo scaffolding (24), rotation UX (runbook, 30), Windows-host sealing (ISSUE_PLAN §7).

## Design References

DESIGN §8 (seal row, no `--json`), §8.1, §5.2, §6.6, §10.3 rules 1/5, §14.
