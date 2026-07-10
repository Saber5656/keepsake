# Title

Mail composer and SMTP sender with provider failover

## Summary

Implement `internal/mail`: SMTP-URL parsing, plain-text mail composition with header-injection defenses, template rendering interface, go-mail-based delivery with retry and primary/secondary failover, and per-recipient outcome reporting.

## Context

Mail is the only push channel to humans (ADR-006). Its failure handling feeds state.json retries (F2); its input validation is a §10.4 boundary; owner BCC on recipient mail is the A6 detection mechanism.

## Scope

- `internal/mail/url.go`, `compose.go`, `send.go`, `templates.go` (interface + rendering; content arrives in 21), `_test.go`

## Detailed Requirements

1. `ParseSMTPURL(s string) (Endpoint, error)`: schemes `smtps` (implicit TLS) and `smtp+starttls` ONLY (plaintext smtp:// rejected E-MAIL-SCHEME); userinfo required; port required; percent-decoding for credentials; never include credentials in Endpoint.String()/errors [E-MAIL-URL].
2. `Compose(t TemplateID, lang string, data map[string]string, from, to []string, ...) (*Msg, error)`: text/plain UTF-8; subject = configured prefix + template subject; ALL header-bound values re-validated (addr-spec, no CR/LF) even though config validated earlier (defense in depth); body from template registry.
3. Template registry interface: `Register(id TemplateID, subjectJA, subjectEN, bodyJA, bodyEN string)` with `text/template` rendering; missing-placeholder = error at render (strict, `missingkey=error`); registry populated by issue 21's embedded content; DRILL variants = same ID + `Drill bool` flag adding `[DRILL]` prefix + explanation header block.
4. `Send(ep Endpoint, epSecondary *Endpoint, msgs []*Msg) []SendResult`: go-mail client, 30s timeout per attempt, one retry after 30s on the primary, then secondary (same retry), per-message `SendResult{To, OK, Provider, Err}`; partial failures do not abort the batch.
5. Owner BCC: composer adds owner email as BCC on every recipient-bound template kind (flag on TemplateID metadata).
6. Secrets: endpoints redacted in all errors/logs; `String()` prints `smtps://<redacted>@host:port`.
7. go.mod: add `github.com/wneessen/go-mail` (allowlisted).

## Acceptance Criteria

- [ ] URL parsing table: valid smtps/starttls, missing user/port, plaintext scheme, CR/LF in creds → exact codes; redaction verified.
- [ ] Header-injection attempts via template data (name containing `\r\nBcc:`) neutralized (rejected at compose).
- [ ] Failover test against two local SMTP test servers (use go-mail's test hooks or a minimal SMTP fixture): primary down → secondary used; both down → all results failed, no panic.
- [ ] BCC-to-owner asserted on recipient templates; absent on owner templates.
- [ ] Strict rendering: unknown/missing placeholder fails, with template ID in error.

## Validation

CI with local SMTP fixture (e.g. embedded test server on 127.0.0.1 ephemeral port); real-provider smoke happens in the drill (26).

## Dependencies

07 (address rules reuse).

## Non-goals

Template CONTENT (21), send scheduling/dedup (19/23 own cadence), HTML mail (rejected, ADR-006), inbound mail.

## Design References

DESIGN §12, §10.4, §10.5 A6; ADR-006.
