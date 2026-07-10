# Title

OSS repository hardening: rulesets, required checks, CodeQL, secret scanning

## Summary

Harden this public repository for OSS release: branch ruleset (PR-only main, required checks, no force-push), CodeQL scanning, secret scanning with push protection, and repo-settings verification — scripted with `gh api` where possible, documented as an owner checklist where permissions require a human.

## Context

DESIGN §10.5 A8: the OSS repo is the supply chain root for a binary that later handles share_G and SMTP credentials in users' trigger repos. Hardening must land before the first release announcement. Owner policy: agents may script, but the owner reviews/executes permission-sensitive settings.

## Scope

- `scripts/repo-hardening.sh` (idempotent `gh api` calls + read-back verification), `.github/workflows/codeql.yml`, `docs/runbooks/repo-settings-checklist.md`.

## Detailed Requirements

1. Ruleset (target `main`) via `gh api repos/{owner}/{repo}/rulesets`: require PR before merge (≥1 approval OR owner-only bypass documented — single-maintainer reality: require PR, allow repo-admin bypass, still block force-push/deletion), required status checks: `test`, `lint`, `vuln`, `secrets`, `e2e` (from 02/27), require linear history. Script prints current state, applies only diffs, exits non-zero on drift it cannot fix.
2. `codeql.yml`: language `go`, on PR + weekly schedule, pinned action SHAs, `security-events: write` only.
3. Secret scanning + push protection + private vulnerability reporting + Dependabot alerts: enable via API where the plan allows (`gh api repos/{o}/{r} -f security_and_analysis...`); each setting read back and asserted; anything requiring UI (e.g. plan-dependent features) goes into the checklist with exact click-paths.
4. Checklist doc also covers: repo description/topics, disable wiki/projects if unused, signed-releases note (v2), fork-PR workflow-approval setting = "require approval for all outside collaborators".
5. Script is idempotent, dry-run by default (`--apply` to execute), and never stores tokens (uses ambient `gh` auth).
6. Verify 02's CI + 27's e2e are actually listed as required checks (names must match job names — fix drift here).

## Acceptance Criteria

- [ ] `scripts/repo-hardening.sh --apply` run by the owner completes; read-back verification section all green (transcript committed, tokens redacted).
- [ ] Direct push to main rejected (demonstrated + reverted test), force-push rejected.
- [ ] CodeQL first scan completed with zero open alerts (or triaged with justifications).
- [ ] Push protection verified by attempting to push a canary secret on a scratch branch (blocked; transcript).
- [ ] Checklist items all checked in the committed doc.

## Validation

Read-back assertions in the script + the demonstrated rejection tests; recurring drift caught by re-running the script (documented as annual maintenance).

## Dependencies

02 (check names), 27 (e2e as required check — may be added later; script tolerates missing check with warning).

## Non-goals

Org-level policies, CLA/DCO setup, releases signing (v2), trigger-repo settings (private repo; covered by 24's checklist).

## Design References

DESIGN §10.5 A8, §10.6; ISSUE_PLAN §6 supply chain; owner global policy (secrets/permission changes are human-executed).
