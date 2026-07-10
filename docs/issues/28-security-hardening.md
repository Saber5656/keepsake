# Title

Security hardening pass: secret-hygiene CI gate, fuzzing, permissions audit, SECURITY.md

## Summary

Systematically enforce DESIGN §10.3/§10.4 across the finished surfaces: a normative secret-hygiene harness with an explicit allowed-locations list, fuzz targets for every parser boundary (names aligned with their owning issues), a file-mode matrix audit, a direct-dependency allowlist test, and the SECURITY.md policy (repo setting enabled by issue 31).

## Context

Individual issues implement local rules; this issue proves them globally and wires regression protection. Runs after the major surfaces exist (13/14/16/23).

## Scope

- `internal/sectest/` harness + `make sectest` + `make sectest-negative`, nightly fuzz workflow, `SECURITY.md`, `.github/workflows/nightly.yml`, permission fixes discovered.

## Detailed Requirements

1. **Secret-hygiene gate** (`internal/sectest`, normative harness spec):
   - Layout: one t.TempDir per scenario; planted fixture secrets with unique markers: SMTP password `ZZSMTPSECRETZZ`, share values from a fixture seal, GitHub token `ZZTOKENZZ`.
   - Command sequence (compiled binary): `init` → `seal --tools-dir` → `verify --deep` → `guide --yes` → `trigger init` → `checkin` (bare-repo fixture) → monitor timeline runs (REMINDING day, RELEASED day, drill run) → `open`.
   - Captured surfaces: all stdout/stderr; every file created under the vault, trigger repo, dest dirs, and `$TMPDIR`; `git log -p` of the trigger repo; the SMTP capture mailbox.
   - **Allowed-locations table (exact; everything else = failure)**:
     | Secret | Allowed locations |
     |---|---|
     | share_G text | seal stdout framed block; `state/seal-manifest.json`; release/postrelease bodies in SMTP capture |
     | passphrase P | `out/guides/owner-safe-sheet-ja.html` (text + QR payload) |
     | share_R text | `out/bundle/share-R.txt`; recipient guide HTML (text + QR payload) |
     | mail-auth code | guides + release/postrelease mail bodies |
     | SMTP password / token | NOWHERE |
   - `make sectest-negative`: builds the binary with `-tags sectest_leak` which enables one deliberate `fmt.Println(shareG)` behind the tag; the harness MUST fail on it (automated sensitivity proof, CI-run, nothing committed unbuilt).
2. **Fuzz targets** (nightly 60s each; PR smoke 10s): `FuzzDecode` (04, `internal/encoding/sharetext`), `FuzzLoadVaultBytes`/`FuzzLoadTriggerBytes` (07), `FuzzVerify` (17), `FuzzStateJSON` (19), `FuzzSMTPURL` (20 — added here if 20 didn't), `FuzzExtract` (11), `FuzzWorkflowRunsDecode` (NEW, added here to `internal/gh`: fuzz the runs-response decoder). Names/packages are exact — a script asserts each exists.
3. **Mode matrix audit test** (fixtures from real command runs):
   | Path | Mode |
   |---|---|
   | vault dir, `state/`, `payload/`, `out/` | 0700 |
   | `state/*` files, config.yaml, owner-safe-sheet | 0600 |
   | bundle dir | 0755 |
   | bundle documents (`kit.age`, manifests, share-R.txt, README html/txt, CHECKSUMS) | 0644 |
   | `tools/*` binaries | 0755 |
   | opened dir / files | 0700 / 0600 |
4. **Dependency test**: parse `go.mod` and assert the DIRECT require set ⊆ DESIGN §10.6 allowlist (indirects accepted); unauthorized direct addition fails with a pointer to the ADR process.
5. **SECURITY.md**: supported-versions table, private vulnerability reporting channel (the repo setting itself is enabled by issue 31 — cross-referenced), 90-day coordinated disclosure, scope table (accepted residual risks link DESIGN §10.5), no-bounty statement. Linked from README.
6. `nightly.yml`: scheduled fuzz (60s/target) + govulncheck, pinned actions, read-only permissions; any crasher minimized into the corpus as a regression test.

## Acceptance Criteria

- [ ] Hygiene gate green on the real tree; `make sectest-negative` red (CI job proves both every run).
- [ ] All 8 fuzz targets exist by exact name (script), nightly workflow green on the PR tip, PR smoke wired.
- [ ] Mode matrix + dependency tests green (and any violations found are fixed in this issue, each fix referencing the gate).
- [ ] SECURITY.md present, linked, and its private-reporting claim carries the "enabled in issue 31" cross-reference until 31 lands.
- [ ] gosec: zero unsuppressed findings; every suppression has a justification comment.

## Validation

CI (PR + nightly); the tag-gated negative build is the meta-test.

## Dependencies

13, 14, 16, 23 (audits their surfaces; fuzz hooks from 04/07/11/17/19/20/22).

## Non-goals

External pentest (post-v1), cosign/SLSA (v2), repo settings themselves (31), threat-model changes (DESIGN owns).

## Design References

DESIGN §10.3 (rule-1 gate + §5.3 artifact exceptions), §10.4 (every boundary → fuzz), §10.6; ISSUE_PLAN §6.
