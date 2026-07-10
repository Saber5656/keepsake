# Title

Minimal GitHub API client for workflow_dispatch check-in verification

## Summary

Implement `internal/gh`: a tiny authenticated GitHub REST client that lists workflow runs of `checkin-button.yml` and returns dispatch check-in facts (`triggering_actor`, `created_at`), with strict response validation and bounded pagination.

## Context

The phone check-in channel's authenticity rests on Actions API run records, which cannot be forged via repo contents (research 03 C6; ADR-005 channel 2). This is the only GitHub API surface in v1 — keep it minimal, stdlib-only.

## Scope

- `internal/gh/client.go`, `runs.go` (+ tests with httptest fixtures)

## Detailed Requirements

1. `New(baseURL, token, repo string) *Client` — baseURL injectable for tests (default `https://api.github.com`); token from env (caller passes; client never reads env itself).
2. `DispatchRuns(ctx, workflowFile string, since time.Time) ([]Run, error)` calling `GET /repos/{repo}/actions/workflows/{workflowFile}/runs?event=workflow_dispatch&created=>{since RFC3339}` with `per_page=100`, following pagination max 5 pages [E-GH-PAGES beyond].
3. `Run{ID int64, CreatedAt time.Time, TriggeringActor string, Conclusion string}` — parse defensively: required fields missing → E-GH-SCHEMA; ignore unknown fields; actor from `triggering_actor.login`.
4. HTTP hygiene: 10s timeout per request, `Accept: application/vnd.github+json`, `X-GitHub-Api-Version` pinned to a current dated version, retry once on 5xx/429 honoring `Retry-After` (cap 30s), NO retry on 4xx; auth header never logged; errors include status + request path only.
5. Rate-limit awareness: surface remaining-quota header in a debug field, but the monitor makes ≤2 calls/run — no complex handling.
6. Filtering (`triggering_actor == owner login`, case-insensitive) is the CALLER's job (23) — client returns raw facts; document why (testability, single responsibility).
7. Stdlib only (net/http, encoding/json).

## Acceptance Criteria

- [ ] httptest fixture suite: happy path, pagination (2 pages), missing fields, 500-then-200 retry, 429 with Retry-After, 404 (workflow absent → typed ErrWorkflowNotFound for first-run tolerance), malformed JSON.
- [ ] `since` parameter correctly encoded; ctx cancellation honored.
- [ ] Token absent from all error strings (grep test).
- [ ] No new go.mod requirements.

## Validation

CI unit tests; live-API behavior exercised in issue 24 validation (real trigger repo) and drill.

## Dependencies

01.

## Non-goals

Any write API, secrets API (owner sets secrets manually — never automated), GraphQL, other endpoints (issues/releases downloads live in `internal/dist`, issue 13).

## Design References

DESIGN §11.2, §11.3 step 3, §10.4 (API boundary); ADR-005; research 03 C6.
