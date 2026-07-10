# Title

End-to-end timeline integration test: seal → silence → release → open

## Summary

Build the whole-product integration test: with a fake clock, fixture trigger repo, stub GitHub API, and capture SMTP server, drive the real binaries through seal → check-ins → silence → REMINDING → ALERTING → RELEASED → recipient `open` using the share_G extracted from the captured release mail.

## Context

ISSUE_PLAN §6 integration layer: proves the issues compose into the actual product story (DESIGN §3.1 completion items 1–4) and guards against regressions that unit layers cannot see (e.g. share encoding drift between monitor mail and open wizard).

## Scope

- `e2e/` package (build tag `e2e`), `Makefile` target `e2e`, CI job (linux only).

## Detailed Requirements

1. Harness components (all in-process or local): tmp vault; tmp bare git repo + local clone as trigger repo; httptest GitHub API stub (dispatch runs feed); local SMTP capture server; `--now` injection for monitor; `keepsake` invoked as a compiled binary via `os/exec` (NOT in-process) to test the real CLI surface.
2. Script (assert after every step):
   1. `init` → fill payload from fixture (3 files incl. 日本語 filename) → `seal --tools-dir` (prebuilt fixture binaries) → capture printed share_G from stdout (test-only parse)
   2. set env share_G for monitor from the captured value (simulating the manual secret step)
   3. `trigger init --tools-dir` into the clone; commit+push
   4. `checkin` twice over simulated days 0 and 10 (git-level verification of counters)
   5. run monitor daily via `--now` day 10→50: assert exact phase transitions and mail sequence (subjects, recipients, BCC) matching DESIGN §9.2; assert heartbeat commits exist per run
   6. day 42+: extract share_G string from the captured release mail body with the SAME regex documented for humans (the sharetext format §7)
   7. `open --kit --share-r-file --share-g <extracted> --out` → extracted tree byte-identical to original payload; exit 0
   8. post-release: `checkin` again + monitor → `released_but_alive` mail asserted
3. Negative sub-scenario: corrupt one byte of kit.age → `open` exits 4 with E-OPEN guidance.
4. Runtime budget ≤ 3 min in CI (compile once, reuse binary; daily loop only evaluates ~40 monitor invocations).
5. Flake policy: zero tolerance — no sleeps; all waits are deterministic (in-process servers).

## Acceptance Criteria

- [ ] Full script green in CI on linux; runnable locally via `make e2e`.
- [ ] Mail sequence assertion covers counts + ordering + `[DRILL]` absence.
- [ ] The release-mail share extraction uses only information a human recipient would have (format regex from §7 + CHECK validation) — documented in test comments.
- [ ] Failure of any single prior issue's contract (mutation-tested by intentionally breaking sharetext CHECK once during development) is caught — note the exercise in PR.

## Validation

This IS the validation layer; CI gate from merge onward. Keep in required checks (31).

## Dependencies

13, 14, 23 (uses nearly everything transitively).

## Non-goals

Real GitHub/SMTP (drill covers that, 26), performance/load testing, Windows e2e (cross-platform unit coverage suffices in v1 — document as known gap in 30).

## Design References

DESIGN §3.1, §9.2; ISSUE_PLAN §6.
