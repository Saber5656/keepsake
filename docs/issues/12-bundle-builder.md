# Title

Recipient bundle builder and bundle manifest

## Summary

Implement `internal/bundle`: assemble the USB-ready bundle directory (DESIGN §6.3) from already-sealed artifacts, write the bundle `manifest.json` (DESIGN §6.6), parse/emit `share-R.txt`, and provide the integrity `Check` reused by `verify`. The vault-side `seal-manifest.json` is NOT written here (it belongs to issue 13's orchestration — this package never sees share_G text).

## Context

The bundle is the physical product recipients hold for years. Its layout, manifest, and checksums are load-bearing for `open` (fingerprint pre-check), `verify`, and the guides.

## Scope

- `internal/bundle/build.go`, `manifest.go`, `sharefile.go`, `check.go`, `_test.go`

## Detailed Requirements

1. API (exact; primitives only — no config structs, no share_G anywhere):
   ```go
   type BuildInput struct {
       OutDir        string      // parent of the final "bundle" dir (vault out/)
       KitPath       string      // sealed kit.age (moved/copied in)
       ShareR        keys.Share  // role R
       KEKFingerprint string
       SealedAt      time.Time
       GuideVersion  string
       ToolsDir      string      // validated dir containing exactly the 4 binaries
       Content       fs.FS       // internal/content accessor
   }
   func Build(in BuildInput) (*Manifest, error)
   func Check(bundleDir string) (*Report, error)
   func WriteShareRFile(w io.Writer, shareText string) error
   func ReadShareRFile(r io.Reader) (string, error)   // shared with `open` (issue 14)
   ```
2. Output tree per DESIGN §6.3, EXCEPT `KEEPSAKE-README-ja.html`, which issue 13 copies in after guide generation (Build leaves it absent; Check treats it as required-if-manifest-lists-it). `README-ja.txt` comes from `Content` (issue 09).
3. `share-R.txt` grammar (normative, shared by Write/Read): lines beginning `#` are comments (Japanese warning header from content pack), blank lines ignored, EXACTLY one non-comment non-blank line containing the sharetext; Read returns it or `E-BND-SHAREFILE` (zero or multiple candidate lines). File mode 0644 (physical custody is the control).
4. `manifest.json` (canonical JSON contract identical to issue 11 §3: SetEscapeHTML(false), struct order, RFC3339 UTC, lowercase hex, trailing LF): `{schema:1, kit_format:1, sealed_at, kit_sha256, kit_size, tools:{name:sha256}, guide_version, share_r_checksum_group, kek_fingerprint, readme_html_sha256}` — `name` = bundle-relative basename; `readme_html_sha256` filled by issue 13 after copying the guide (Build writes it empty, 13 rewrites the manifest — sequence documented there).
5. `tools/` handling: source dir must contain exactly `keepsake-darwin-arm64`, `keepsake-darwin-amd64`, `keepsake-windows-amd64.exe`, `keepsake-linux-amd64` as regular files (no symlinks/specials — `E-BND-TOOLS`); copied with 0755 and hashed during copy; `tools/CHECKSUMS.txt` = lines `<sha256 lowercase>  <basename>` sorted by basename, LF endings, no entry for itself.
6. Atomicity: assemble under `OutDir/.tmp-<unixnano>/bundle`; on success remove any `OutDir/bundle.bak`, rename existing `OutDir/bundle` → `bundle.bak`, rename tmp → `bundle`. Stale `.tmp-*` dirs older than 24h are removed at start. Failure leaves the previous bundle untouched.
7. `Check`: recompute kit hash/size, every tool hash, CHECKSUMS.txt consistency, share-R.txt parses (role R) with CHECK matching `share_r_checksum_group`, manifest schema/kit_format, `readme_html_sha256` when non-empty; Report lists per-item pass/fail with codes (consumed by issue 15).
8. Secret/PII discipline: the bundle tree must contain no share_G, no passphrase, no mail-auth code, no emails, no owner PII beyond what the guide content pack deliberately includes (owner display name).

## Acceptance Criteria

- [ ] Built tree structurally matches §6.3 (minus README html) with asserted modes; golden manifest fixture (timestamps injected).
- [ ] `Check` green on fresh build; targeted corruptions (kit byte, tool swap, CHECKSUMS edit, share-R edit, manifest edit) each flagged with the right code.
- [ ] `WriteShareRFile`/`ReadShareRFile` round-trip; zero-line and two-line fixtures → `E-BND-SHAREFILE`.
- [ ] Hostile tools-dir fixtures: symlinked binary, missing binary, extra file, wrong name — `E-BND-TOOLS`.
- [ ] Atomicity: induced failure mid-build leaves prior `bundle/` byte-identical; success rotates `.bak` correctly; stale tmp cleanup verified.
- [ ] PII/secret grep test over the built tree with planted fixture values (share_G text, passphrase, emails) — zero hits.
- [ ] API-level compile check: no share_G parameter exists in this package (reviewer criterion).

## Validation

Unit + golden-tree tests; reviewer confirms §6.6 field parity with DESIGN.

## Dependencies

06, 09, 11.

## Non-goals

Sealing orchestration and seal-manifest (13), guide HTML (16), binary download (13), multi-bundle variants (v2).

## Design References

DESIGN §6.3, §6.6, §10.2; ADR-004.
