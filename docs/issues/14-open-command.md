# Title

`keepsake open`: recipient decryption wizard with hardened extraction

## Summary

Implement `keepsake open`: the interactive, Japanese-first wizard that locates kit.age, collects both shares with forgiving input handling, combines and fingerprint-checks the KEK, decrypts, and extracts with full path-traversal hardening.

## Context

This command is executed by a grieving non-technical person following a paper guide (DESIGN §2.2). Every error message must say what to do next in plain Japanese. Extraction is a hostile-input boundary (§10.4): the kit is trusted in the normal flow, but `open` must be safe against a swapped/corrupted kit (abuse A9/A10 adjacent).

## Scope

- `internal/vault/open.go`, `extract.go` (+ tests), command wiring.

## Detailed Requirements

1. Invocation modes:
   - Wizard (no flags): search order for kit.age — CWD, executable's directory, executable's parent (USB layouts); if found confirm with user; else prompt for path (drag&drop friendly: strip quotes/escapes/trailing spaces).
   - Non-interactive: `--kit PATH --share-r-file PATH|--share-r TEXT --share-g TEXT --out DIR --yes`.
2. Share collection: auto-detect `share-R.txt` next to kit (confirm before use); share_G prompted as paste/typed; on sharetext errors show the specific problem (E-SHARE-* → bilingual guidance: "4文字の確認コードが合いません。メールの文字列をもう一度..."), max 5 attempts then exit 4 with "call your helper" message.
3. Fingerprint pre-check: if `manifest.json` present, compare combined KEK fingerprint before decryption; mismatch → explain "shares don't match this kit" (likely wrong mail vs old bundle) with next steps.
4. Decrypt streaming to extraction; passphrase derived in-memory only.
5. Extraction hardening (all violations abort with E-OPEN-UNSAFE + path):
   - reject absolute paths, `..` segments, NUL, paths escaping dest after `filepath.Clean` join check
   - reject symlink/hardlink/device/fifo entries
   - per-file and total-size ceilings from payload-manifest (first entry; if missing → conservative defaults 1 GiB/2 GiB), file count cap 100k
   - dest dir `./keepsake-opened-<date>/` (or `--out`), must not pre-exist, created 0700; files 0600
   - post-extract: verify every file's SHA-256 against payload-manifest; report any mismatch prominently (E-OPEN-HASH, list)
6. Success epilogue: point user at `00-README-FIRST.md` (from templates), remind them the folder is now PLAINTEXT (「この中身は暗号化されていません。USB にコピーして持ち歩かないでください」).
7. Wrong-passphrase vs corrupt kit are indistinguishable (05); message covers both with retry guidance.

## Acceptance Criteria

- [ ] Happy-path wizard transcript test (scripted stdin) on a sealed fixture bundle: decrypts, extracts, hashes verified.
- [ ] Malicious-tar suite: traversal (`../x`, absolute, nested `..`), symlink entry, oversize vs manifest, count bomb — each rejected pre-write (no partial files outside dest; assert dest-only writes via tmpdir walk).
- [ ] Fingerprint mismatch fixture → specified message, no decryption attempt.
- [ ] 5-attempt share retry limit; drag&drop path cleanup cases.
- [ ] Non-interactive mode covered; exit codes per DESIGN §8.

## Validation

Unit + adversarial fixture suite (crafted tars encrypted with test KEK); manual walkthrough by a non-engineer following the printed guide is tracked in issue 30's docs QA.

## Dependencies

04, 05, 06, 08, 12.

## Non-goals

Web decryptor (v2), reading Recovery-Sheet path (that flow bypasses keepsake entirely via `age`), share_G QR scanning from mail (text paste only in v1).

## Design References

DESIGN §8 (open row), §10.4 (tar boundary), §5.4, §2.2.
