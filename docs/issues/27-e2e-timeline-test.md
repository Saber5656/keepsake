# Title

End-to-end timeline integration test: seal → silence → release → open

## Summary

Build the whole-product integration test: with injected time, a local bare trigger repo, an httptest GitHub API stub, and a STARTTLS capture SMTP server, drive the REAL compiled binary through init → seal → trigger init → checkin → daily monitor runs → release → recipient `open` using the share_G extracted from the captured release mail.

## Context

ISSUE_PLAN §6 integration layer: proves the issues compose into the product story (DESIGN §3.1 items 1–4, including the dispatch check-in channel) and guards cross-issue contracts (share encoding in mail ↔ open wizard).

## Scope

- `e2e/` package (build tag `e2e`), `Makefile e2e` target, CI job (linux; listed among required checks by issue 31).

## Detailed Requirements

1. Harness (all local/in-process; binary invoked via `os/exec`):
   - Vault in t.TempDir; payload fixture EXACT: `00-README-FIRST.md` (fixed bytes), `10-accounts.md` (fixed bytes), `写真リンク.md` (fixed bytes, multibyte name); expected SHA-256s committed.
   - Tools fixture: 4 files with the §6.3 names whose contents are small fixed markers (not real binaries) — sufficient for bundle hashing; documented.
   - Trigger repo: `git init --bare` + working clone; `git config url."<bare path>".insteadOf "https://github.com/e2eowner/keepsake-switch.git"` in the clone so issue 18's origin validation passes with `trigger.repo=e2eowner/keepsake-switch` (exact commands in the harness).
   - GitHub API stub: httptest server via `KEEPSAKE_GH_API_BASE`; serves empty dispatch runs by default and one owner-actor dispatch run when the scenario injects it.
   - SMTP: local `smtp+starttls` capture server with a self-signed cert trusted via `KEEPSAKE_SMTP_TEST_ROOTCA`; `KEEPSAKE_SMTP_URL=smtp+starttls://user:pass@127.0.0.1:<port>`.
   - Monitor env: `KEEPSAKE_SHARE_G` read from `state/seal-manifest.json.share_g_text` (NOT stdout parsing), `GITHUB_TOKEN=e2e-dummy`, `GITHUB_REPOSITORY` unset (local mode).
2. Timeline (all days ABSOLUTE from t0; check-ins on days 0 and 10 ⇒ elapsed resets; expected-mail table is committed as the assertion source):
   - Days 10..52+: daily `monitor --now <t0+D>`; asserted per §9.2 relative to the day-10 check-in: reminders elapsed days 14,17,…, safety-check alerts at elapsed 28,35 (per recipient), release at elapsed 42 = absolute day 52; postrelease confirms +7d.
   - Mid-timeline (absolute day 30): API stub serves a dispatch run by the owner actor dated day 30 → monitor resets to ACTIVE (covers DESIGN §3.1 item 3); silence resumes (stub returns nothing new) and the clock re-derives from day 30 (release at absolute day 72 — the table continues from there).
   - Assertions per run: phase in state.json, heartbeat commit exists, mail sequence (template ID via subject, recipient, owner BCC) from SMTP capture.
3. Release-mail extraction: regex committed in the test = `KEEPSAKE-G1-[0-9A-HJKMNP-TV-Z-]+` (Crockford, no I/L/O/U) — the same grammar a human uses (§7); decoded share must pass role-G validation and CHECK; then `open --kit ... --share-r-file ... --share-g <extracted> --out ...` → extracted tree byte-identical to fixture payload; exit 0.
4. Post-release: `checkin` + next monitor run → `released_but_alive` mail, phase stays RELEASED, no new release mail, T6 weekly dedup asserted over two more runs.
5. Negative scenarios: (a) corrupt final byte of `kit.age` → `open` exit 4 (wrong-passphrase-or-corrupt path); (b) mutate one char inside the extracted share's CHECK group → `open` exit 3 with `E-SHARE-CHK` message ID (committed negative, not a dev-only exercise).
6. Boundary assertions: trigger repo tree never contains `kit.age`/`*.age`/`share-R*`; planted SMTP password + share_G appear nowhere in commits/state/logs except release/confirm mail bodies in the SMTP capture.
7. Runtime ≤ 3 min in CI (compile once; ~65 monitor invocations); zero sleeps (deterministic waits only).

## Acceptance Criteria

- [ ] Full script green in CI and via `make e2e`; expected-mail table drives the assertions.
- [ ] Dispatch-reset scenario proves the phone channel path end-to-end against the API stub.
- [ ] Both negative scenarios with exact exit codes/messages.
- [ ] Boundary greps green; byte-identical payload round-trip incl. multibyte filename.

## Validation

This IS the validation layer; issue 31 adds it to required checks.

## Dependencies

13, 14, 18, 23, 24.

## Non-goals

Real GitHub/SMTP (drill, 26), performance testing, Windows e2e (documented gap, 30).

## Design References

DESIGN §3.1, §7 (extraction regex grammar), §9.2, §11.3; ISSUE_PLAN §6.
