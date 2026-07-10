# keepsake — Design Document (v1)

Document status: **DRAFT for review** · 2026-07-10 · Language: English (recipient-facing artifacts are Japanese; see §2.3)
Canonical source of truth: this repository's `docs/` directory. GitHub Issues are derived from `docs/ISSUE_PLAN.md` and `docs/issues/*.md`.

---

## 1. エグゼクティブサマリ（日本語）

**keepsake は「デジタル終活キット」**。本人（技術者）が家族のために暗号化された緊急キット（アカウント一覧・認証情報・手紙・手続きの指示）を作成し、**生前に物理媒体（USB + 紙のガイド）で家族へ渡しておく**。キットは本人が生きている間は誰にも開けない。

開封の鍵は 2 つに分割される（Shamir 2-of-2）：

- **share_R**: 家族が持つ USB / 紙に印刷（単独では無力）
- **share_G**: 本人の private GitHub リポジトリの Actions secret（単独では無力）

**デッドマンスイッチ**は GitHub Actions（無料枠・サーバ不要・本人の死後も動き続ける）上で毎日動く。本人は定期的に check-in（CLI または スマホの GitHub ボタン）する。check-in が途絶えると 段階的に：本人へ督促 → 家族へ「本人の様子を見て」安全確認メール → 猶予期間満了で **share_G を家族へメール送付**。家族は「USB の share_R + メールの share_G」を合わせて初めて復号できる。

- GitHub にも、メール事業者にも、泥棒にも、**単独ではいっさい中身を読めない**
- 暗号化は標準 **age フォーマット**。keepsake が消滅しても家族は標準ツールで復号可能（可用性）
- 本人の金庫に**復元用パスフレーズの紙**（最後の砦。USB と同じ場所に保管しないこと）
- 年 1 回の**避難訓練（drill）**で、配送経路が生きていることを本人が生前に検証する

v1 は CLI + GitHub Actions で完結。Web 復号 UI・k-of-n 人間トラスティ・受領者ごとの個別ペイロードは v2。

---

## 2. Product Concept and Personas

### 2.1 One-line definition

A digital end-of-life kit: an owner-operated, serverless dead man's switch (GitHub Actions) that releases one half of a split key to family members who already physically hold the encrypted kit and the other half.

### 2.2 Personas

| Persona | Description | Capabilities assumed |
|---|---|---|
| **Owner** | Software engineer preparing 終活. Runs macOS/Linux, has GitHub account, SSH keys, can operate CLI and GitHub Actions secrets. | Full technical capability. |
| **Recipient** | Owner's family member (e.g. spouse). Non-technical. Can use a smartphone camera (QR), a web browser, and follow a printed Japanese step-by-step guide. May enlist a "helper". | No CLI knowledge. Japanese reading. |
| **Helper** | Optional IT-literate acquaintance or professional assisting a Recipient at recovery time. | Can run a downloaded binary or `age` following instructions. |

### 2.3 Language policy

| Artifact | Language |
|---|---|
| Code, comments, CLI output (default), engineering docs, issues | English |
| Recipient-facing artifacts: printed guides, bundle README, alert/release mails, payload templates | Japanese (primary), with short English fallback sections for helpers |
| Owner-facing mails (reminders) | Japanese |

### 2.4 Product principles

1. **The trigger must outlive the owner.** No infrastructure whose continued operation depends on the owner's payments or hardware (research 01: self-hosting paradox).
2. **No single party can decrypt alone.** Not GitHub, not the mail provider, not a recipient before the trigger, not a thief of any one artifact (2-of-2 split, §5).
3. **The kit must outlive the project.** Ciphertext is standard age v1; paper artifacts are self-describing; recovery must be possible with third-party tools (§5.4).
4. **False-fire is embarrassing; missed-fire is catastrophic; both are design failures.** Escalation ladder with a human safety-check phase before release (§9).
5. **Non-technical recipient UX is a v1 requirement, not a nicety.** Printed Japanese guide + QR codes + interactive wizard (§8 `open`, §14).
6. **Everything fails; every failure has a documented fallback.** Ultimate backstop is paper in the owner's safe, reachable via ordinary probate (§13).

---

## 3. Scope

### 3.1 v1 scope (completion definition)

v1 is complete when the Owner can, using only released keepsake artifacts and documentation:

1. Create a local vault, fill payload from Japanese templates, and `seal` it into recipient bundles (USB-ready directory + printable Japanese guides).
2. Scaffold a private trigger repository on GitHub with `trigger init`, set the documented secrets manually, and have the monitor run daily.
3. Check in via CLI (`keepsake checkin`) and via phone (GitHub workflow_dispatch button).
4. Stop checking in and observe, on a compressed test timeline: owner reminders → recipient safety-check alert → automatic release mail containing share_G → successful decryption by following the printed guide with `keepsake open`. (The standard-`age`-CLI fallback applies only to the Recovery-Sheet path, which carries the passphrase P directly; the share_R+share_G path requires a keepsake binary to combine shares — see §5.4.)
5. Run an end-to-end drill that exercises schedule, binary integrity, SMTP delivery to every recipient, and template rendering, without exposing real key material.
6. Recover from the documented failure scenarios per runbooks (§14).

### 3.2 v1 non-goals

| Non-goal | Rationale / future |
|---|---|
| Web-based decryption UI | v2. Second crypto implementation doubles audit surface. v1 mitigates with wizard CLI + `age` interop. |
| k-of-n human trustee shares (lawyer, siblings) | v2. Share format (Shamir) already supports n≤255; only UX/runbooks missing. |
| Per-recipient distinct payloads | v2. v1: all recipients receive identical bundles. |
| Hosted/SaaS offering | Out of scope permanently for this repo. |
| Password manager features (item CRUD, autofill) | Never. Payload carries password-manager *exports*. |
| macOS notarization / Windows code signing | v2. Guide documents Gatekeeper/SmartScreen bypass steps. |
| PDF generation | v1 generates printable self-contained HTML; user prints via browser. |
| Non-GitHub trigger substrates (GitLab CI, etc.) | v2. State machine is substrate-agnostic by design. |
| Automatic cloud replication of bundles | v2, needs its own threat analysis. |

### 3.3 Deferred (v2 candidate) items

Tracked in ISSUE_PLAN.md "Deferred v2". Highlights: static web decryptor, k-of-n trustees, per-recipient payloads, secondary trigger substrate, cosign/SLSA release provenance, macOS notarization, Windows signing, localization framework, encrypted cloud sync of vault.

---

## 4. System Architecture

### 4.1 Components

```
┌────────────────────── Owner's machine (trusted) ──────────────────────┐
│  keepsake CLI                                                         │
│  Local Vault  ~/KeepsakeVault/                                        │
│    config.yaml   payload/ (PLAINTEXT)   out/ (bundles, guides)        │
│    state/ (seal manifest, share_G copy, checkin counter)              │
└──────────────┬─────────────────────────────┬──────────────────────────┘
               │ seal (offline)              │ checkin (git push, signed)
               ▼                             ▼
┌─────────────────────────┐   ┌──────────────────────────────────────────┐
│ Recipient Bundle (USB×n)│   │ Trigger Repo (private, GitHub)           │
│  kit.age  (ciphertext)  │   │  keepsake.yaml  (config, no secrets)     │
│  share-R.txt + paper QR │   │  state/ checkin.json .sig  state.json    │
│  README-ja.html  tools/ │   │  bin/keepsake-linux-amd64 (+.sha256)     │
└─────────────────────────┘   │  .github/workflows/ monitor.yml          │
                              │                     checkin-button.yml   │
   Owner's safe (paper):      │  Actions secrets (set manually by owner):│
   Recovery Sheet =           │   KEEPSAKE_SHARE_G, KEEPSAKE_SMTP_URL,   │
   KEK passphrase + QR        │   KEEPSAKE_SMTP_URL_SECONDARY (optional) │
                              └───────────────┬──────────────────────────┘
                                              │ daily cron: monitor
                                              ▼
                              reminders → owner; alert / release → recipients
                                       (SMTP via owner-chosen provider)
```

### 4.2 Repositories

| Repo | Visibility | Contents | Created by |
|---|---|---|---|
| `keepsake` (this repo) | public (OSS) | CLI source, workflows templates, docs | — |
| trigger repo (e.g. `keepsake-switch`) | **private**, owner's account | config, state, vendored binary, workflows. **Never** ciphertext, never payload, never share_R, never KEK. | `keepsake trigger init` scaffolds a local directory; owner creates the GitHub repo and pushes (manual, documented). |

### 4.3 Go module layout (normative)

Module path: `github.com/Saber5656/keepsake`. Binary: `keepsake`. CLI framework: **standard library `flag`** with a small subcommand router (no cobra; dependency policy §10.6).

```
cmd/keepsake/main.go            # dispatch only
internal/cliutil/               # router, printer (ja/en), exit codes, prompts
internal/version/               # version, embedded at build (ldflags)
internal/config/                # schemas, strict YAML load, validation
internal/payload/               # payload walk, manifest, deterministic tar
internal/crypto/agefile/        # age seal/open wrappers
internal/crypto/keys/           # KEK generation, split/combine orchestration
internal/shamir/                # VENDORED from vault v1.14.8 (MPL-2.0)
internal/encoding/sharetext/    # §7 codec (encode/decode/checksum)
internal/bundle/                # recipient bundle builder + verification
internal/guide/                 # printable HTML guides, QR embedding
internal/checkin/               # statement format, SSH sign/verify
internal/statemachine/          # §9 pure evaluation engine
internal/mail/                  # composer, templates, SMTP send (go-mail)
internal/gh/                    # minimal GitHub REST client (dispatch-run verification)
internal/trigger/               # trigger repo scaffold/update, monitor orchestration
```

Dependency allowlist: §10.6.

---

## 5. Cryptographic Design

### 5.1 Key hierarchy

```
KEK (32 bytes, crypto/rand)
 ├─ encoded as passphrase string P = sharetext-encode(role=K, KEK)      → printed Recovery Sheet (owner's safe)
 │   P is the age scrypt passphrase for kit.age
 └─ Shamir Split(KEK, n=2, threshold=2)  [GF(2^8), 33-byte shares]
     ├─ share_R → sharetext-encode(role=R) → in every recipient bundle (file + paper QR)
     └─ share_G → sharetext-encode(role=G) → GitHub Actions secret KEEPSAKE_SHARE_G
                                              (+ a copy in local vault state/ for verify/rotation)
```

- Combine(share_R, share_G) = KEK → re-encode as P → decrypt kit.age. `keepsake open` does this internally; the manual fallback path documents it for `age` users.
- All recipients hold the **same** share_R (v1). Rationale: distinct shares with threshold 2 would let two recipients collude to decrypt pre-trigger; identical shares make recipient collusion useless by construction (§10.5 A3). Revocation of a recipient therefore requires rotation (§14).
- The Recovery Sheet holds P (equivalently the KEK) — it alone decrypts a stolen kit.age. Its storage rule (owner's safe, physically separate from any bundle USB) is a **normative instruction** printed on the sheet itself and in the owner runbook.

### 5.2 Encryption format

- `kit.age`: standard age v1 file, single ScryptRecipient with passphrase P, `SetWorkFactor(18)` pinned.
- Note: P has full 256-bit entropy, so scrypt cost is not security-load-bearing (brute force is infeasible regardless); the pinned factor is for UX predictability on old family hardware.
- Payload container inside: deterministic USTAR tar (§6.2) of the vault `payload/` directory plus `payload-manifest.json`.
- AEAD (ChaCha20-Poly1305) + header MAC come from age; any ciphertext tampering fails decryption loudly.

### 5.3 Randomness and zeroization

- All key material from `crypto/rand` only.
- KEK/shares/P live in `[]byte`/`string` briefly; best-effort zeroization of `[]byte` buffers after use (`for i := range b { b[i] = 0 }`); no secret ever written to disk unencrypted except: share_G copy + seal manifest in local vault `state/` (0600, vault is trusted-machine scope), and printed artifacts the owner explicitly generates.
- No secret material in CLI output except via explicit `--print-secrets`-style commands used during setup (`seal` prints share_G once for manual GitHub secret entry, with a "clear terminal" reminder).

### 5.4 Long-term decryptability (availability)

| Layer | Guarantee |
|---|---|
| Format | age v1 is a published spec with ≥3 independent implementations. CI proves `age` CLI interop every build. |
| Tools | Bundle `tools/` carries static binaries for darwin-arm64/darwin-amd64/windows-amd64/linux-amd64. |
| Paper | Guide documents the manual path: combine shares with any Shamir tool is NOT required — recipients never combine manually; `keepsake open` or (fallback) printed instructions: install age, run `age -d kit.age`, enter P. But P = combine(share_R, share_G) requires keepsake... **therefore the release mail includes P-reconstruction NOT as a manual step but the guide directs the recipient to `keepsake open`; the pure-age fallback applies to the Recovery Sheet path (safe → P directly).** |
| Versioning | `kit-format: 1` recorded in bundle manifest; CLI refuses to open newer major formats with a clear upgrade message. |

Note: the emailed share_G plus USB share_R without ANY keepsake binary cannot be combined by hand practically; this is accepted because (a) bundle carries binaries for 4 platforms, (b) the Recovery Sheet (safe) provides a keepsake-free path via plain `age`, (c) any future age-ecosystem tool can decrypt given P. Explicitly documented in the recipient guide.

---

## 6. Data Formats and Storage Layout

### 6.1 Local vault (`~/KeepsakeVault/` default, overridable via `--vault` / `KEEPSAKE_VAULT`)

```
config.yaml                    # §6.4
payload/                       # owner-authored plaintext content (templates: issue 15)
out/                           # regenerated by seal/guide; safe to delete
  bundle/                      # recipient bundle (single, identical for all recipients)
  guides/recipient-guide-ja.html
  guides/owner-safe-sheet-ja.html
state/                         # 0700 dir, 0600 files
  seal-manifest.json           # §6.6 (includes share_G copy, kit checksum, timestamps)
  checkin-counter              # integer, last used counter
```

Rules: vault MUST NOT be inside a cloud-synced directory (`init` warns on known paths: Dropbox, iCloud `Mobile Documents`, Google Drive, OneDrive); vault is not a git repo; `payload/` max default 512 MiB warn / 2 GiB hard error (config-overridable).

### 6.2 Deterministic payload tar

USTAR format via `archive/tar`; entries sorted lexicographically by path; uid/gid=0, uname/gname empty, mtime = seal timestamp truncated to UTC midnight, mode 0644 files / 0755 dirs; symlinks REJECTED at walk time (error names the offending path); hardlinks rejected; path separator `/`; max single file 1 GiB. `payload-manifest.json` (first tar entry) lists every file with size + SHA-256.

### 6.3 Recipient bundle layout (`out/bundle/`)

```
KEEPSAKE-README-ja.html        # printed guide's digital twin; self-contained HTML
KEEPSAKE-README-ja.txt         # plaintext short version
kit.age                        # ciphertext
share-R.txt                    # §7 text, one line + human note header
manifest.json                  # §6.6 bundle manifest (no secrets beyond share-R presence)
tools/
  keepsake-darwin-arm64
  keepsake-darwin-amd64
  keepsake-windows-amd64.exe
  keepsake-linux-amd64
  CHECKSUMS.txt                # sha256 of the four binaries
```

### 6.4 `config.yaml` (local vault) — schema v1

```yaml
version: 1
owner:
  name: "山田太郎"                     # required, 1..100 chars
  email: "taro@example.com"           # required, RFC 5322 addr-spec, no CR/LF
  github_login: "Saber5656"           # required for dispatch check-in verification
  ssh_signing_key: "~/.ssh/id_ed25519"  # private key file used by `checkin`
  allowed_signers:                    # ≥1 entries, OpenSSH allowed_signers line format
    - "taro@example.com ssh-ed25519 AAAA..."
recipients:                           # 1..10 entries
  - name: "山田花子"
    relation: "妻"                    # free text, shown in mails/guides
    email: "hanako@example.com"       # carrier domains (docomo.ne.jp etc.) → lint warning W-MAIL-1
trigger:
  repo: "Saber5656/keepsake-switch"   # owner/name
  local_path: "~/dev/keepsake-switch"
  binary_version: "v0.1.0"            # release tag to vendor
timing:                               # all integers, days
  checkin_interval_days: 14           # advisory (shown in reminders)
  remind_after_days: 14
  remind_every_days: 3
  alert_recipients_after_days: 28
  alert_every_days: 7
  release_after_days: 42
  postrelease_confirm_every_days: 7
  postrelease_confirm_count: 4
  pause_until: null                   # RFC 3339 date or null; max 90 days ahead at validation time
mail:
  from: "keepsake <taro+keepsake@example.com>"   # display-name + addr-spec
  subject_prefix: "[keepsake]"
  reply_to: "taro@example.com"        # optional
```

Validation (single pass, all errors reported together): strict fields (unknown key = error), `7 ≤ remind_after_days`, `remind_after_days < alert_recipients_after_days < release_after_days`, `release_after_days − alert_recipients_after_days ≥ 7`, `release_after_days ≥ 21`, `remind_every_days ≥ 1`, `alert_every_days ≥ 1`, email syntax + CR/LF rejection everywhere, ≤10 recipients, allowed_signers parse.

### 6.5 `keepsake.yaml` (trigger repo) — generated, schema v1

Subset of §6.4: `version, owner{name,email,github_login}, allowed_signers, recipients[{name,email}], timing, mail, binary{path,sha256,version}`. No paths from the owner's machine, no ssh private key reference, no secrets. Regenerated by `trigger init` / `trigger update`; hand-edits allowed but must re-validate (monitor validates on every run; invalid config → owner error mail + job failure, never silent).

### 6.6 Manifests

`seal-manifest.json` (local, 0600): `{schema:1, sealed_at, kit_sha256, kit_size, payload_file_count, payload_total_size, kek_fingerprint (SHA-256 of KEK, first 8 bytes hex), share_g_text, share_r_checksum_group, share_g_checksum_group, binary_version, recipients:[emails]}`.
`manifest.json` (bundle): `{schema:1, kit_format:1, sealed_at, kit_sha256, kit_size, tools:{name:sha256}, guide_version, share_r_checksum_group, kek_fingerprint}`. No secret material; `share_r_checksum_group` is the 4-char checksum of share_R (already on the same medium) used by `verify`; `kek_fingerprint` lets `verify`/`open` confirm combined KEK correctness before attempting decryption.

### 6.7 Check-in statement (`state/checkin.json` + `state/checkin.sig` in trigger repo)

```json
{"version":1,"counter":42,"timestamp":"2026-07-10T09:00:00Z","note":""}
```
Canonical bytes = the exact file bytes (no re-serialization). Signature: SSHSIG armored, namespace `keepsake-checkin`, produced with owner key; verified against `allowed_signers`. Monitor acceptance: signature valid AND `counter > state.accepted_counter` AND `timestamp ≤ now+15min`. Counter recovery on a new machine: CLI reads `state.json.accepted_counter` from the trigger repo and continues from max+1.

### 6.8 Monitor state (`state/state.json` in trigger repo, committed by monitor)

```json
{
  "schema": 1,
  "phase": "ACTIVE|REMINDING|ALERTING|RELEASED",
  "accepted_counter": 42,
  "last_valid_checkin": "2026-07-10T09:00:00Z",
  "last_checkin_channel": "cli|dispatch",
  "last_run_at": "2026-07-11T21:23:41Z",
  "last_remind_sent_at": null,
  "last_alert_sent_at": null,
  "release": {"released_at": null, "sent": {"hanako@example.com": "2026-08-21T21:24:00Z"}, "confirm_sends": 0},
  "history": [{"at":"...","event":"PHASE_CHANGE","from":"ACTIVE","to":"REMINDING","detail":""}]
}
```
`history` capped at 500 entries (oldest dropped). state.json is informational + idempotency record; authenticity of check-ins never derives from it alone (§6.7, §11.3).

---

## 7. Key Material Text Encoding (`sharetext`)

Normative format for share_R, share_G, and the recovery passphrase P:

```
KEEPSAKE-<ROLE>1-<GROUPS>-<CHECK>
ROLE   ∈ {R, G, K}            # R/G = Shamir shares, K = KEK (recovery passphrase P); "1" = format version
GROUPS = Crockford Base32 (uppercase, alphabet 0123456789ABCDEFGHJKMNPQRSTVWXYZ)
         of the raw bytes (33 bytes for shares, 32 for KEK), in dash-separated groups of 4
CHECK  = first 4 Crockford chars of SHA-256(role-byte || raw-bytes)
```

Example (share): `KEEPSAKE-R1-04X2-...-9TQD-K7M2`

- Decoder: case-insensitive; maps `I→1, L→1, O→0, U→V`(reject U? Crockford excludes U — treat as error); ignores spaces and dashes anywhere; validates length for role, then checksum. Errors are specific: `E-SHARE-LEN`, `E-SHARE-CHK`, `E-SHARE-ROLE`, `E-SHARE-CHAR` (each with a Japanese + English message in the wizard).
- The final CHECK group (4 chars) doubles as the **anti-phishing token**: the printed recipient guide shows the expected CHECK group of share_G, so a recipient can validate that a "release mail" is genuine before trusting it (§10.5 A7).
- QR payload = the exact same string. QR error correction level M.
- Fuzz + property tests required (round-trip, mutation detection ≥ any single-char error).

---

## 8. CLI Command Surface

Global flags: `--vault PATH`, `--lang ja|en` (wizard/messages; default: `ja` if `LANG` contains `ja`, else `en`), `--json` (machine-readable output where noted), `--yes` (suppress confirmations). Exit codes: 0 ok; 1 generic error; 2 usage; 3 validation failed; 4 integrity/crypto failure; 5 network/remote failure.

| Command | Purpose | Key behaviors (normative) |
|---|---|---|
| `keepsake init` | Create vault skeleton | mkdir vault + payload templates (ja) + example config; cloud-sync path warning; refuses non-empty target unless `--force`. Never overwrites payload. |
| `keepsake status` | Show vault + switch state | payload inventory (count/size/mtime), last seal vs payload drift (hash compare → "re-seal needed"), trigger repo state.json summary if `trigger.local_path` exists (pull optional `--fetch`). `--json` supported. |
| `keepsake seal` | Build kit + bundles + shares | pipeline: validate config → build tar+manifest → gen KEK → encrypt kit.age → split shares → write bundle (§6.3) → write seal-manifest → print share_G (once) + GitHub secret instructions + checklists. Atomic: builds into `out/.tmp-<ts>/` then rename. `--tools-dir` for offline binary source, else verified download (§8.1). Requires `--yes` or interactive confirm if a previous seal exists (rotation reminder shown). |
| `keepsake open` | Recipient decryption wizard | interactive (ja default): locate `kit.age` (arg or same-dir autodetect), prompt share_R (auto-read `share-R.txt` if beside kit, confirm), prompt share_G (typed/pasted from mail), validate both (§7 errors in plain language), combine, check `kek_fingerprint` if manifest present, decrypt to `./keepsake-opened/` (0700), print next-step message pointing at `00-README-FIRST.md`. Non-interactive mode: `--kit --share-r-file --share-g "..." --out`. |
| `keepsake verify` | Owner self-test | config validation; bundle integrity (manifests, checksums, tools checksums); share round-trip: reads share_G from seal-manifest + share_R from bundle, combines, compares kek_fingerprint, test-decrypts kit.age header (or full with `--deep`); prints report; exit 3/4 on failure. |
| `keepsake guide` | (Re)generate printable HTML guides | writes both guides from current config + seal-manifest (needs a prior seal for QR/checksum data). |
| `keepsake checkin` | Liveness signal via git | bump counter, write+sign statement (§6.7), commit + push to trigger repo (`trigger.local_path`; clones/pulls first; conflict → pull --rebase once, retry once). Prints current phase from last known state.json. `--note`. |
| `keepsake monitor` | Trigger-side evaluation (runs in Actions; also locally testable) | see §11.3. Flags: `--repo-dir`, `--now` (test), `--drill`, `--dry-run` (no send/commit, print action plan). |
| `keepsake trigger init` | Scaffold trigger repo directory | writes keepsake.yaml (from vault config), workflows, vendored binary (§8.1) + `.sha256`, README-switch.md incl. the **manual secrets checklist** (KEEPSAKE_SHARE_G, KEEPSAKE_SMTP_URL[, _SECONDARY]) — owner sets secrets by hand; keepsake never touches GitHub secret APIs. |
| `keepsake trigger update` | Refresh binary/workflows after upgrade | re-vendor binary at `trigger.binary_version`, regenerate workflows + keepsake.yaml preserving hand-edited timing (three-way: regenerate, but refuse if local uncommitted changes). |
| `keepsake drill` | Run a full test-fire | convenience wrapper: triggers monitor workflow via `gh workflow run monitor.yml -f drill=true` if `gh` present, else prints exact manual steps. Verifies afterwards via Actions API that the run succeeded. |
| `keepsake version` | Version info | semver + commit, `--json`. |

### 8.1 Binary provisioning for `seal` (tools/) and `trigger init` (vendored binary)

Download `keepsake_<ver>_<os>_<arch>` + `checksums.txt` from this repo's GitHub Releases for `trigger.binary_version`; verify sha256 BEFORE any use/copy; cache in `~/.cache/keepsake/dist/<ver>/`; `--tools-dir` bypasses download for air-gapped sealing. Never auto-select "latest": version is pinned in config.

---

## 9. Dead Man's Switch State Machine

Pure function (no I/O): `Evaluate(cfg Timing, st State, checkins CheckinFacts, now time.Time) → (State', []Action)`.

`elapsed = floor(now − last_valid_checkin, 24h)` in whole days. All comparisons day-granular; DST/timezones irrelevant (UTC everywhere).

### 9.1 Phases and transitions

| # | Condition (evaluated in order) | Phase | Actions |
|---|---|---|---|
| T0 | new valid check-in with phase ∈ {ACTIVE, REMINDING, ALERTING} | → ACTIVE | record channel+timestamp; if previous phase ≠ ACTIVE, send owner "check-in received, switch reset" mail |
| T1 | `pause_until` set AND now < pause_until (and phase ≠ RELEASED) | ACTIVE (reason=paused) | none (pause validated ≤90d at config load) |
| T2 | elapsed < remind_after | ACTIVE | none |
| T3 | remind_after ≤ elapsed < alert_after | REMINDING | owner reminder mail if `last_remind_sent_at` is null or ≥ remind_every_days ago |
| T4 | alert_after ≤ elapsed < release_after | ALERTING | owner reminder (same cadence); recipient safety-check mail ("please contact the owner"; NO key material) if `last_alert_sent_at` null or ≥ alert_every_days ago |
| T5 | elapsed ≥ release_after | RELEASED (sticky) | release mail (share_G + instructions + anti-phishing CHECK group context) to each recipient not yet in `release.sent`; then confirmation re-sends every postrelease_confirm_every_days up to postrelease_confirm_count |
| T6 | valid check-in while RELEASED | RELEASED (stays) | owner mail "switch already fired — share_G must be considered burned; rotate now" (weekly max) |

Rules: transitions are computed fresh each run from durable facts (idempotent, missed-cron tolerant); send-then-persist ordering (duplicate mail is tolerated; silently-missed mail is not); RELEASED is never auto-exited (rotation runbook only); drill mode (§11.4) leaves state.json untouched except appending a `DRILL_RUN` history entry and never uses real share_G.

### 9.2 Default timeline (defaults from §6.4)

```
day 0        last check-in
day 14..27   REMINDING: owner mail on days 14,17,20,23,26
day 28..41   ALERTING: owner mail cadence continues; recipients mailed days 28,35
day 42       RELEASED: share_G mailed to all recipients; confirms days 49,56,63,70
```

### 9.3 Validation-enforced sanity

Config validation (§6.4) guarantees ≥7 days between first recipient alert and release (family intervention window) and ≥21 days total before release. The state machine additionally clamps: if config was hand-edited into an invalid state, monitor refuses to run release actions and mails the owner an error (fail-closed toward "no release", because a wrongful release is the irreversible direction).

---

## 10. Security Model

### 10.1 Assets

| ID | Asset | Compromise impact |
|---|---|---|
| A-1 | Payload plaintext | Total loss of confidentiality |
| A-2 | KEK / recovery passphrase P | Decrypts any copy of kit.age |
| A-3 | share_R (bundle) | Half of key; useless alone |
| A-4 | share_G (GitHub secret) | Half of key; useless alone |
| A-5 | kit.age ciphertext | Useless alone; combined with A-2 (or A-3+A-4) = A-1 |
| A-6 | Liveness signal integrity | Forged liveness = switch never fires (missed-fire); forged death = early fire (release of A-4 only) |
| A-7 | Recipient PII (names, emails) | Privacy leak (private repo scope) |
| A-8 | Owner GitHub account | Controls A-4 + A-6 + trigger config |

### 10.2 Trust boundaries and expectations

| Boundary | Trusted with | Explicitly NOT trusted with | Enforcement |
|---|---|---|---|
| Owner's machine + vault | Everything (plaintext, KEK at seal time) | — (documented: FileVault/LUKS recommended; vault outside cloud sync; state/ 0600) | docs + init warnings + file modes |
| Recipient (pre-trigger) | ciphertext + share_R | KEK, share_G, plaintext | 2-of-2 split |
| GitHub | share_G, config, PII (A-7), liveness state | ciphertext, share_R, KEK, plaintext | nothing else is ever pushed to the trigger repo; `trigger init` asserts kit.age & share-R patterns in a repo-local ignore + monitor refuses to run if `kit.age`/`share-R*` files are detected in the repo tree (belt & suspenders) |
| Mail provider | mails incl. share_G at release time | ciphertext (never attached), share_R, KEK | design: release mail carries share_G only |
| `age`/vendored crypto | correctness of primitives | — | pinned versions, interop tests, vendored MPL files with provenance |
| GitHub Actions runner | executing pinned binary with secrets in env | long-term storage | binary sha256 verified in-workflow before execution; workflows pinned by action SHA |

### 10.3 Secret-handling rules (normative for every issue)

1. No secret (KEK, P, shares, SMTP creds) is ever logged, echoed, or included in errors/panics. A test greps a full-flow debug-log run for known test-secret substrings (CI gate).
2. Actions log masking is NOT relied upon; rule 1 is enforced in the binary itself.
3. Secrets reach the monitor only via environment variables (`KEEPSAKE_SHARE_G`, `KEEPSAKE_SMTP_URL*`, `GITHUB_TOKEN`); they are read once into locals and never re-exported, never written to the repo checkout.
4. On-disk secrets on owner machine limited to vault `state/` (0600, 0700 dir).
5. `seal` output of share_G is explicit, single-shot, and followed by a "clear your terminal / do not screenshot" notice.
6. Owner sets GitHub secrets manually (agent/tooling never does; aligns with operator policy).

### 10.4 Input validation boundaries

| Boundary | Validation (normative) |
|---|---|
| YAML configs | strict decode (unknown fields error), schema versions checked, §6.4 rules, size cap 1 MiB |
| sharetext input (wizard/mail paste) | §7: charset/length/role/checksum before any crypto; error codes E-SHARE-* |
| kit.age | age header parse errors surfaced as "wrong shares or corrupted kit" guidance; size cap = manifest size + slack |
| tar extraction (`open`) | reject absolute paths, `..`, symlinks/hardlinks/devices; per-file and total size caps from manifest; extraction into freshly-created 0700 dir; file count cap 100k |
| checkin statement | exact-bytes signature verify; counter monotonicity; timestamp ≤ now+15min; size cap 4 KiB |
| Actions API responses | JSON schema-checked; actor login exact match (case-insensitive per GitHub semantics); pagination bounded |
| Mail addresses | RFC 5322 addr-spec subset; CR/LF injection rejected at config load AND at send |
| CLI file args | no shell interpolation (Go exec with arg vectors only); paths cleaned |

### 10.5 Abuse cases and responses

| ID | Abuse case | Outcome / mitigation |
|---|---|---|
| A1 | Forge check-in with stolen push credentials (no SSH signing key) | Monitor rejects unsigned/invalid statements → switch keeps counting down; owner reminded (mails reference last VALID check-in) |
| A2 | Replay an old signed check-in | counter monotonicity + timestamp window |
| A3 | Two recipients collude pre-trigger | identical share_R ⇒ no new information; release requires share_G |
| A4 | Thief steals bundle USB (or backup copy) | has A-3+A-5; cannot decrypt; owner rotates on known theft (runbook) |
| A5 | Owner GitHub account compromised (read) | attacker gets share_G (A-4) only: useless without physical bundle. Rotate per runbook. |
| A6 | Owner GitHub account compromised (write) — early-fire, config tamper, or switch deletion | Early-fire: recipients get share_G early; only recipients (who hold share_R) gain anything, and owner is notified (T6 mail + release mails are BCC'd to owner) → rotate. DoS/deletion: switch silently dead → mitigations: owner notices missing reminder cadence (documented expectation), annual drill, GitHub 2FA hardware-key recommendation; ultimate backstop = safe Recovery Sheet via probate. |
| A7 | Phishing mail pretending to be the release mail | Printed guide carries share_G's expected 4-char CHECK group + fixed sender address + instruction "no one will ever ask you to send shares back" |
| A8 | Malicious PR / dependency compromise in OSS repo | branch protection, review requirement, pinned deps + govulncheck + dependabot, actions pinned by SHA, vendored crypto with provenance; trigger repos pin binary by sha256 (upgrade is a deliberate owner action) |
| A9 | Tampered binary or workflow inside trigger repo (post-compromise) | equivalent to A6-write; sha256 self-check catches accidental corruption, not malicious rewrite of both binary and checksum — documented residual risk of A-8 boundary |
| A10 | Burglar opens owner's safe (Recovery Sheet) | P alone decrypts only if they ALSO obtain kit.age (bundle/USB). Sheet instruction: never store a bundle in the same safe. Rotate on known safe breach. |

### 10.6 Dependency and supply-chain policy

- Direct deps allowlist (anything else requires an ADR): `filippo.io/age`, `golang.org/x/crypto` (SSHSIG), `gopkg.in/yaml.v3`, `github.com/wneessen/go-mail`, one QR lib (`github.com/skip2/go-qrcode` or `github.com/yeqown/go-qrcode/v2` — decided in issue 14), stdlib. Test-only: `github.com/google/go-cmp`.
- `internal/shamir` vendored from `hashicorp/vault` tag **v1.14.8** (last MPL-2.0), headers + `LICENSES/MPL-2.0.txt` + provenance in `NOTICE` (research 02).
- CI: `govulncheck`, `golangci-lint`, `gitleaks`, tests with `-race`; all third-party actions pinned to commit SHAs; workflow `permissions:` least-privilege.
- Go toolchain pinned via `go.mod` `toolchain` directive; releases built by CI only.
- Project license: **Apache-2.0** (pending owner confirmation — see ISSUE_PLAN known unknowns KU-6) with MPL-2.0 for vendored files.

### 10.7 Privacy

Recipient names/emails exist in: private trigger repo (A-7, GitHub-scoped), mails, and printed artifacts. Never in the OSS repo. `keepsake status --json` redacts emails unless `--show-pii`.

---

## 11. Trigger Repository and GitHub Actions Workflows

### 11.1 `monitor.yml` (normative sketch)

```yaml
name: keepsake-monitor
on:
  schedule:
    - cron: "23 21 * * *"          # daily ~06:23 JST; off-peak minute (C3)
  workflow_dispatch:
    inputs:
      drill: {description: "Run as drill (no real shares, TEST-marked mails)", type: boolean, default: false}
permissions:
  contents: write                   # state heartbeat commit
  actions: read                     # dispatch check-in verification
concurrency:
  group: keepsake-monitor
  cancel-in-progress: false
jobs:
  monitor:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@<pinned-SHA>
      - name: Verify binary integrity
        run: sha256sum -c bin/keepsake-linux-amd64.sha256
      - name: Run monitor
        env:
          KEEPSAKE_SHARE_G: ${{ secrets.KEEPSAKE_SHARE_G }}
          KEEPSAKE_SMTP_URL: ${{ secrets.KEEPSAKE_SMTP_URL }}
          KEEPSAKE_SMTP_URL_SECONDARY: ${{ secrets.KEEPSAKE_SMTP_URL_SECONDARY }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: ./bin/keepsake-linux-amd64 monitor --repo-dir . ${{ inputs.drill && '--drill' || '' }}
```

### 11.2 `checkin-button.yml`

```yaml
name: keepsake-checkin
on: workflow_dispatch
permissions: {}                     # ZERO permissions; the run record itself is the check-in
jobs:
  checkin:
    runs-on: ubuntu-latest
    timeout-minutes: 2
    steps:
      - run: echo "Check-in recorded via workflow run (actor is verified by monitor through the Actions API)"
```

The monitor verifies dispatch check-ins exclusively through `GET /repos/{repo}/actions/workflows/checkin-button.yml/runs` filtering `triggering_actor.login == owner.github_login` (server-side fact, unforgeable via repo contents; research 03 C6).

### 11.3 Monitor execution sequence

1. Load + validate `keepsake.yaml` (fail-closed: on invalid config, mail owner error if SMTP config itself is valid, exit non-zero, make NO release decisions).
2. Refuse to proceed if forbidden files present (`kit.age`, `share-R*`) — §10.2.
3. Gather check-in facts: verify `state/checkin.json(.sig)` (§6.7); query dispatch runs since `state.last_run_at − 48h`; compute `last_valid_checkin = max(signed, dispatch)`.
4. `Evaluate()` (§9) with `now = time.Now().UTC()`; execute returned actions (mail sends, §12) — send-then-persist.
5. Persist `state/state.json`, commit (`keepsake-monitor: <phase> day <elapsed>`) and push — this commit is also the heartbeat (research 03 C1/C2). Push conflict → fetch/rebase once → retry once → else fail (next run recovers).
6. Exit non-zero iff an action failed (visible in Actions UI + GitHub failure notification to owner as extra signal).

### 11.4 Drill semantics

`--drill`: full pipeline with (a) subjects prefixed `[DRILL]` and a leading explanation block, (b) share_G replaced by `KEEPSAKE-G1-TEST...` placeholder (never reads the real secret), (c) forced phase walk: sends one owner reminder, one recipient safety-check, one release-template mail to every recipient, (d) no state.json mutation except `history` gets a `DRILL_RUN` entry (committed — doubles as heartbeat), (e) exit code reflects delivery success per recipient. Runbook: annual drill (owner calendar), verify every recipient confirms receipt.

---

## 12. Mail Delivery

- Sender: `internal/mail` using go-mail; SMTP endpoint parsed from `KEEPSAKE_SMTP_URL` (`smtps://user:pass@host:port` or `smtp+starttls://…`); on send failure of the batch → retry once after 30s → try `KEEPSAKE_SMTP_URL_SECONDARY` if set; per-recipient outcome recorded (state.json), failed recipients retried next daily run.
- All owner-bound and recipient-bound templates (Japanese primary + short English footer): `remind_owner`, `checkin_reset`, `alert_recipients`, `release`, `postrelease_confirm`, `released_but_alive`, `config_error_owner`, plus `[DRILL]` variants. Templates embedded (`go:embed`), rendered with `text/template` (plain text mails only — no HTML mail in v1: maximizes deliverability + carrier compatibility).
- Release mail body contains: what happened (plain Japanese), share_G string + QR? (QR in mail = attachment/PNG — skip; text only), the 4-char CHECK context sentence, exact `keepsake open` steps, `age` fallback pointer, "no one will ever ask you to send this back" warning, owner-configured `mail.from` consistency note.
- Every recipient-bound mail BCCs the owner (§10.5 A6 detection).
- Header injection prevention: addresses validated at load AND at compose; template values that end up in headers are restricted to validated config fields.

---

## 13. Failure Modes, Recovery, Known Unknowns

### 13.1 Failure modes table

| ID | Failure | Detection | Response (v1) |
|---|---|---|---|
| F1 | Cron delayed/skipped (platform load) | next run's day-granular recompute | none needed; design absorbs (§9) |
| F2 | SMTP outage at release | per-recipient send record | daily retry until success; secondary provider; postrelease confirms |
| F3 | GitHub Actions outage | no runs | next successful run catches up; drill validates annually |
| F4 | Vendored binary corrupted | sha256 step fails, workflow red | owner alerted via GitHub failure mail; `trigger update` re-vendors |
| F5 | state.json push race | push rejected | rebase-retry-once; next run recovers (idempotent) |
| F6 | Trigger repo deleted / account lost after owner death | none (silent) | ultimate backstop: Recovery Sheet in safe via probate; documented in recipient guide ("if no mail ever arrives") |
| F7 | Owner locked out of GitHub while alive | reminder mails keep arriving | owner restores account before day 42; worst case: warn recipients to ignore release (rotate after) |
| F8 | Recipient email dead/spam-foldered | drill (annual) + postrelease confirms ×4 | fix address, re-drill; multiple recipients reduce single-point risk |
| F9 | Owner loses SSH signing key | `checkin` fails locally | runbook: generate new key, update `allowed_signers` in config, `trigger update`, push |
| F10 | Owner loses laptop/vault | — | bundle + Recovery Sheet unaffected; rebuild vault from templates; rotate if theft suspected |
| F11 | Recipient loses bundle | owner notices at annual drill conversation | re-issue identical bundle; rotate if theft suspected |
| F12 | False release (owner alive) | T6 owner mail + BCC copies | rotation runbook (new KEK, re-seal, re-distribute, new share_G secret) |
| F13 | KEK sheet destroyed (house fire) | owner discovers | shares still work (bundle + GitHub); re-print sheet from `seal-manifest` — wait: sheet = P = KEK; regenerate via `keepsake guide` from seal-manifest (share_G copy + share_R needed → owner holds both in vault state/ + bundle) |
| F14 | Payload drift (accounts changed, kit stale) | `status` drift warning | re-seal + re-distribute (runbook cadence: annually or on major life event) |
| F15 | Actions minutes quota exhausted (owner's other private repos consumed the monthly free quota; over-quota private-repo usage is blocked without a payment method) | monitor runs skipped near month-end; Actions UI shows blocked runs | day-granular recompute catches up when quota resets (≤1 month delay worst case, within release tolerances); runbook guidance: keep the trigger repo on an account with minimal other private Actions usage; drill verifies |

### 13.2 Known unknowns (may create issues during implementation)

| ID | Unknown | Impact if adverse |
|---|---|---|
| KU-1 | GitHub policy drift on Actions free tier / cron / dormant accounts over multi-year horizon | trigger substrate migration (state machine is portable by design) |
| KU-2 | SMTP deliverability to specific JP mailbox providers (carrier domains warned against; Gmail/iCloud assumed OK) | may need provider-specific guidance or a second channel (v2: LINE/webhook) |
| KU-3 | QR library final selection (maintenance, print-DPI quality) | swap between the two MIT candidates |
| KU-4 | Japanese rendering fidelity of printable HTML across OS/browsers (fonts) | may need embedded font subset (license-checked) |
| KU-5 | Windows SmartScreen / macOS Gatekeeper friction for `tools/` binaries | more guide detail; v2 signing/notarization |
| KU-6 | Project license final confirmation (Apache-2.0 proposed) | ADR-002/NOTICE update; must be settled before first public release |
| KU-7 | `age` CLI availability/behavior drift for the interop CI gate | pin age version in CI; the format spec is the real contract |

---

## 14. Operational Runbooks (summaries; full docs in issue 29)

| Runbook | Trigger | Core steps |
|---|---|---|
| Initial setup | first use | init → fill payload → seal → create private repo → trigger init → push → set 2–3 secrets manually → checkin → drill |
| Annual drill | calendar | `keepsake drill` → confirm receipt with every recipient verbally → fix addresses → log date |
| Rotation | compromise/false release/recipient change | re-`seal` (new KEK) → re-distribute bundles → update KEEPSAKE_SHARE_G secret → destroy old Recovery Sheet → print new one |
| Re-seal (content update) | payload drift | same as rotation (KEK always rotates on seal — simpler and safer than KEK reuse) |
| Restore vault | lost machine | new machine → install → init → re-import payload from sources → rotate |
| Decommission | product exit | disable workflows → delete secrets → shred sheets → collect/destroy bundles → delete trigger repo |

Recipient-side "runbook" is the printed guide itself (issue 14/15): what this is, when mail arrives, `keepsake open` steps per OS, `age` fallback, "if no mail ever arrives → owner's safe / probate", anti-phishing rules, whom to call (helper line).

---

## 15. Glossary and Traceability

| Term | Definition |
|---|---|
| **KEK** | 32-byte random key-encryption key; its sharetext encoding is the recovery passphrase **P** |
| **share_R / share_G** | Shamir 2-of-2 shares of KEK held by Recipients (physical) / GitHub (Actions secret) |
| **kit.age** | age v1 scrypt-mode ciphertext of the payload tar |
| **bundle** | USB-ready directory: kit.age + share-R + guides + tools |
| **vault** | owner-local working directory (plaintext payload + config + state) |
| **trigger repo** | private GitHub repo running the dead man's switch |
| **seal** | payload → kit.age + shares + bundle build |
| **drill** | test-fire with placeholder share and [DRILL] mails |
| **Recovery Sheet** | printed P (KEK) stored in owner's safe |

Design-section → issue coverage matrix lives in `docs/ISSUE_PLAN.md`. ADRs: 001 (Go), 002 (age+Shamir), 003 (GitHub Actions substrate), 004 (physical distribution / share-only release), 005 (check-in channels + signatures), 006 (SMTP delivery).
