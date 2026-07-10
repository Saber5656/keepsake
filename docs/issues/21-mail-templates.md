# Title

Mail template content: all message kinds in Japanese with drill variants

## Summary

Author and register the full mail template set (DESIGN §12) using the registry interface from issue 20: owner reminders, reset confirmation, recipient safety-check alert, release, post-release confirms, released-but-alive, config-error — Japanese bodies with English helper footers on recipient-bound mail, plus `[DRILL]` variants, and the placeholder-inventory extension.

## Context

These texts are what a family member actually receives at release time; clarity, tone, and the anti-phishing framing (A7: mail-auth code + fixed sender + never-send-back) are product-critical. Kept separate from sender machinery so a human can review all wording in one PR.

## Scope

- `internal/mail/content/<id>.subject.ja.tmpl` + `<id>.body.ja.tmpl` (embedded; recipient-bound bodies include a literal English-summary section), `internal/mail/register.go`, tests.
- Extends `internal/content/PLACEHOLDERS.md` (issue 09's inventory) with the mail placeholders — inventory update is in scope here.

## Detailed Requirements

1. Template IDs and audiences (registered per issue 20 §3; exact):
   | ID | Audience | Language shape |
   |---|---|---|
   | `remind_owner` | owner | ja |
   | `checkin_reset` | owner | ja |
   | `alert_recipients` | recipient | ja + EN footer |
   | `release` | recipient | ja + EN footer |
   | `postrelease_confirm` | recipient | ja + EN footer |
   | `released_but_alive` | owner | ja |
   | `config_error_owner` | owner | ja |
2. Required content per template (normative checklists; goldens enforce):
   - `remind_owner`: days silent, dates of next recipient-alert AND release, exact check-in steps (CLI one-liner + phone button path), pause pointer.
   - `checkin_reset`: accepted check-in time (JST rendering) + channel, confirmation the switch reset, next reminder threshold date, "no action needed".
   - `alert_recipients`: 「<owner名> さんの keepsake が {{days_silent}} 日間応答を確認できていません。まずご本人に連絡してください」; explicitly NO kit action needed yet; what happens at {{release_date}} if silence continues.
   - `release`: what happened (plain JP); `{{share_g}}` on its own line; `{{mail_auth_code}}` with the paper-matching instruction (「印刷されたガイドの『メール確認コード』と同じであることを確かめてください」); 3 numbered `keepsake open` steps; the clarification that the age-only fallback applies to the Recovery-Sheet path; 「このメールの文字列を誰かに送り返す必要は絶対にありません。求められたら詐欺です」; sender-pinning sentence using `{{mail_from}}`.
   - `postrelease_confirm`: short re-notice + same `{{share_g}}` + `{{mail_auth_code}}` (F8 insurance).
   - `released_but_alive`: share_G-is-burned statement + rotation summary + `{{rotation_runbook_url}}`.
   - `config_error_owner`: `{{error_list}}` verbatim (multiline) + fail-closed statement ("no release actions until fixed").
3. Placeholder inventory (added to PLACEHOLDERS.md with sources; only these may appear): `{{owner_name}} {{recipient_name}} {{days_silent}} {{release_date}} {{next_alert_date}} {{next_remind_date}} {{checkin_time}} {{checkin_channel}} {{share_g}} {{mail_auth_code}} {{checkin_days}} {{error_list}} {{mail_from}} {{rotation_runbook_url}}`. Sources normative: `{{share_g}}` = the value the monitor injects at send time; `{{mail_auth_code}}` = `sharetext.MailAuthCode` computed from that same share_G at send time (single source, always consistent); `{{rotation_runbook_url}}` = `https://github.com/Saber5656/keepsake/blob/main/docs/runbooks/rotation.md`.
4. Restriction test: `{{share_g}}` and `{{mail_auth_code}}` appear ONLY in `release` and `postrelease_confirm`.
5. Formats (normative): dates rendered in JST as `2026年8月21日` (source timestamps UTC, converted with fixed `Asia/Tokyo`); day counts plain integers; times as `2026年8月21日 06:23 (JST)`.
6. Drill variants: issue 20's Drill flag adds the `[DRILL]` subject prefix + leading block 「これは訓練です。実際の開封ではありません。」; recipient-bound drill bodies additionally end with the confirm-receipt footer 「このメールが届いたことを <owner名> さんに知らせてください（訓練の確認のためです）」 (issue 26 requirement) — implemented as a registry-level drill footer for recipient audience.
7. BCC/audience behavior is metadata via `Register` (issue 20); templates contain no BCC wording.
8. Wrapping ≤ 78 chars where Japanese text allows; plain text only.

## Acceptance Criteria

- [ ] Golden rendered outputs (normal + drill) for every ID with fixture data, committed and human-reviewed (owner reads all Japanese text — recorded in PR).
- [ ] Placeholder audit test: every used placeholder is in the inventory; restriction test for `{{share_g}}`/`{{mail_auth_code}}`; PLACEHOLDERS.md completeness test extended.
- [ ] Required-content checklist assertions per template (grep-based on goldens: mail-auth instruction, never-send-back sentence, age-fallback clarification, fail-closed sentence, drill footer).
- [ ] Security golden scan: planted share_R text, passphrase, SMTP password never appear in ANY rendered output; share_G only in the two allowed IDs.
- [ ] Missing-placeholder render fails (one negative per template).
- [ ] JST conversion table test incl. year boundary.

## Validation

CI golden tests; human wording review; end-to-end appearance verified during drill (26).

## Dependencies

09 (inventory + tone), 20 (registry interface).

## Non-goals

Delivery mechanics (20), when-to-send logic (19/23), HTML mail, localization beyond ja+EN-footer (v2).

## Design References

DESIGN §12, §9.1, §7 (mail-auth code), §10.5 A7; ADR-006.
