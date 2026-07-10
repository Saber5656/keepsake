# Title

`keepsake trigger init`: private trigger repository scaffolding

## Summary

Implement `keepsake trigger init`: generate the complete trigger-repo working directory — `keepsake.yaml` derived from vault config, the two pinned workflows, the vendored linux binary with checksum, ignore rules, and a README with the manual GitHub setup + secrets checklist.

## Context

DESIGN §4.2/§11: the trigger repo is a separate private repo the owner creates manually. Scaffolding must make the manual steps (repo creation, secrets) impossible to get silently wrong — the checklist and monitor-side validations (23) are paired defenses.

## Scope

- `internal/trigger/scaffold.go`, embedded workflow templates (+ tests), command wiring.

## Detailed Requirements

1. `keepsake trigger init` writes to `trigger.local_path` (must be empty or `--force` for regeneration; never touches `.git/`):
   - `keepsake.yaml` per §6.5 derived from vault config (recipients/timing/mail/owner/allowed_signers, `binary{path,sha256,version}`)
   - `.github/workflows/monitor.yml` + `checkin-button.yml` — exactly the DESIGN §11.1/§11.2 shape; `actions/checkout` pinned to a full commit SHA recorded in an embedded constants file with version comment; cron minute randomized per-scaffold within 17–43 to spread load (documented)
   - `bin/keepsake-linux-amd64` vendored via `dist.Ensure` (13) or `--tools-dir`; plus `bin/keepsake-linux-amd64.sha256` (format: `<hex>  bin/keepsake-linux-amd64`)
   - `.gitignore` (nothing secret is ever generated here; ignore OS junk) and a `.keepsake-forbidden` marker documenting the never-commit list (kit.age, share-R*, any payload) — the monitor's forbidden scan (23) is the enforcement
   - `state/` with empty `state.json` skeleton (schema 1, phase ACTIVE, null timestamps)
   - `README-switch.md`: numbered manual checklist — create PRIVATE repo `<trigger.repo>` (UI or `gh repo create <name> --private`), push scaffold, set secrets `KEEPSAKE_SHARE_G` / `KEEPSAKE_SMTP_URL` (+ optional secondary) via UI or `gh secret set` (commands shown for copy-paste, NEVER executed by keepsake), enable Actions if org-restricted, run `checkin-button` once from phone to bookmark it, then `keepsake checkin`, then drill. Each step has a verification line ("you should now see …").
2. Idempotent regeneration: `--force` rewrites generated files but preserves `state/` and refuses if `git status` shows uncommitted changes in generated paths [E-TRG-DIRTY].
3. Validation pass at the end: run trigger-config validation (07) on the generated `keepsake.yaml`; verify vendored binary hash matches `keepsake.yaml`.
4. Output: file list + THE CHECKLIST rendered to terminal too (Printer, bilingual).
5. Sanity guard: refuse to scaffold inside the vault or inside this OSS repo checkout (path containment check) [E-TRG-PATH].

## Acceptance Criteria

- [ ] Scaffold on empty dir produces the exact tree; `keepsake.yaml` passes validation; workflows parse as YAML and contain pinned SHAs + least-privilege permissions blocks from §11.1/§11.2 verbatim (structural asserts).
- [ ] Binary vendored with matching `.sha256`; `sha256sum -c` passes in test.
- [ ] `--force` preserves `state/state.json` contents; dirty-tree refusal works.
- [ ] Vault-path / OSS-repo-path scaffolding refused.
- [ ] Manual E2E (recorded in PR): real private repo created from the scaffold, secrets set by owner, `checkin-button` run from phone, monitor run green with `--dry-run` first, then real run commits heartbeat.

## Validation

Unit + structural tests in CI; the manual E2E transcript (with screenshots of the Actions runs) attached to the PR is the acceptance gate.

## Dependencies

07, 23 (also uses `internal/dist` from 13).

## Non-goals

Automating repo creation or secret setting (owner-manual by policy), `trigger update` (25), drill (26), multi-substrate (v2).

## Design References

DESIGN §4.2, §6.5, §11.1–11.3, §10.2 (GitHub boundary), §10.3 rule 6; ADR-003; research 03 C6/C7.
