# Title

Owner documentation and operational runbooks

## Summary

Write the complete owner-facing documentation set: full README (EN + JA), the six operational runbooks from DESIGN §14, a setup tutorial, and execute every runbook once on a clean environment as the docs QA gate.

## Context

keepsake is an operational practice, not just a binary: rotation cadence, drills, and failure recovery ARE the product's reliability story (F6–F14 responses live here). Docs are also the OSS project's front door.

## Scope

- `README.md` (rewrite), `docs/ja/README-ja.md`, `docs/runbooks/*.md` (six files), `docs/tutorial.md`.

## Detailed Requirements

1. README: what/why (the two paradoxes from research 01), threat-model summary table (from DESIGN §10.2 — who can read what), quickstart (10 steps max, linking tutorial), install (from 29), FAQ (「GitHubが死んだら？」「家族がPC音痴でも？」「途中で家族と不仲になったら？」→ rotation), status badges, license/NOTICE pointers, link to DESIGN.md + SECURITY.md. Japanese README mirrors it (not machine-translated tone; owner review).
2. Runbooks (each: preconditions, numbered steps with exact commands, verification line per step, rollback/abort notes, expected duration):
   - `initial-setup.md` (vault→payload→seal→repo→secrets→checkin→drill; mirrors 24's checklist at full depth)
   - `annual-drill.md` (drill→per-recipient confirmation→address fixes→media refresh→calendar next date)
   - `rotation.md` (triggers: compromise/theft/false release/recipient change/regular re-seal; new-KEK seal→re-distribute→update KEEPSAKE_SHARE_G→destroy old sheets/bundles→drill)
   - `restore-vault.md` (F10: new machine rebuild + when rotation is mandatory)
   - `lost-signing-key.md` (F9: new key→allowed_signers→trigger update→push→verify checkin)
   - `decommission.md` (disable workflows→delete secrets→shred sheets→collect/destroy bundles→delete repo→tell recipients)
3. Failure-mode index: table mapping F1–F15 (DESIGN §13.1) → "handled automatically" or runbook link — completeness checked against DESIGN by a docs test (grep-based) so new failure modes can't go unmapped. (F15 quota guidance: keep the trigger repo on an account with minimal other private Actions usage.)
4. Tutorial: narrative first-run with expected output snippets (kept honest by running it — see QA gate).
5. QA gate: execute initial-setup + annual-drill + rotation runbooks end-to-end on a clean user account/VM with a real private repo and real (owner-controlled) recipient addresses; attach the execution log with timestamps and any doc corrections made (this closes ISSUE_PLAN §6 "Docs" layer and the v1 completion statement item 6).
6. Documentation for known gaps: Windows e2e coverage note (27), KU-5 Gatekeeper flow with exact dialog wording.

## Acceptance Criteria

- [ ] All files present; failure-mode index passes the completeness test.
- [ ] Runbook QA execution log committed under `docs/runbooks/qa-log-v1.md` (redacting private values) — every step verified once.
- [ ] README quickstart reproduced verbatim by the tutorial run.
- [ ] JA/EN parity review by owner recorded in PR.

## Validation

The QA gate execution itself; docs tests in CI (index completeness, link checker via lychee action pinned).

## Dependencies

26 (all flows must exist; uses 29's release artifacts in the tutorial).

## Non-goals

Recipient-facing guide content (09/16 own it), marketing site, localization beyond ja/en.

## Design References

DESIGN §13.1 (F6–F14), §14, §3.1 item 6; research 01 (positioning).
