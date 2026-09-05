# Title

`keepsake trigger init`: private trigger repository scaffolding

## Summary

Implement `keepsake trigger init` in `internal/trigger`: generate the complete trigger-repo working directory — `keepsake.yaml` (with `repo`), the two workflows, the vendored linux binary with checksum, the initial `state.json`, ignore/forbidden markers, and a verification-lined manual checklist README.

## Context

DESIGN §4.2/§11: the trigger repo is a separate private repo the owner creates manually. The scaffold pairs with monitor-side runtime validation (23) so manual steps cannot fail silently.

## Scope

- `internal/trigger/scaffold.go`, embedded workflow templates + a pinned-SHA constants file (+ tests), command wiring.

## Detailed Requirements

1. Generated tree (exact; golden-manifest tested with modes):
   | Path | Mode | Content |
   |---|---|---|
   | `keepsake.yaml` | 0644 | §6.5 fields incl. top-level `repo:` derived from vault `trigger.repo`; passes `config.LoadTrigger` |
   | `.github/workflows/monitor.yml` | 0644 | DESIGN §11.1 shape; allowed substitutions ONLY: cron minute (deterministically derived `17 + (fnv32(repo) % 27)` → 17–43), checkout action SHA from the constants file |
   | `.github/workflows/checkin-button.yml` | 0644 | DESIGN §11.2 verbatim |
   | `bin/keepsake-linux-amd64` | 0755 | via `dist.Ensure(version, []Platform{{"linux","amd64"}})` or `--tools-dir` |
   | `bin/keepsake-linux-amd64.sha256` | 0644 | `<hex>  bin/keepsake-linux-amd64` |
   | `state/state.json` | 0644 | initial schema-19 JSON verbatim: `{"schema":1,"phase":"ACTIVE","initialized_at":"<now RFC3339 UTC>","armed":false,"accepted_counter":0,"last_valid_checkin":null,"last_checkin_channel":null,"last_run_at":null,"last_remind_sent_at":null,"last_alert_sent_at":null,"last_unarmed_warning_at":null,"pause_active":false,"invalid_sig_warned":false,"release":{"released_at":null,"sent":{},"confirm":{},"last_alive_mail_at":null},"history":[]}` |
   | `.gitignore` | 0644 | OS junk only |
   | `.keepsake-forbidden` | 0644 | documents the exact forbidden globs enforced by monitor: `kit.age`, `*.age`, `share-R*` |
   | `README-switch.md` | 0644 | manual checklist (below) |
2. `--tools-dir` accepts either dist-layout names (`keepsake_<ver>_linux_amd64`) or the plain `keepsake-linux-amd64`; hash recorded into both `keepsake.yaml` and the `.sha256` file from the actual vendored bytes.
3. README-switch.md checklist: numbered steps each with a "確認:" verification line — create PRIVATE repo `<repo>` (`gh repo create <repo> --private` or UI); `git init/remote/push` commands; set secrets `KEEPSAKE_SHARE_G` / `KEEPSAKE_SMTP_URL` (+ optional secondary) via UI or `gh secret set NAME --repo <repo>` (values pasted at the prompt — commands shown WITHOUT values, never executed by keepsake); enable Actions if restricted; run `checkin-button` once from the phone and bookmark it; run `keepsake checkin`; run monitor `--dry-run` from Actions; run `keepsake drill`. The same checklist is printed to the terminal (Printer, bilingual).
4. Target rules: `trigger.local_path` must be empty OR `--force`; `--force` = scaffold repair for the SAME `binary_version` only (version differs → error pointing to `trigger update` [`E-TRG-VERSION`]); when the dir is a git repo, refuse if generated paths are dirty (`git status --porcelain -- <generated paths>` non-empty) [`E-TRG-DIRTY`]; non-git dir → no dirty check. `state/` is NEVER overwritten by `--force`.
5. Containment guards [`E-TRG-PATH`]: refuse to scaffold inside the vault, inside this OSS repo checkout, or inside any parent that contains `payload/` (plaintext proximity).
6. Post-generation validation: `config.LoadTrigger` on the generated file must pass; vendored binary hash must equal both records; workflows parse as YAML with the §11.1/§11.2 permission blocks asserted structurally.
7. Secret/PII discipline: generated tree + stdout contain no `ssh_signing_key` path, no `trigger.local_path`, no share material, no SMTP values; recipient emails DO appear in `keepsake.yaml` by design (A-7 boundary) — grep test with planted values covers the forbidden set.

## Acceptance Criteria

- [ ] Golden tree manifest (paths, modes, content hashes for static files) matches exactly; initial state.json byte-equal to the spec (timestamp injected).
- [ ] Cron-minute derivation deterministic per repo name and within 17–43.
- [ ] `--force` repairs a deleted workflow file, preserves `state/` byte-identical, refuses on dirty generated paths and on version mismatch.
- [ ] Containment fixtures (vault path, OSS repo path, payload-adjacent) → `E-TRG-PATH`.
- [ ] tools-dir both layouts accepted; hash parity asserted; `sha256sum -c` passes.
- [ ] Planted-secret grep over tree + captured stdout.
- [ ] Manual E2E recorded in PR: real private repo from the scaffold, secrets set by owner manually, phone `checkin-button` run, `monitor --dry-run` green in Actions, then a real monitor run committing the heartbeat (screenshots + run URLs).

## Validation

Unit + golden tests in CI; the manual E2E transcript is the acceptance gate.

## Dependencies

07, 13 (`internal/dist`), 23.

## Non-goals

Automating repo creation or secrets (owner-manual by policy), `trigger update` (25), drill (26), multi-substrate (v2).

## Design References

DESIGN §4.2, §6.5 (`repo` field), §6.8 (initial state), §11.1–11.3, §10.2, §10.3 rule 6; ADR-003; research 03 C6/C7.
