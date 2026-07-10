# Title

Recipient bundle builder and manifests

## Summary

Implement `internal/bundle`: assemble the USB-ready bundle directory (DESIGN §6.3) from sealed artifacts, write `manifest.json` and `seal-manifest.json` (DESIGN §6.6), and provide the integrity-check core reused by `verify`.

## Context

The bundle is the physical product recipients hold for years. Its layout, manifests, and checksums are load-bearing for `open` (fingerprint pre-check), `verify`, and the guides (CHECK groups).

## Scope

- `internal/bundle/build.go`, `manifest.go`, `check.go`, `_test.go`

## Detailed Requirements

1. `Build(in BuildInput) error` where BuildInput carries: out dir, kit.age path, shareR (keys.Share), shareG CHECK group + KEK fingerprint (strings — shareG raw NEVER enters this package), tools dir (resolved binaries), content FS (09), seal time, config (recipients for README personalization is NOT done — bundle identical for all, ADR-004).
2. Output tree exactly DESIGN §6.3; `share-R.txt` = header comment lines (Japanese: これは鍵の半分です。単体では何も開けません。) + the sharetext line; 0644 in bundle (physical custody is the control).
3. `manifest.json` per §6.6: `{schema:1, kit_format:1, sealed_at, kit_sha256, kit_size, tools:{filename:sha256}, guide_version, share_r_checksum_group, kek_fingerprint}`.
4. `seal-manifest.json` (vault `state/`, 0600) per §6.6 including `share_g_text` and recipient emails; written atomically (tmp+rename); prior file rotated to `seal-manifest.prev.json`.
5. `tools/CHECKSUMS.txt`: `sha256  filename` lines matching manifest.
6. `Check(bundleDir string) (*Report, error)`: recompute all hashes, validate share-R.txt decodes (role R) and its CHECK matches manifest, kit size/hash match; Report lists per-item pass/fail (consumed by 15).
7. Atomic build: assemble in `out/.tmp-<unix>/bundle` then rename to `out/bundle` (replace old only after success; old renamed `.bak-<date>` — one level kept).

## Acceptance Criteria

- [ ] Built tree matches §6.3 byte-for-byte against a golden fixture (except timestamps/hashes which are asserted structurally).
- [ ] `Check` passes on a fresh build; targeted corruptions (kit byte flip, tool swap, share-R edit, manifest edit) each fail with the right item flagged.
- [ ] seal-manifest atomic-rotation behavior verified.
- [ ] No shareG raw/text parameter anywhere in the package API (compile-level assurance reviewed).

## Validation

Unit + golden-tree tests; reviewer confirms §6.6 field parity with DESIGN.

## Dependencies

06, 09, 11.

## Non-goals

Sealing pipeline orchestration (13), guide HTML (16), binary download (13), multi-bundle variants (v2).

## Design References

DESIGN §6.3, §6.6, §10.2; ADR-004.
