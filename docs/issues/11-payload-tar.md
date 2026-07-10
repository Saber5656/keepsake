# Title

Payload manifest, deterministic tar builder, and hardened extraction primitives

## Summary

Implement `internal/payload`: policy-checked payload scanning, `payload-manifest.json`, a byte-deterministic USTAR writer, and the hardened extraction function later driven by `keepsake open` and `verify --deep`.

## Context

DESIGN §6.2 (determinism) and §10.4 (tar extraction is a hostile-input boundary). Keeping build and extraction in one package gives one owner for the path-grammar and manifest contract.

## Scope

- `internal/payload/scan.go`, `manifest.go`, `tar.go`, `extract.go`, `_test.go`, `testdata/`

## Detailed Requirements

1. Path grammar (shared by scan and extract): relative, `/`-separated, no leading `/`, no `\`, no empty/`.`/`..` segments, no NUL, valid UTF-8 [`E-PAY-PATH` / `E-PAY-UTF8`]; byte-wise lexicographic ordering. (No Unicode normalization in v1: names are used byte-as-is; macOS NFD pitfall documented in the package doc. Sealing is supported on macOS/Linux only — Windows sealing is out of v1 scope, see ISSUE_PLAN §7.)
2. `Scan(dir string, caps Caps) (*ScanReport, error)`:
   ```go
   type Caps struct { SoftTotal, HardTotal, MaxFile int64; MaxCount int } // zero value = defaults 512MiB/2GiB/1GiB/100k
   type Issue struct { Code, Path string; Severity string } // "error" | "warning"
   type ScanReport struct { Files []FileEntry; Issues []Issue; TotalSize int64 }
   type FileEntry struct { Path string; Size int64; SHA256 string } // lowercase hex
   ```
   Collected error codes: `E-PAY-SYMLINK`, `E-PAY-HARDLINK` (Nlink>1 via `syscall.Stat_t`, build-tagged darwin/linux), `E-PAY-SPECIAL`, `E-PAY-FILESIZE`, `E-PAY-TOTAL`, `E-PAY-COUNT`, `E-PAY-EMPTY`, `E-PAY-PATH`, `E-PAY-UTF8`; warning `W-PAY-SIZE` at SoftTotal.
3. `Manifest` JSON: `{schema:1, created_at, files:[{path,size,sha256}], total_size, file_count}` — `created_at` is passed in by the caller (= sealTime; Scan takes no clock, preserving determinism). Canonical encoding everywhere in this package: `json.Encoder`, `SetEscapeHTML(false)`, no indent, struct field order, RFC3339 UTC timestamps, single trailing LF.
4. `PayloadDigest(files []FileEntry) string`: SHA-256 (lowercase hex) over the canonical concatenation `path\x00size\x00sha256\n` of the sorted entries — the drift-detection digest of DESIGN §6.6 (no timestamps).
5. `WriteTar(w io.Writer, dir string, m *Manifest) error`: every header `Format: tar.FormatUSTAR` (names unfit for USTAR → `E-PAY-PATHLEN`); entry order = `payload-manifest.json` first, then the sorted union of directory entries (every ancestor, once) and files, byte-lexicographic; dirs 0755, files 0644, uid=gid=0, empty uname/gname, mtime = `m.created_at` truncated to 00:00:00 UTC. Files re-hashed while streaming; any divergence from the manifest (content, size, disappearance, type change) → `E-PAY-RACE` abort.
6. `Extract(r io.Reader, destParent string, opts ExtractOpts) (*ExtractReport, error)` — the hardened boundary (DESIGN §10.4):
   - First entry MUST be `payload-manifest.json` (parse + schema check) → `E-OPEN-MANIFEST` otherwise; caps derive from it.
   - Reject: absolute paths, `..`, symlink/hardlink/device/fifo entries, duplicate paths [`E-OPEN-DUP`], entries not in the manifest [`E-OPEN-EXTRA`], per-file/total size or count over manifest [`E-OPEN-UNSAFE`]; after EOF, manifest entries never seen → `E-OPEN-MISSING`.
   - Join-and-verify every target stays under dest (`filepath.Clean` containment check).
   - Dest dir created fresh 0700 (pre-existing → error), dirs 0700, files 0600, tar modes ignored.
   - Streaming SHA-256 per file; mismatch vs manifest → `E-OPEN-HASH` (collected; extraction continues so the report is complete).
7. No logging; all errors typed with Code+Path.

## Acceptance Criteria

- [ ] Determinism: two builds of a shuffled-creation-order fixture tree (incl. 日本語 NFC filenames) are byte-identical; a decomposed-NFD-named file is preserved byte-as-is (documented behavior test).
- [ ] USTAR purity: parsing our output asserts every header Format==USTAR; >100-byte-name fixture that USTAR cannot split → `E-PAY-PATHLEN`.
- [ ] Every scan policy code has a fixture; multi-error collection verified.
- [ ] Race matrix: mutate content / change size / delete / replace-with-symlink between Scan and WriteTar → `E-PAY-RACE`.
- [ ] Extraction adversarial suite: traversal (`../x`, absolute), symlink entry, duplicate entry, extra entry, missing entry, oversize file, count bomb, hash mismatch — each rejected with its exact code and zero writes outside dest (tmpdir walk assert).
- [ ] Round-trip: WriteTar → Extract reproduces the tree byte-identically with the specified modes.
- [ ] `PayloadDigest` golden vector committed.

## Validation

CI unit tests; issue 14 consumes `Extract` and issue 28 fuzzes it (`FuzzExtract` target defined here, 10s smoke).

## Dependencies

01.

## Non-goals

Encryption (05/13), wizard UX (14), compression (v2, needs size-oracle analysis).

## Design References

DESIGN §6.2, §6.6 (`payload_digest`), §10.4.
