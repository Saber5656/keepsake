# Title

Japanese content pack: payload templates, bundle README, guide copy, config template

## Summary

Author the non-mail Japanese (plus short English fallback) content as embedded assets: payload starter templates, the plaintext bundle README, the prose blocks for both printable guides, the commented config template used by `init`, and the `PLACEHOLDERS.md` inventory contract. Mail wording is explicitly OUT of scope (issue 21 owns ALL mail content).

## Context

DESIGN §2.2/§2.3: recipients are non-technical Japanese family members; the printed guide and templates ARE the recipient UX. Centralizing wording lets a human review all family-facing text at once; the placeholder inventory is the contract that issues 16/21 render against.

## Scope

- `internal/content/` (embedded via `go:embed`):
  - `payload-templates/00-README-FIRST.md`, `payload-templates/10-accounts.md`, `payload-templates/20-credentials/README.md`, `payload-templates/30-wishes.md`, `payload-templates/40-documents/README.md`
  - `bundle/README-ja.txt`
  - `guide/recipient-guide-copy.md`, `guide/recovery-sheet-copy.md` (aka owner safe sheet — file header states the alias)
  - `config-template.yaml` (commented, `REPLACE_ME` placeholders; consumed by issue 10)
  - `PLACEHOLDERS.md` (inventory: name, meaning, source field — for guide placeholders; issue 21 extends it with mail placeholders)
  - `content.go`: `func FS() fs.FS` returning the embedded tree rooted at the paths above (content-relative, `internal/content/` prefix stripped)
- `internal/content/content_test.go`

## Detailed Requirements

1. All family-facing text: polite Japanese (です・ます), no unexplained jargon; each file ends with a short English summary section for helpers.
2. `recipient-guide-copy.md` MUST cover, in order: これは何？ / いつメールが届く？ / **ニセモノの見分け方** — `{{mail_auth_code}}` の照合手順・差出人固定 (`{{mail_from}}`)・「シェアや暗号文を送り返させる依頼は全て詐欺」 / 開け方 Windows（{{tool_win}} をダブルクリックできない場合の SmartScreen 手順込み）/ 開け方 Mac（Gatekeeper 右クリック→開く手順込み）/ うまくいかない時（合言葉エラー別の対処）/ **メールが来ないままの場合**（金庫の Recovery Sheet・遺言・相続手続きへの導線）/ 困ったら（helper 連絡先欄 `{{helper_note}}`）。age での代替復号は Recovery Sheet の紙にだけ載る旨を明記。
3. `recovery-sheet-copy.md`: 「この一枚だけでキットを開けられます」警告 / USB と同じ場所に保管しない指示 / `age -d kit.age` の 3 手順 / 印刷日・ローテーション日欄 / 鍵指紋・キット指紋の欄。
4. `bundle/README-ja.txt`: これは何か / 勝手に開けない約束（開けられない設計であることも一文で）/ 詳しいことは同梱の HTML ガイドか紙のガイドへ。
5. Payload templates: placeholder rows/examples only; `10-accounts.md` has NO password column (explicit column set: 機関名/種類/口座・契約番号の下 4 桁/連絡先/メモ) plus a pointer row to the password-manager export in `20-credentials/`; every template's header comment states it will be encrypted as-is.
6. `config-template.yaml`: field-by-field JP+EN comments; placeholder values exactly the `REPLACE_ME…` markers from `config.DefaultVaultConfig()` (issue 07); golden-tested equality with the struct defaults.
7. Placeholders used across guide copy — normative set: `{{owner_name}} {{seal_date}} {{guide_version}} {{mail_auth_code}} {{mail_from}} {{helper_note}} {{kek_fingerprint}} {{kit_sha256_short}} {{tool_win}} {{tool_mac}}`. (The share integrity CHECK group is deliberately NOT family-facing; mail authentication uses `{{mail_auth_code}}` only — DESIGN §7.) Every `{{...}}` occurring in any content file MUST have a PLACEHOLDERS.md row (tested).
8. `content_test.go`: embedded set == expected file list; valid UTF-8; non-empty; placeholder inventory completeness; accounts-template password-column negative (grep for パスワード column header).

## Acceptance Criteria

- [ ] All listed files exist with the mandated sections; inventory + password-column + UTF-8 tests green.
- [ ] Native-Japanese review by the owner recorded in the PR.
- [ ] `FS()` accessor test: exact expected file set, content-relative paths.
- [ ] Anti-phishing copy includes all three A7 elements (mail-auth code, fixed sender, never-send-back) — grep-tested.

## Validation

Content review by the owner (human) + automated tests in CI.

## Dependencies

01.

## Non-goals

HTML layout/QR (16), ALL mail wording (21), translations beyond ja/en (v2).

## Design References

DESIGN §2.2, §2.3, §6.3, §7 (mail-auth code), §10.5 A7, §14; ISSUE_PLAN KU-4/KU-5.
