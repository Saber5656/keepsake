# Title

Security hardening pass: secret-hygiene CI gate, fuzzing, permissions audit, SECURITY.md

## Summary

Systematically enforce DESIGN §10.3/§10.4 across the finished surfaces: a CI gate proving no secret ever reaches logs/output, fuzz targets for every parser boundary, file-permission and dependency audits, and the public SECURITY.md policy.

## Context

Individual issues implement local rules; this issue proves them globally and wires regression protection. It runs after the major surfaces exist (13/14/23).

## Scope

- `internal/sectest/` harness, fuzz targets, `.github/workflows/ci.yml` additions, `SECURITY.md`, permission fixes discovered.

## Detailed Requirements

1. **Secret-hygiene gate**: harness runs the compiled binary through: seal (fixture), verify --deep, checkin, monitor timeline (REMINDING + RELEASED + drill), open — with planted recognizable secrets (KEK/shares/SMTP password/token from fixtures, e.g. containing marker `ZZSECRETZZ` in the SMTP password and known share strings). Capture ALL stdout+stderr+created non-secret files+git commits+state.json and grep for: raw share strings (except the single allowed share_G print in seal stdout and release-mail bodies in the SMTP capture), passphrase, SMTP password, token. Any hit fails CI. Allowed-location list is explicit in the harness (path + reason).
2. **Fuzz targets** (native Go fuzzing, 60s each in a nightly workflow + 10s smoke in PR CI): `FuzzSharetextDecode` (exists, 04 — wire into CI), `FuzzConfigLoad` (07's LoadVaultBytes/LoadTriggerBytes), `FuzzCheckinVerify` (17), `FuzzStateJSON` (19), `FuzzSMTPURL` (20), `FuzzTarExtract` (14 — crafted archives through the extraction hardening path).
3. **Permission audit test**: walk vault + trigger scaffold fixtures asserting modes (state 0700/0600, bundle intentionally 0644 — with comment, guides 0644, opened dir 0700/0600).
4. **Dependency audit**: `go.mod` require list must equal the DESIGN §10.6 allowlist (test parses go.mod; deviations fail with pointer to ADR process); `govulncheck` already in CI (02) — add `nightly` scheduled run.
5. **SECURITY.md**: supported-versions table, private reporting channel (GitHub private vulnerability reporting), 90-day coordinated disclosure, scope notes (what's a vuln vs. accepted residual risk — link DESIGN §10.5 table), no-bounty statement.
6. Fix everything the gates find; each fix references the gate that caught it.

## Acceptance Criteria

- [ ] Hygiene gate green and demonstrably sensitive: seeding a deliberate `fmt.Println(shareG)` in a scratch branch fails the gate (transcript in PR, then reverted).
- [ ] All 6 fuzz targets run in nightly workflow; PR smoke wired; zero findings open at merge.
- [ ] Permission + dependency audit tests green.
- [ ] SECURITY.md present and linked from README.
- [ ] gosec (via golangci-lint, 01) has zero suppressed findings without justification comments.

## Validation

CI (PR + nightly); the deliberate-leak demonstration is the meta-test.

## Dependencies

13, 14, 23 (audits their surfaces; earlier fuzz hooks from 04/07/17/19/20).

## Non-goals

External pentest/audit (post-v1, KU list), cosign/SLSA (v2, 29 notes), threat-model changes (DESIGN owns it).

## Design References

DESIGN §10.3 (rule 1 gate), §10.4 (each boundary → fuzz target), §10.6; ISSUE_PLAN §6.
