# Title

`keepsake verify`: owner self-test of bundle, shares, and configuration

## Summary

Implement `keepsake verify` in `internal/vault`: one command proving the current seal is recoverable — config lint, bundle integrity, share round-trip with three-way fingerprint agreement, header or deep test-decrypt, payload drift — with a defined report/JSON schema and exit precedence.

## Context

Availability is a selected threat-model pillar; `verify` is step 1 of the annual drill (DESIGN §14). Builds on `bundle.Check` (12) and the hardened extractor (11).

## Scope

- `internal/vault/verify.go` (+ tests), command wiring.

## Detailed Requirements

1. Inputs (normative, no positional args): `${vault}/config.yaml`, `${vault}/state/seal-manifest.json`, `${vault}/out/bundle/` (manifest.json, kit.age, share-R.txt, tools/), flags `--deep`, `--json`.
2. Check items (IDs fixed; each reports PASS/FAIL/WARN/SKIPPED-dependency):
   - V1 config validation (07) — errors → FAIL, warnings → WARN
   - V2 seal-manifest present/parses; W-VER-STALE when `floor((now.UTC() − sealed_at)/24h) > 365`; future or unparsable `sealed_at` → FAIL
   - V3 bundle: `bundle.Check` report inlined per item
   - V4 fingerprint agreement: combine(share_R from bundle file, share_G from seal-manifest) → fingerprint; MUST equal seal-manifest `kek_fingerprint` AND bundle-manifest `kek_fingerprint` (three-way)
   - V5 decrypt proof — default: `agefile.Decrypt` reader opened with the combined passphrase, read the FIRST tar header and assert it is `payload-manifest.json` (proves passphrase + kit head integrity), then stop. `--deep`: full `payload.Extract` into `os.MkdirTemp` under `${vault}/state/` (0700; wiped via defer even on panic) with all hash checks
   - V6 payload drift: recompute `payload.PayloadDigest` from a fresh Scan of `payload/` and compare with seal-manifest `payload_digest`; mismatch → W-VER-DRIFT with changed/added/removed counts (warning, not failure)
   - V7 sync-path check: vault path matches the issue-10 cloud-sync detector → WARN (W-INIT-1 reused)
3. All items run even after failures, except honest dependency skips (e.g. V4/V5 SKIPPED when V2 failed) — skip reasons name the blocking item.
4. Exit precedence (normative): any V1 config ERROR → 3; else any FAIL → 4; else 0 (warnings allowed).
5. `--json` schema (documented in help): `{items:[{id:"V1",status:"pass|fail|warn|skipped",codes:[...],detail:"..."}],exit:N}`; emails redacted per §10.7 (no `--show-pii` here — verify shows no PII at all).
6. Secrets: only CHECK groups, mail-auth code absent, fingerprints shown; share texts/passphrase never printed (grep-tested).

## Acceptance Criteria

- [ ] All-green run on fresh seal fixture; each item individually falsifiable (≥8 negative fixtures incl. three-way fingerprint disagreement created by editing bundle manifest).
- [ ] Drift fixture (touch payload file) → V6 WARN with correct counts; agrees with `status` on the same fixture (shared helper noted).
- [ ] `--deep` on tampered kit → V5 FAIL with extractor code; temp dir gone afterwards (incl. induced-panic test).
- [ ] Dependency-skip semantics verified (missing seal-manifest → V4/V5 SKIPPED naming V2).
- [ ] Exit precedence table test; `--json` golden fixtures.
- [ ] Secret-hygiene grep over stdout/stderr/json with planted values.

## Validation

CI unit/negative suite; drill runbook (30) references `verify` as step 1.

## Dependencies

11, 12, 13, 14 (message-catalog reuse for share errors; extractor via 11).

## Non-goals

Trigger-repo state checks (32 `status` owns switch-side), remote secret verification (GitHub secrets are write-only by design — help text explains), payload content linting.

## Design References

DESIGN §8 (verify row), §6.6 (`payload_digest`, three fingerprint sources), §13.1 F13/F14, §14 (drill).
