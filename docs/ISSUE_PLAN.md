# keepsake — Issue Plan (v1)

Status: DRAFT for review · 2026-07-10
Derived from: `docs/DESIGN.md` (canonical). GitHub Issues are generated from `docs/issues/*.md` and are stale derived artifacts whenever they disagree with these files.

## 1. v1 completion statement

v1 is complete when **all 31 issues below are closed and validated**, at which point the following is true end-to-end using only released artifacts and repository documentation (DESIGN §3.1):

1. Owner creates a vault, fills Japanese payload templates, and `keepsake seal` produces a USB-ready recipient bundle + printable Japanese guides (recipient guide, owner Recovery Sheet).
2. Owner scaffolds a private trigger repo with `keepsake trigger init`, sets the documented Actions secrets manually, and the monitor runs daily on GitHub Actions.
3. Check-in works via `keepsake checkin` (SSHSIG-verified) and via the phone workflow_dispatch button (Actions-API-verified).
4. On check-in silence the switch walks REMINDING → ALERTING → RELEASED per DESIGN §9 and emails share_G to every recipient; a recipient following only the printed guide decrypts the kit with `keepsake open` (and the Recovery-Sheet path decrypts with the standard `age` CLI).
5. An annual drill exercises schedule → binary integrity → SMTP to every recipient → templates, with zero real key material.
6. Every failure mode F1–F14 (DESIGN §13.1) has a tested behavior or a documented runbook.

No important v1 product behavior exists outside this plan; discovery during implementation goes through new issues derived from DESIGN.md updates.

## 2. Issue list (recommended execution order)

| # | File | Title | Wave |
|---|---|---|---|
| 01 | `issues/01-repo-scaffolding.md` | Repository scaffolding: Go module, layout, Makefile, licenses | 0 |
| 02 | `issues/02-ci-pipeline.md` | CI pipeline: build, test, lint, vuln/secret scanning, pinned actions | 0 |
| 03 | `issues/03-vendor-shamir.md` | Vendor Shamir secret sharing from Vault v1.14.8 (MPL-2.0) | 0 |
| 04 | `issues/04-sharetext-codec.md` | sharetext codec: Crockford Base32 key-material encoding | 0 |
| 05 | `issues/05-age-wrapper.md` | age encryption wrapper (scrypt recipient, streaming) | 0 |
| 06 | `issues/06-kek-service.md` | KEK service: generate, split, combine, fingerprint | 0 |
| 07 | `issues/07-config-schema.md` | Config schemas and strict validation (vault + trigger) | 0 |
| 08 | `issues/08-cli-skeleton.md` | CLI skeleton: subcommand router, global flags, bilingual printer | 1 |
| 09 | `issues/09-japanese-content-pack.md` | Japanese content pack: payload templates, bundle README, guide text | 1 |
| 10 | `issues/10-init-command.md` | `keepsake init`: vault creation with safety checks | 1 |
| 11 | `issues/11-payload-tar.md` | Payload manifest and deterministic tar builder | 1 |
| 12 | `issues/12-bundle-builder.md` | Recipient bundle builder and manifests | 1 |
| 13 | `issues/13-seal-command.md` | `keepsake seal`: end-to-end kit sealing pipeline | 1 |
| 14 | `issues/14-open-command.md` | `keepsake open`: recipient decryption wizard with hardened extraction | 1 |
| 15 | `issues/15-verify-command.md` | `keepsake verify`: owner self-test | 1 |
| 16 | `issues/16-guide-generator.md` | Printable HTML guide generator with QR codes | 1 |
| 17 | `issues/17-checkin-statement.md` | Check-in statement format and SSHSIG sign/verify | 2 |
| 18 | `issues/18-checkin-command.md` | `keepsake checkin`: sign, commit, push | 2 |
| 19 | `issues/19-state-machine.md` | Dead man's switch state machine engine | 2 |
| 20 | `issues/20-mail-sender.md` | Mail composer and SMTP sender with failover | 2 |
| 21 | `issues/21-mail-templates.md` | Mail template content (Japanese/English, all kinds + drill variants) | 2 |
| 22 | `issues/22-github-api-client.md` | Minimal GitHub API client for dispatch check-in verification | 2 |
| 23 | `issues/23-monitor-command.md` | `keepsake monitor`: trigger-side orchestration | 2 |
| 24 | `issues/24-trigger-init.md` | `keepsake trigger init`: trigger repo scaffolding | 2 |
| 25 | `issues/25-trigger-update.md` | `keepsake trigger update`: binary/workflow refresh | 2 |
| 26 | `issues/26-drill-mode.md` | Drill mode: test-fire semantics and `keepsake drill` | 2 |
| 27 | `issues/27-e2e-timeline-test.md` | End-to-end timeline integration test (fake clock, mock SMTP) | 3 |
| 28 | `issues/28-security-hardening.md` | Security hardening pass: no-secret-logging gate, fuzzing, SECURITY.md | 3 |
| 29 | `issues/29-release-engineering.md` | Release engineering: goreleaser, checksums, install docs | 3 |
| 30 | `issues/30-owner-docs-runbooks.md` | Owner documentation and operational runbooks | 3 |
| 31 | `issues/31-oss-repo-hardening.md` | OSS repository hardening (rulesets, scanning, Dependabot, CodeQL) | 3 |

## 3. Dependency table

`A ← B` means A depends on (is blocked by) B.

| Issue | Blocked by | Notes |
|---|---|---|
| 01 | — | root |
| 02 | 01 | needs Makefile targets |
| 03 | 01 | |
| 04 | 01 | |
| 05 | 01 | |
| 06 | 03, 04, 05 | combines shamir + codec + passphrase encoding |
| 07 | 01 | |
| 08 | 01, 07 | global flags load config |
| 09 | 01 | content only, embedded later |
| 10 | 08, 09 | init embeds templates |
| 11 | 01 | |
| 12 | 06, 09, 11 | manifests, share files, README inclusion |
| 13 | 07, 12 | orchestration; also implements verified release download (§8.1) |
| 14 | 04, 05, 06, 08, 12 | reads bundle manifest |
| 15 | 12, 13 | verifies seal outputs |
| 16 | 09, 13 | QR + data from seal-manifest |
| 17 | 01 | |
| 18 | 08, 17 | |
| 19 | 07 | pure engine |
| 20 | 07 | address validation reuse |
| 21 | 20 | fills template interface defined in 20 |
| 22 | 01 | |
| 23 | 08, 17, 19, 20, 21, 22 | glue |
| 24 | 07, 23 | workflow files reference monitor flags |
| 25 | 24 | |
| 26 | 23, 24 | |
| 27 | 13, 14, 23 | |
| 28 | 13, 14, 23 | audits existing surfaces |
| 29 | 01, 02 | produces artifacts consumed at runtime by 13/24 (no build-order dep) |
| 30 | 26 | documents final flows |
| 31 | 02 | |

## 4. Implementation waves

| Wave | Issues | Parallelism guidance | Exit criteria |
|---|---|---|---|
| 0 Foundation | 01–07 | 01 first; then 02–05, 07 in parallel; 06 last | `make build test lint` green; crypto primitives unit-tested with vectors |
| 1 Kit lifecycle | 08–16 | 08, 09, 11 in parallel; then 10, 12; then 13; then 14–16 | full local flow: init → seal → verify → open round-trip on a sample payload |
| 2 Switch | 17–26 | 17, 19, 20, 22 in parallel; then 18, 21; then 23; then 24–26 | monitor runs against a fixture trigger repo locally (`--now` injected) through all phases |
| 3 Hardening & release | 27–31 | 27, 28, 29, 31 in parallel; 30 last | e2e CI job green; first tagged pre-release built; drill executed against a real private repo |

## 5. Coverage table (DESIGN.md section → issues)

| DESIGN section | Issue(s) |
|---|---|
| §2 Concept/personas/language | 09, 16, 30 |
| §3 Scope | this plan |
| §4.1–4.2 Architecture/repos | 24, 30 |
| §4.3 Module layout | 01 |
| §5.1 Key hierarchy | 06 |
| §5.2 Encryption format | 05, 13 |
| §5.3 Randomness/zeroization | 06, 28 |
| §5.4 Long-term decryptability | 05 (interop test), 29 (tools), 30 (fallback docs) |
| §6.1 Vault layout | 10 |
| §6.2 Deterministic tar | 11 |
| §6.3 Bundle layout | 12 |
| §6.4 config.yaml | 07 |
| §6.5 keepsake.yaml | 07, 24 |
| §6.6 Manifests | 12 |
| §6.7 Check-in statement | 17, 18 |
| §6.8 Monitor state | 19, 23 |
| §7 sharetext | 04 (codec), 16 (QR), 21 (anti-phishing text) |
| §8 CLI surface | 08, 10, 13, 14, 15, 16, 18, 23, 25, 26 |
| §8.1 Binary provisioning | 13, 25 |
| §9 State machine | 19, 27 |
| §10.1–10.2 Assets/trust boundaries | 28, 30 (docs), enforced across issues |
| §10.3 Secret handling | 06, 13, 20, 23, 28 (CI gate) |
| §10.4 Input validation | 04, 07, 14, 17, 20, 22, 28 |
| §10.5 Abuse cases | A1/A2→17,23 · A3→06 · A7→16,21 · A8→02,31 · rest→30 (runbooks) |
| §10.6 Dependency policy | 01, 02, 03, 31 |
| §10.7 Privacy | 08 (`--show-pii`), 24 |
| §11.1–11.3 Workflows/monitor | 22, 23, 24 |
| §11.4 Drill | 26 |
| §12 Mail | 20, 21 |
| §13.1 Failure modes | F1–F5,F15→19/23 · F4→24/25 · F8→26 · F9–F15→30 |
| §13.2 Known unknowns | tracked below §8 |
| §14 Runbooks | 30 |

## 6. Validation strategy (whole product)

| Layer | Mechanism | Where |
|---|---|---|
| Crypto primitives | vendored upstream test vectors (shamir), round-trip + `age` CLI interop vector | 03, 05, CI |
| Codecs/parsers | golden vectors + Go native fuzzing (sharetext, config, checkin statement) | 04, 07, 17, 28 |
| State machine | exhaustive table-driven tests over phase × elapsed × dedup matrix incl. missed-cron gaps | 19 |
| Extraction safety | negative-case suite (traversal, symlink, bomb) | 14, 28 |
| Secret hygiene | CI gate greping full-verbose run output for planted test secrets | 28 |
| Integration | e2e fake-clock timeline: seal → checkins → silence → remind → alert → release → open, mock SMTP capture | 27 |
| Platform | annual drill runbook against real GitHub + real SMTP | 26, 30 |
| Supply chain | pinned actions audit, govulncheck, gitleaks, dependabot, CodeQL | 02, 31 |
| Docs | every runbook step executed once on a clean machine before v1 tag | 30 |

## 7. Deferred v2 items

Static web decryptor (browser-only `open`) · k-of-n human trustee shares · per-recipient payloads · pluggable trigger substrates (GitLab CI etc.) · macOS notarization / Windows signing · cosign/SLSA provenance · localization framework beyond ja/en · encrypted vault cloud sync · secondary notification channels (LINE/webhook) · Recovery-Sheet-free full paper recovery (printable ciphertext for small payloads).

## 8. Known unknowns → potential new issues

| KU (DESIGN §13.2) | Trigger for new issue |
|---|---|
| KU-1 GitHub policy drift | drill failure or docs change → substrate abstraction issue |
| KU-2 JP deliverability | drill bounce/spam evidence → provider guidance or channel issue |
| KU-3 QR lib selection | decided inside issue 16; if both fail print-DPI QA → new issue |
| KU-4 JP font fidelity in print | issue 16 QA fails → embedded-font issue (license check) |
| KU-5 Gatekeeper/SmartScreen friction | recipient-test feedback → v2 signing issue |
| KU-6 License confirmation (Apache-2.0 proposed) | owner decision → may amend 01 (LICENSE) before public release |
| KU-7 age CLI drift in CI | pin/update strategy inside 05; breakage → maintenance issue |
