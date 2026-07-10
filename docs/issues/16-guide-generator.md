# Title

Printable HTML guide generator with QR codes (`keepsake guide`)

## Summary

Implement `internal/guide` + the `guide` command and complete seal-pipeline step 7 (issue 13's `guideGen` interface): generate the two self-contained printable HTML artifacts — recipient guide (Japanese, share_R QR + mail-auth code) and owner Recovery Sheet (passphrase P + QR) — from config, seal-manifest, the bundle share-R file, and the content pack.

## Context

Paper is the recipient's primary interface (DESIGN §2.2, §14) and the mail-auth code's distribution channel (§7, §10.5 A7). Files must render fully offline and print correctly on A4.

## Scope

- `internal/guide/guide.go`, `qr.go`, `templates/*.gohtml` (layout ONLY — all prose loads from `internal/content/guide/*.md`, issue 09), tests.
- QR encode lib selection (KU-3): `github.com/skip2/go-qrcode` vs `github.com/yeqown/go-qrcode/v2` — evaluate print quality and maintenance; record choice+rationale in the PR. Test-only QR decoder: `github.com/makiuchi-d/gozxing` (allowlisted test-only, DESIGN §10.6).

## Detailed Requirements

1. API (exact; implements issue 13's interface):
   ```go
   type Data struct {
       OwnerName, GuideVersion, KeepsakeVersion string
       SealedAt time.Time
       ShareRText string            // read by caller from bundle via bundle.ReadShareRFile
       ShareGCheckGroup, MailAuthCode, KEKFingerprint, KitSHA256 string
       Passphrase string            // P; ONLY used for the Recovery Sheet; caller recombines via keys.Combine
       MailFrom string              // A7 fixed-sender rendering
       HelperNote string            // optional free text from config
   }
   func Generate(d Data, content fs.FS) (recipientHTML, ownerHTML []byte, err error)
   ```
   The `guide` command builds `Data` from config + seal-manifest + bundle share-R.txt (recombining P from share_R + seal-manifest share_g_text), writes `out/guides/recipient-guide-ja.html` (0644) and `out/guides/owner-safe-sheet-ja.html` (0600) atomically (tmp+rename), and refreshes the bundle copy `KEEPSAKE-README-ja.html` + its manifest hash if `out/bundle/` exists. Writing the owner sheet requires `Confirm` (red-flag notice) unless `--yes`. Exit codes per DESIGN §8.
2. Recipient guide content (prose from 09, placeholders filled): the full section list of issue 09 req 2, PLUS rendered: share_R QR (PNG data-URI, ≥480 px, EC level M, ≥4-module quiet zone, print CSS 45×45 mm) + share_R text in monospace groups; the **mail-auth code** boxed with the matching instruction; `mail.from` as the pinned sender; owner name; seal date; guide_version. It must NOT contain: P, share_G text, share_G CHECK group.
3. Recovery Sheet: P QR + monospace text, `age -d kit.age` 3-step instructions, storage warnings (verbatim from 09), print/rotation date lines, `kek_fingerprint` + first 12 hex of `kit_sha256` footer (labeled 「鍵指紋」/「キット指紋」).
4. HTML constraints: single file, inline CSS only, images as data: URIs only, no scripts/forms; `@page` A4 with page-break rules per major section; JP system-font stack (KU-4 comment). An automated scan asserts NO `http://`/`https://` occurrences in `src|href|url(...)` across both artifacts.
5. Placeholder contract: `GuideData→placeholder` mapping table included in the package doc, validated against `internal/content/PLACEHOLDERS.md` — unknown or unfilled placeholder fails a test (render with `missingkey=error`).
6. Templating: `html/template` only.

## Acceptance Criteria

- [ ] Golden renders for both artifacts with fixture Data (version/date injected).
- [ ] QR round-trip: recipient QR decodes (gozxing) to the exact share_R text; owner QR decodes to exactly P; recipient artifact's decoded QR payloads contain NO P and NO share_G (decode-all-images assertion).
- [ ] Text-level secret scan: recipient HTML contains neither P nor share_G text nor share_G CHECK; owner HTML contains P exactly once in text + once as QR.
- [ ] External-request scan green; placeholder completeness test green.
- [ ] A7 triple assertion in recipient golden: mail-auth code + `mail.from` + 「送り返す必要はありません」sentence all present.
- [ ] File modes and atomic write behavior asserted; owner-sheet confirm flow tested.
- [ ] Manual print QA rubric attached to PR: A4 PDFs from 2 browsers, decoded-QR strings from phone scans of printed paper, browser versions listed (KU-3/KU-4 gate).

## Validation

CI golden + QR round-trip tests; human print/scan QA in the PR.

## Dependencies

09, 13.

## Non-goals

PDF generation (browser print), mail bodies (21), multi-language guides (v2), photo/screenshot illustrations (v2).

## Design References

DESIGN §7 (QR, mail-auth code), §8 (guide row), §10.5 A7, §10.6 (QR libs), §14; ISSUE_PLAN KU-3/KU-4.
