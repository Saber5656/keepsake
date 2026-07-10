# Title

`keepsake verify`: owner self-test of bundle, shares, and configuration

## Summary

Implement `keepsake verify`: one command that proves the owner's current seal is recoverable — bundle integrity, share round-trip against the seal-manifest, fingerprint match, optional deep test-decrypt — plus config lint, producing a clear pass/fail report.

## Context

Availability is a selected threat-model pillar: "it decrypts when needed" must be checkable any time, and is a step in the annual drill runbook (DESIGN §14). Builds on `bundle.Check` (12).

## Scope

- `internal/vault/verify.go` (+ tests), command wiring.

## Detailed Requirements

1. Checks (each an independently reported item, all run even after failures):
   - V1 config: full validation + warnings (07)
   - V2 seal-manifest present/parses; age of seal shown (warn > 365 days: W-VER-STALE)
   - V3 bundle: `bundle.Check` report inlined
   - V4 share round-trip: shareR from bundle text + shareG from seal-manifest → Combine → fingerprint equals manifest `kek_fingerprint`
   - V5 kit header test-decrypt (default): decrypt first bytes only to prove passphrase correctness; `--deep`: full decrypt to temp (0700, deleted) with payload-manifest hash verification
   - V6 payload drift: hash current `payload/` scan vs sealed manifest → W-VER-DRIFT ("re-seal recommended") listing changed/added/removed counts
   - V7 tools presence/checksums (part of V3 but reported per-binary)
2. Output: table-style report via Printer (item, status ✔/✖/⚠, detail); `--json` structured equivalent; exit 0 all pass (warnings allowed), 3 config errors, 4 integrity failures.
3. `--deep` temp dir is wiped even on panic (defer + best effort), never inside cloud-synced paths (reuse 10's detector for the temp location choice: use `os.MkdirTemp` under the vault `state/`).
4. Never prints share texts or passphrase; only CHECK groups and fingerprints.

## Acceptance Criteria

- [ ] All-green run on fresh seal fixture; each check individually falsifiable via targeted fixture corruption (7 negative tests minimum).
- [ ] Drift detection: touch one payload file → W-VER-DRIFT with counts.
- [ ] `--deep` verifies hashes and cleans temp (assert dir gone).
- [ ] Report readable in both languages; `--json` schema documented in command help.

## Validation

CI unit/negative suite; drill runbook (30) references `verify` as step 1.

## Dependencies

12, 13.

## Non-goals

Trigger-repo state checks (`status` command in 08/23 scope), remote secret verification (impossible by design — GitHub secrets are write-only; documented in help text).

## Design References

DESIGN §8 (verify row), §6.6, §13.1 F13/F14, §14 (drill).
