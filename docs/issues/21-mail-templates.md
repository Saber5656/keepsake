# Title

Mail template content: all message kinds in Japanese/English with drill variants

## Summary

Author and register the full mail template set (DESIGN §12) using the content-pack wording (09) and the registry interface (20): reminders, reset confirmation, recipient safety-check alert, release, post-release confirms, released-but-alive, config-error — each with `[DRILL]` variants.

## Context

These texts are what a family member actually receives at release time; clarity, tone, and anti-phishing framing (A7) are product-critical. Kept separate from sender machinery so a human can review all wording in one PR.

## Scope

- `internal/mail/content/*.tmpl` (embedded), `internal/mail/register.go`, tests.

## Detailed Requirements

1. Template IDs (exact): `remind_owner`, `checkin_reset`, `alert_recipients`, `release`, `postrelease_confirm`, `released_but_alive`, `config_error_owner`. Each: subject+body, ja+en, plain text, ≤78-char soft wrapping.
2. Required content per template (from DESIGN §9/§12):
   - `remind_owner`: days since last check-in, days remaining until recipient alert AND until release, exact check-in instructions (CLI one-liner + phone button path), pause pointer.
   - `alert_recipients`: 「<owner name> さんの keepsake が <N> 日間応答を確認できていません。まずご本人に連絡してください」; explicitly states NO action needed on the kit yet; what happens next and when (release date); owner BCC'd.
   - `release`: what happened; the share_G sharetext on its own line; the CHECK-group verification sentence referencing the printed guide; `keepsake open` steps (3 numbered lines); 「このメールの文字列を誰かに送り返す必要は絶対にありません」 warning; helper note; owner BCC'd.
   - `postrelease_confirm`: short re-notice + same share line (re-send insurance F8).
   - `released_but_alive`: rotation instruction summary + runbook link placeholder.
   - `config_error_owner`: validation error list verbatim + fail-closed statement ("no release actions will run until fixed").
3. Placeholders strictly from a documented set (extends PLACEHOLDERS.md from 09): `{{owner_name}} {{recipient_name}} {{days_silent}} {{release_date}} {{share_g}} {{check_group}} {{checkin_days}} {{error_list}} {{drill_notice}}` — registry test asserts each template uses only known placeholders and `release`/`postrelease_confirm` are the ONLY ones referencing `{{share_g}}`.
4. Drill variants: automatic `[DRILL]` subject prefix + leading block 「これは訓練です。実際の開封ではありません」 (from 20's Drill flag) — templates must read correctly with the block injected; `release` drill uses placeholder share text supplied by caller (26), never real.
5. Sender display: `mail.from` config used verbatim; templates reference it so recipients can pin the expected sender (A7).

## Acceptance Criteria

- [ ] Golden rendered outputs (ja+en × normal+drill) for every ID with fixture data, committed and reviewed.
- [ ] Placeholder audit test green (incl. share_g restriction).
- [ ] Japanese native review recorded in PR (owner).
- [ ] Render with missing placeholder fails (inherits 20 strictness) — one negative test per template.

## Validation

CI golden tests; human wording review; end-to-end appearance verified during drill (26).

## Dependencies

20 (registry interface), 09 (wording sources).

## Non-goals

Delivery mechanics (20), when-to-send logic (19/23), HTML mail.

## Design References

DESIGN §12, §9.1, §10.5 A7; ADR-006.
