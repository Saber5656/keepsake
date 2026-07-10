# Title

`keepsake status`: vault and switch state overview

## Summary

Implement `keepsake status` per DESIGN §8: a read-only overview of payload inventory, seal freshness/drift, and — when the trigger repo clone is available — the switch state from `state/state.json`, with a documented `--json` schema and PII redaction.

## Context

DESIGN §8 defines `status` but no other issue implements it (found by plan review). It is the owner's daily-driver command and the surface for the `--show-pii` privacy rule (DESIGN §10.7).

## Scope

- `internal/vault/status.go` (+ `_test.go`), command wiring in `cmd/keepsake`.

## Detailed Requirements

1. Sections (each independently degradable — missing inputs render as `status: unknown (reason)`, never a hard error):
   - **Payload**: file count, total size, newest mtime from `payload.Scan` in inventory-only mode (no caps errors — warnings shown).
   - **Seal**: `sealed_at`, age in days (warn >365: W-VER-STALE wording reused), `kek_fingerprint`, recipients count; drift = recompute `payload_digest` (DESIGN §6.6) and compare — on mismatch print "re-seal recommended" with changed/added/removed counts.
   - **Switch**: if `trigger.local_path` exists and is a git repo: read `state/state.json` (schema 19); show phase, armed, last_valid_checkin + elapsed days, next thresholds (remind/alert/release dates computed from timing config), release/sent summary if RELEASED. `--fetch` runs `git -C <path> fetch && git merge --ff-only @{u}` first (failure → note, not error; never merges non-FF).
2. Exit code: always 0 when the command itself succeeds, even if the switch is in a bad state (status is an observer); 3 only for unusable vault config; never 4/5.
3. `--json`: stable schema `{payload:{...}, seal:{...}, switch:{...}}` documented in command help; emails redacted as `h***@example.com` unless `--show-pii` (DESIGN §10.7); all timestamps RFC3339 UTC.
4. No secrets ever shown: share texts, passphrase, mail-auth code are all excluded; fingerprints and CHECK groups allowed (same policy as `verify`, DESIGN §10.3).
5. Human output via Printer (bilingual); table-like alignment, one section header per block.

## Acceptance Criteria

- [ ] Fixture matrix: fresh vault (no seal, no trigger) / sealed no-drift / sealed with drift / trigger repo in each phase (ACTIVE unarmed, ACTIVE armed, REMINDING, ALERTING, RELEASED) — golden outputs for human and `--json` forms.
- [ ] Redaction on by default in `--json`; `--show-pii` reveals; human output shows names but not emails unless `--show-pii`.
- [ ] Drift detection agrees with `verify` V6 on the same fixtures (shared helper).
- [ ] `--fetch` behavior tested against a local bare-repo fixture (ahead → ff-only merge; diverged → note without merge).
- [ ] No-secret grep test over all outputs with planted fixture secrets.

## Validation

CI unit tests; used daily by the owner from wave-2 onward (dogfood feedback loops into issue 30 docs).

## Dependencies

08, 13, 19.

## Non-goals

Mutating anything (no checkin, no seal), remote API calls (dispatch verification is monitor's job), notifications.

## Design References

DESIGN §8 (status row), §6.6 (`payload_digest`), §6.8, §10.3, §10.7.
