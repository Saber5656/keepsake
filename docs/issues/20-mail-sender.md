# Title

Mail composer and SMTP sender with provider failover

## Summary

Implement `internal/mail`: strict SMTP-URL parsing, one-message-per-recipient composition with header-injection defenses and verified TLS, the template-registry interface (content lands in 21), go-mail delivery with per-message retry and primary/secondary failover, and sanitized per-recipient outcomes.

## Context

Mail is the only push channel to humans (ADR-006). Its failure handling feeds state retries (F2); its parsing is a §10.4 boundary; owner BCC is the A6 detection mechanism; TLS carries share_G at release.

## Scope

- `internal/mail/url.go`, `compose.go`, `send.go`, `registry.go`, `_test.go`

## Detailed Requirements

1. `ParseSMTPURL(s string) (Endpoint, error)`: schemes `smtps` (implicit TLS) and `smtp+starttls` ONLY (`smtp://` → `E-MAIL-SCHEME`); host, port, non-empty username AND password all required (percent-decoded) [`E-MAIL-URL`]; `Endpoint.String()` renders `smtps://<redacted>@host:port`.
2. TLS policy (normative): certificate verification ALWAYS on (no InsecureSkipVerify anywhere; lint-guarded comment); ServerName = URL host; for `smtp+starttls`, STARTTLS is mandatory before AUTH (go-mail `TLSMandatory`); an optional CA override for tests only via `Endpoint.TestRootCAs *x509.CertPool` populated from `KEEPSAKE_SMTP_TEST_ROOTCA` (path to PEM) — documented as test/e2e hook.
3. Registry + compose:
   ```go
   type TemplateID string
   type Audience string // "owner" | "recipient"
   func Register(id TemplateID, audience Audience, subjectJA, bodyJA string) // EN footer embedded in ja bodies where required (21)
   type ComposeInput struct {
       ID TemplateID; Data map[string]string
       To string                      // EXACTLY one address
       From, SubjectPrefix string; ReplyTo, OwnerBCC string // OwnerBCC applied iff audience==recipient
       Drill bool                     // [DRILL] prefix + explanation block
   }
   func Compose(in ComposeInput) (*Msg, error)
   ```
   - Rendering: `text/template`, `missingkey=error` (unknown/unfilled placeholder → error naming the template).
   - Header-bound values (From/To/ReplyTo/Subject incl. rendered prefix) re-validated at compose: addr-spec/mailbox rules + CR/LF rejection (`E-MAIL-HDR`) — defense in depth over config validation. Body data is NOT CR/LF-restricted (multi-line `{{error_list}}` legal).
   - Audience table is data, not convention: `Register` stores it; `Compose` enforces OwnerBCC presence for recipient audience and absence for owner audience.
4. `Send(primary Endpoint, secondary *Endpoint, msgs []*Msg) []SendResult` with `SendResult{To string; OK bool; Provider string /* "primary"|"secondary" */; Err error /* sanitized */}`:
   - Per MESSAGE: attempt primary → on failure wait 30s (context-aware) → retry primary → failover to secondary (same single retry) → record failure.
   - Batch continues past failures; one SMTP connection per endpoint reused across the batch when possible.
   - Sanitization: `Err` never contains URL userinfo (unit-tested with credentialed fixture errors).
5. go.mod: add `github.com/wneessen/go-mail` (allowlisted).
6. No logging; timeouts: 30s dial/command per attempt.

## Acceptance Criteria

- [ ] URL table: valid smtps/starttls; missing user, missing pass, missing port, plaintext scheme, CR/LF in creds → exact codes; redaction verified in String() and errors.
- [ ] TLS: STARTTLS-refusing server → fail (no cleartext AUTH — asserted via test server transcript); bad cert → fail; test-CA override path succeeds against self-signed fixture server.
- [ ] Header-injection fixtures (`\r\nBcc:` in name/subject data) → `E-MAIL-HDR`.
- [ ] One-To rule: recipient-audience Compose with OwnerBCC asserted present; owner-audience with BCC asserted absent; two-address To impossible by API shape.
- [ ] Failover matrix: primary down → secondary used per message; both down → all results failed, no panic, batch complete.
- [ ] Strict-render negative per registered template shape.
- [ ] Secret-hygiene: planted password `ZZSECRETZZ` never appears in any SendResult.Err/String output.

## Validation

CI with local SMTP fixture servers (plain-refusing, STARTTLS, smtps self-signed); real-provider smoke happens in the drill (26).

## Dependencies

07 (address rules reuse).

## Non-goals

Template CONTENT (21), send scheduling/dedup (19/23), HTML mail (rejected, ADR-006), inbound mail.

## Design References

DESIGN §12 (one message per recipient, verified TLS), §10.4, §10.5 A6; ADR-006.
