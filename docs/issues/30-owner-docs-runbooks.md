# Title

Owner documentation and operational runbooks

## Summary

Write the complete owner-facing documentation: full README (EN + JA), six runbook files covering all seven DESIGN §14 rows plus F9, a failure-mode index test-mapped to F1–F15, a setup tutorial, and a QA gate executing EVERY runbook once (destructive ones against scratch fixtures).

## Context

keepsake is an operational practice, not just a binary: rotation cadence, drills, and failure recovery ARE the reliability story (F6–F15 live here). Docs are also the OSS project's front door.

## Scope

- `README.md` (consolidating rewrite — preserves 02's badge, 29's install section, 28's SECURITY link as-is), `docs/ja/README-ja.md`, `docs/runbooks/*.md` (six files + index), `docs/tutorial.md`, docs-test additions.

## Detailed Requirements

1. README: what/why (naming the **self-hosting paradox** and the **subscription paradox** from research 01 explicitly), who-can-read-what table (from DESIGN §10.2), 10-step quickstart linking the tutorial, install (29's section), FAQ (「GitHub が死んだら？」「家族が PC 音痴でも？」「途中で家族と不仲になったら？→rotation」「クォータ超過したら？(F15)」), links to DESIGN/SECURITY/runbooks. `docs/ja/README-ja.md` mirrors it in natural Japanese (owner-reviewed, not machine tone).
2. Runbook files and DESIGN §14 mapping (explicit table in the index):
   | File | Covers |
   |---|---|
   | `initial-setup.md` | Initial setup (mirrors 24's checklist at full depth; secrets are OWNER-set manually per DESIGN §10.3 rule 6 — the runbook never has the agent handle credentials) |
   | `annual-drill.md` | Annual drill + per-recipient confirmation + media refresh + F8 address fixes |
   | `rotation.md` | Rotation AND Re-seal (content update) — one file, two entry sections; sequence: seal new → **verify new bundle** → distribute → update `KEEPSAKE_SHARE_G` → only then destroy old Recovery Sheet and collect/destroy old bundles; includes F11 (bundle lost → re-issue; rotate if theft suspected) and F13 (sheet destroyed → `keepsake guide` regeneration path vs full rotation) |
   | `restore-vault.md` | F10 lost machine; when rotation is mandatory |
   | `lost-signing-key.md` | F9 new key → allowed_signers → `trigger update` → push → verify checkin |
   | `decommission.md` | Retire safely: disable workflows → delete secrets → shred sheets → collect/destroy bundles → delete repo → tell recipients |
3. Failure-mode index `docs/runbooks/failure-mode-index.md`: table F1–F15 → "handled automatically (issue N)" or runbook#section link. Completeness test (normative): a Go docs-test parses `docs/DESIGN.md` §13.1 for IDs matching `^F[0-9]+$` and asserts each appears as a row in the index file.
4. Tutorial: narrative first-run with expected output snippets, kept honest by the QA gate run.
5. QA gate (normative execution plan): every runbook executed once and logged —
   - real: initial-setup, annual-drill (real private repo + owner-controlled recipient addresses; secrets set manually by owner)
   - scratch-fixture: rotation (second scratch vault+repo), restore-vault, lost-signing-key, decommission (executed against the scratch setup, then cleaned)
   - Log `docs/runbooks/qa-log-v1.md`: per-step timestamp + outcome + doc corrections made. **Redaction rules**: no share texts, no P, no mail-auth code, no SMTP values, no tokens; emails as `h***@…`; run URLs allowed. A grep test over the committed log enforces the deny-list (planted-marker canary in the template).
6. Known-gap notes: Windows e2e coverage (27), KU-5 with BOTH flows spelled out — macOS Gatekeeper (right-click → open → 開く) and Windows SmartScreen (詳細情報 → 実行) — reusing the canonical wording from `internal/content` (09).
7. Link check: `lychee` action pinned by SHA with config (`.lychee.toml`: exclude localhost/example.com, 10s timeout, retry 2) added to CI docs job — coordinated as an extension of 02's workflow, listed in scope here.

## Acceptance Criteria

- [ ] All files present; §14-mapping table complete; failure-index completeness test green.
- [ ] `qa-log-v1.md` committed with every runbook logged (real or scratch as specified) and redaction grep green.
- [ ] README quickstart reproduced verbatim by the tutorial run; JA/EN parity review by owner recorded.
- [ ] KU-5 both-OS wording present and sourced from the content pack.
- [ ] lychee job green.

## Validation

The QA gate execution IS the validation; docs tests (index completeness, link check, log redaction) run in CI.

## Dependencies

26, 27, 28, 29.

## Non-goals

Recipient-facing guide content (09/16), marketing site, localization beyond ja/en, editing 02's non-docs CI jobs.

## Design References

DESIGN §13.1 (F1–F15), §14, §3.1 item 6, §10.3 rule 6; research 01 (the two paradoxes).
