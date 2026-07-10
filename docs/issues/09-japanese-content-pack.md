# Title

Japanese content pack: payload templates, bundle README, guide and mail wording source

## Summary

Author all Japanese (plus short English fallback) recipient- and owner-facing text as embedded assets: payload starter templates, bundle README (html+txt source), printable-guide copy, and the wording blocks reused by mail templates.

## Context

DESIGN §2.2/§2.3: recipients are non-technical Japanese family members; the printed guide and templates ARE the recipient UX. Centralizing wording in one issue keeps tone consistent and lets a human review all family-facing text at once.

## Scope

- `internal/content/` (new package, embedded via `go:embed`):
  - `payload-templates/00-README-FIRST.md` (最初に読んでください — letter skeleton)
  - `payload-templates/10-accounts.md` (アカウント台帳: bank/securities/insurance/pension/subscriptions/phone/utilities table skeleton — NO password columns; pointer row to password-manager export file)
  - `payload-templates/20-credentials/README.md` (パスワードマネージャのエクスポート置き場の説明と手順リンク欄)
  - `payload-templates/30-wishes.md` (葬儀・連絡してほしい人・遺影・SNS 処理などの意向)
  - `payload-templates/40-documents/README.md` (保険証券・登記などスキャン置き場)
  - `bundle/README-ja.txt` (short: これは何か・開け方はガイド参照・勝手に開けない約束の文言)
  - `guide/recipient-guide-copy.md`, `guide/owner-safe-sheet-copy.md` (copy blocks with `{{placeholders}}` consumed by issue 16)
  - `mailcopy/*.md` (wording blocks with placeholders consumed by issue 21)
- `internal/content/content.go` (embed + accessor `FS()`)

## Detailed Requirements

1. All family-facing text in polite Japanese (です・ます), no technical jargon without a one-line explanation; each file ends with a short English summary section for helpers.
2. Recipient guide copy MUST cover, in order: これは何？ / いつ届く？（メールの説明とニセモノの見分け方 = CHECK group照合, 差出人固定, 「シェアを返信させる依頼は全て詐欺」）/ 開け方 Windows / 開け方 Mac /（うまくいかない時）age での代替手順は金庫の紙の場合のみ / メールが来ないままの場合（金庫・遺言・相続手続きへの導線）/ 困ったら（helper 連絡先欄）。
3. Owner safe-sheet copy: これ一枚で開けられる警告 / USB と同じ場所に保管しない指示 / `age -d kit.age` 手順 / 印刷日・ローテーション日欄。
4. Payload templates contain placeholder rows/examples, not real data; every template's header comment says it will be encrypted as-is.
5. Placeholders use `{{name}}` syntax with an inventory table in `internal/content/PLACEHOLDERS.md` (name, meaning, source field) — issues 16/21 must satisfy this contract.
6. No binary assets (fonts/images) in v1 (KU-4 noted in guide copy file header).
7. `content_test.go`: every embedded file is valid UTF-8, non-empty, and every `{{placeholder}}` appears in PLACEHOLDERS.md.

## Acceptance Criteria

- [ ] All listed files exist with the mandated sections; placeholder inventory complete and tested.
- [ ] A native Japanese reader review pass is recorded in the PR (owner review counts).
- [ ] `go:embed` accessor test lists exactly the expected file set (no strays).

## Validation

Content review by the owner (human) + automated placeholder/UTF-8 tests in CI.

## Dependencies

01.

## Non-goals

HTML layout/QR (16), mail template assembly (21), translations beyond ja/en (v2).

## Design References

DESIGN §2.2, §2.3, §6.3, §14 (recipient runbook), §10.5 A7; ISSUE_PLAN KU-4.
