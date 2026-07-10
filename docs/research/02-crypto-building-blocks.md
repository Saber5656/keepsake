# Research: Cryptographic Building Blocks

Status: verified 2026-07-10. This research materially affects DESIGN.md section "Cryptographic Design" and ADR-002.

## 1. File encryption: age (filippo.io/age)

- Spec: [age-encryption.org/v1](https://age-encryption.org/v1); reference Go implementation [filippo.io/age](https://pkg.go.dev/filippo.io/age), stable at major version v1 ([repo](https://github.com/FiloSottile/age)).
- Passphrase mode uses scrypt: [`ScryptRecipient`](https://pkg.go.dev/filippo.io/age#ScryptRecipient) / [`ScryptIdentity`](https://pkg.go.dev/filippo.io/age#ScryptIdentity). `SetWorkFactor(logN)` tunes cost (default is reasonable; we pin an explicit value for reproducibility).
- Payload encryption is ChaCha20-Poly1305 (AEAD) with a MAC'd header — tampering with ciphertext is detected at decrypt time.
- **Why it matters for keepsake**: if the kit ciphertext is a *standard* age scrypt-mode file, then anyone holding the passphrase P (the Recovery Sheet path) can decrypt with ANY age implementation (`age`, `rage`, ports) decades from now, even if the keepsake project disappears. This directly serves the 可用性 (availability / "must not become undecryptable") threat-model requirement. (The share_R+share_G release path still needs a keepsake binary to combine shares — bundles therefore carry binaries for 4 platforms; see DESIGN §5.4.)

Decision input → ADR-002: encrypt the kit payload as a standard age v1 scrypt-recipient file whose passphrase is the word/base32-encoded KEK. Verify interop against the `age` CLI in CI.

Rejected alternatives:

| Alternative | Rejection reason |
|---|---|
| Raw AES-GCM + custom container | Custom format = long-term decode risk; no third-party tooling for recipients. |
| OpenPGP (gpg) | Complex UX, agent/keyring assumptions unfit for non-technical recipients; format sprawl. |
| X25519 age identities per recipient | Requires recipients to hold private keys safely for years; passphrase+Shamir keeps recipient-side material printable on paper. |

## 2. Secret splitting: Shamir's Secret Sharing

Requirement: split the KEK so that the GitHub-side share and the recipient-side share are individually useless (2-of-2 in v1), with a format that can grow to k-of-n (human trustees) in v2 without redesign.

### Licensing landscape (verified)

- HashiCorp relicensed Vault (including new versions of its `shamir` package) from MPL-2.0 to **BUSL-1.1 on 2023-08-10** ([announcement](https://www.hashicorp.com/en/blog/hashicorp-adopts-business-source-license), [FAQ](https://www.hashicorp.com/en/license-faq)). pkg.go.dev now marks current `github.com/hashicorp/vault/shamir` as **not redistributable** for documentation purposes ([pkg.go.dev](https://pkg.go.dev/github.com/hashicorp/vault/shamir)).
- The last MPL-2.0 Vault line is **v1.14.x** (final: v1.14.8); [OpenBao](https://thenewstack.io/meet-openbao-an-open-source-fork-of-hashicorp-vault/) forked from v1.14.0 and keeps MPL-2.0.
- The `shamir` package is a subpackage of the huge `github.com/hashicorp/vault` Go module — importing it as a module dependency would drag in the entire Vault dependency tree AND (for current versions) the BUSL license.

### Options considered

| Option | License | Size/deps | Assessment |
|---|---|---|---|
| Vendor `shamir/` files from Vault tag `v1.14.8` (pre-BUSL) | MPL-2.0 (file-level copyleft, OK to combine with Apache-2.0 project when files kept intact & noticed) | 2 files, stdlib-only | **Chosen.** Battle-tested GF(2^8) implementation used to protect Vault master keys for a decade; includes test vectors. |
| Import [`corvus-ch/shamir`](https://github.com/corvus-ch/shamir) | MPL-2.0 | Small standalone module | Viable fallback; same lineage (based on Vault's implementation), less review traffic. |
| Import OpenBao module | MPL-2.0 | Huge module | Same dependency-tree problem as Vault. |
| Write our own | — | — | Rejected: hand-rolled crypto in a product whose whole point is decade-scale reliability. |

Vendoring rules (→ issue): copy `shamir.go` + `shamir_test.go` from `hashicorp/vault` tag `v1.14.8` into `internal/shamir/`, preserve MPL-2.0 headers, add `LICENSES/MPL-2.0.txt`, record provenance (upstream URL + tag + commit SHA) in `NOTICE`.

Properties of the Vault implementation relevant to our formats: secret is split byte-wise over GF(2^8); each share is `len(secret)+1` bytes (last byte is the x-coordinate); `Split(secret, n, threshold)` / `Combine(shares)`; maximum 255 shares. A 32-byte KEK yields 33-byte shares.

## 3. Human-transportable encoding of key material

Shares and the recovery passphrase must survive **paper and typing by a non-technical Japanese speaker**.

| Encoding | Pros | Cons |
|---|---|---|
| **Crockford Base32, grouped, with checksum (chosen)** | No ambiguous chars (no I/L/O/U), case-insensitive input, compact (33 bytes → 53 chars), language-neutral (JP users type Latin letters more reliably than English words) | Less redundancy than word lists |
| BIP39 word list | Error-detecting via checksum, familiar from crypto wallets | English words are unfriendly to JP non-technical users; 24-word phrases get mis-transcribed |
| Plain hex/base64 | Trivial | Ambiguous characters; base64 case-sensitivity is hostile to manual entry |

Chosen format (normative details in DESIGN.md §7): versioned prefix + role + grouped Crockford Base32 + truncated-SHA-256 checksum group. Both printed as text AND as a QR code — recipients scan the QR with any phone camera instead of typing, with typing as the fallback.

QR generation library candidates (final vetting in the implementing issue): [`skip2/go-qrcode`](https://github.com/skip2/go-qrcode) (MIT, widely used, API-stable) or `yeqown/go-qrcode` (MIT, more active). Constraint: pure Go, no cgo, output PNG bytes embeddable as data-URI in generated HTML.

## 4. Mail delivery library

Requirement: send SMTP mail from GitHub Actions (reminders, alerts, release). `net/smtp` is API-frozen and lacks modern auth/TLS ergonomics.

- [`wneessen/go-mail`](https://github.com/wneessen/go-mail): actively maintained (release observed July 2026), stdlib-first with a minimal curated dependency set, supports SMTP AUTH mechanisms, STARTTLS/implicit TLS. **Chosen** (→ ADR-006).
- Provider-agnostic SMTP keeps the design portable across Gmail app passwords, SES SMTP, Resend SMTP, Mailgun, etc. Provider choice is the owner's; credentials live only in GitHub Actions secrets, set manually by the owner.

## 5. Check-in authenticity: SSH signatures

Requirement: check-in statements pushed to the trigger repo must be verifiable as owner-authored, so that stolen push credentials alone cannot forge liveness (defense in depth; see DESIGN.md §10 abuse cases).

- SSH signature format (`ssh-keygen -Y sign` / SSHSIG) is verifiable in pure Go via `golang.org/x/crypto/ssh`; owners already have SSH keys. No GPG dependency.
- Replay protection: monotonic counter + timestamp inside the signed statement; the monitor rejects statements older than the last accepted one.

## Sources

- https://pkg.go.dev/filippo.io/age
- https://github.com/FiloSottile/age
- https://pkg.go.dev/github.com/hashicorp/vault/shamir
- https://www.hashicorp.com/en/blog/hashicorp-adopts-business-source-license
- https://www.hashicorp.com/en/license-faq
- https://github.com/hashicorp/vault/issues/22318
- https://thenewstack.io/meet-openbao-an-open-source-fork-of-hashicorp-vault/
- https://github.com/corvus-ch/shamir
- https://github.com/wneessen/go-mail
- https://github.com/skip2/go-qrcode
