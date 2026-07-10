# Title

OSS repository hardening: rulesets, required checks, CodeQL, secret scanning

## Summary

Harden this public repository for OSS release: a main-branch ruleset with NO bypass actors (PR-only for everyone including the owner), exact required checks, CodeQL, secret scanning + push protection + private vulnerability reporting via their specific API endpoints, all scripted idempotently with read-back verification and a machine-readable owner checklist.

## Context

DESIGN §10.5 A8: this repo is the supply-chain root for a binary that later handles share_G and SMTP credentials in users' trigger repos. Owner policy: agents script; the owner reviews/executes permission-sensitive settings.

## Scope

- `scripts/repo-hardening.sh` (dry-run default, `--apply` to execute, read-back assertions), `.github/workflows/codeql.yml`, `docs/runbooks/repo-settings-checklist.md`.

## Detailed Requirements

1. Ruleset (target `main`) — exact JSON payload embedded in the script for `gh api repos/{owner}/{repo}/rulesets`:
   - `name: "protect-main"`, `enforcement: "active"`, `conditions.ref_name.include: ["~DEFAULT_BRANCH"]`
   - rules: `pull_request` (required_approving_review_count: 0 — single-maintainer; review comes from Codex/CodeRabbit bots pre-merge per owner workflow), `required_status_checks` with `strict_required_status_checks_policy: true` and contexts EXACTLY: `test-ubuntu`, `test-macos`, `lint`, `vuln`, `secrets`, `cross`, `audit`, `e2e` (names finalized by 02/27 — the script reads current check-run names via the API and fails with a diff if they drift), `non_fast_forward` (blocks force-push), `deletion`, `required_linear_history`
   - `bypass_actors: []` — NOBODY bypasses; emergency procedure = temporarily editing the ruleset, documented in the checklist with a "re-run this script afterwards" step.
2. `codeql.yml`: language go; triggers `push: [main]`, `pull_request`, weekly `schedule`; `permissions: {contents: read, security-events: write}`; actions pinned by SHA; the first default-branch scan URL is recorded in the checklist.
3. Security settings via their SPECIFIC endpoints (each: apply if `--apply`, then read back and assert):
   - Dependabot alerts: `PUT /repos/{o}/{r}/vulnerability-alerts` (assert 204 on GET)
   - Automated security fixes: `PUT /repos/{o}/{r}/automated-security-fixes`
   - Secret scanning + push protection: `PATCH /repos/{o}/{r}` with `security_and_analysis.secret_scanning.status=enabled` and `secret_scanning_push_protection.status=enabled` (read back via GET)
   - Private vulnerability reporting: `PUT /repos/{o}/{r}/private-vulnerability-reporting` (28's SECURITY.md claim depends on this — cross-referenced)
4. Repo metadata (normative values, applied by script): `has_wiki=false`, `has_projects=false`, description set, topics `["dead-mans-switch","digital-legacy","age-encryption","shukatsu"]`; fork-PR workflow approval = "require approval for all outside collaborators" (API if available, else checklist row with exact UI click-path).
5. Push-protection verification: read-back assertion is the acceptance; a live canary push is an OPTIONAL owner-manual step documented in the checklist (using GitHub's documented push-protection test pattern; agents never create or push real-looking credentials).
6. `docs/runbooks/repo-settings-checklist.md`: machine-readable table — `| item | status(✅/⬜) | evidence (URL/transcript path) | date |` — including the manual rows (repo creation settings, fork-PR approval if UI-only, optional canary, ruleset-emergency procedure). A docs test asserts no row's status is empty at v1 tag.
7. Script properties: idempotent (re-run produces "no drift"); dry-run prints the diff of every setting; uses ambient `gh` auth; never stores tokens; exits non-zero on unfixable drift.

## Acceptance Criteria

- [ ] Owner runs `scripts/repo-hardening.sh --apply`; committed transcript (tokens redacted) shows every read-back assertion green.
- [ ] Direct push to main rejected AND force-push rejected (demonstrated on scratch commits, transcript linked; no bypass actor exists).
- [ ] Required-check contexts verified against live check-run names (script diff empty after 02+27 are merged).
- [ ] CodeQL first default-branch scan completed; zero open alerts or each triaged with justification.
- [ ] All four security-setting read-backs assert enabled; checklist has no empty status rows.
- [ ] Re-run of the script reports no drift (idempotency).

## Validation

Read-back assertions + the two rejection demonstrations; annual re-run listed in the maintenance section of the checklist.

## Dependencies

02, 27.

## Non-goals

Org-level policies, CLA/DCO, release signing (v2), trigger-repo settings (24's checklist), SECURITY.md content (28).

## Design References

DESIGN §10.5 A8, §10.6; ISSUE_PLAN §6; owner global policy (permission-sensitive changes are human-executed).
