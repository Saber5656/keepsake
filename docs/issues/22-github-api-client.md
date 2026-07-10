# Title

Minimal GitHub API client for workflow_dispatch check-in verification

## Summary

Implement `internal/gh`: a small authenticated GitHub REST client that lists workflow runs for a workflow file and returns dispatch facts (`triggering_actor`, `created_at`), with strict response validation, Link-header pagination with a hard cap, bounded retries, and injectable base URL. Issue 26 later adds a latest-run lookup to this package (declared extension point).

## Context

The phone check-in channel's authenticity rests on Actions API run records (research 03 C6; ADR-005 channel 2). A missed run record can mean a false release, so partial results are never returned silently.

## Scope

- `internal/gh/client.go`, `runs.go` (+ httptest suite)

## Detailed Requirements

1. Constructor: `New(baseURL, token, repo string) (*Client, error)` — `baseURL` injectable (default `https://api.github.com`; e2e overrides); token REQUIRED non-empty [`E-GH-AUTH`]; `repo` validated `owner/name`.
2. `DispatchRuns(ctx context.Context, workflowFile string, since time.Time) (Result, error)`:
   ```go
   type Run struct { ID int64; CreatedAt time.Time; TriggeringActor string; Conclusion *string }
   type Result struct { Runs []Run; RateLimitRemaining *int }
   var ErrWorkflowNotFound = errors.New(...) // 404: broken setup — HARD error (checkin-button.yml is mandatory scaffold)
   ```
   - Request: `GET /repos/{repo}/actions/workflows/{workflowFile}/runs` with `url.Values{"event":"workflow_dispatch","created":">=" + since.UTC().Format(time.RFC3339),"per_page":"100"}`; `workflowFile` must match `^[A-Za-z0-9._-]+$` (single path segment) [`E-GH-WFNAME`] and is path-escaped.
   - Headers (asserted in tests): `Authorization: Bearer <token>`, `Accept: application/vnd.github+json`, `X-GitHub-Api-Version: 2022-11-28` (literal).
   - Pagination: follow the `Link: rel="next"` header, max 5 pages; if a next link remains after page 5 → `E-GH-PAGES` error with NO partial result.
   - Empty `workflow_runs` on 200 is a normal result (no runs yet).
   - **Run acceptance semantics documented in the package**: ANY run record with matching `triggering_actor` counts as a check-in regardless of `status`/`conclusion` (ADR-005 — the server-side dispatch record is the fact; `Conclusion` is informational and may be null). Actor filtering itself is the caller's job (issue 23) — client returns raw facts.
3. Response validation: top-level `workflow_runs` array required; per-run required fields `id`, `created_at`, `triggering_actor.login` — missing/mistyped → `E-GH-SCHEMA`; unknown fields ignored.
4. HTTP hygiene: per-request exchange timeout 10s (separate from retry sleeps, which are ctx-cancellable); retry ONCE on 5xx, 429, or transport error, honoring `Retry-After` (integer seconds or HTTP-date, capped 30s); 4xx (except 429) never retried; 401/403 → `E-GH-AUTH`-wrapped error. Errors carry status + path only — never the token (grep-tested).
5. Rate info: `X-RateLimit-Remaining` parsed into `Result.RateLimitRemaining` when present.
6. Stdlib only.

## Acceptance Criteria

- [ ] httptest suite: happy path; empty list; 2-page pagination via Link; 6-page fixture → `E-GH-PAGES` and no partial Runs; missing fields → `E-GH-SCHEMA`; 500→200 retry; 429 with both Retry-After forms; 404 → `ErrWorkflowNotFound`; 401 → auth error; malformed JSON; null conclusion preserved.
- [ ] Header assertions (all three + Bearer token) and `created`/`per_page` query encoding golden.
- [ ] ctx cancellation aborts both the exchange and a pending retry sleep.
- [ ] Token never appears in any error string (planted-token grep).
- [ ] No new go.mod requirements.

## Validation

CI unit tests; live-API behavior exercised in issue 24's manual E2E and the drill.

## Dependencies

01.

## Non-goals

Any write API, secrets API (owner sets secrets manually — never automated), GraphQL, release-asset downloads (`internal/dist`, issue 13), run-by-id lookup (added by issue 26 as a declared extension).

## Design References

DESIGN §11.2, §11.3 step 3, §10.4 (API boundary); ADR-005; research 03 C6.
