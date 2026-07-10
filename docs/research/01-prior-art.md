# Research: Prior Art — Dead Man's Switches and Digital Legacy Tools

Status: verified 2026-07-10. This research materially affects DESIGN.md sections "Product Concept", "Trigger Architecture", and "Threat Model".

## Purpose

Survey existing dead man's switch (DMS) and digital-legacy products to (a) avoid reinventing solved problems, (b) identify failure modes observed in the field, and (c) position keepsake's differentiation.

## Comparison table

| Product | Type | Trigger | Encryption | Survivability after owner death | Recipient UX | Notes |
|---|---|---|---|---|---|---|
| [Google Inactive Account Manager](https://alcazarsec.com/deadmanswitch/alternatives/google-inactive-account-manager) | Hosted (free) | Account inactivity (3–18 months) | Server-side only; Google reads everything | High (Google-operated) | Excellent (email link) | Unlocks Google account data only. No client-side encryption. Cannot carry external secrets safely. |
| Apple Digital Legacy (Legacy Contact) | Hosted (free) | Death certificate + access key | Server-side | High | Good (manual process with Apple) | Apple-ecosystem data only. Human review process, weeks of delay. |
| [Bitwarden Emergency Access](https://bitwarden.com/help/emergency-access/) | Hosted (paid premium) | Recipient request + owner veto window (wait-days) | End-to-end (public-key exchange, zero-knowledge) | High while subscription is paid | Good (Bitwarden account required) | The strongest crypto model among hosted tools. Trigger is recipient-initiated, not inactivity-based. Subscription lapse after death is a real risk. |
| [LastSignal](https://www.linuxtoday.com/blog/lastsignal-is-a-new-open-source-dead-mans-switch-you-can-self-host/) | OSS, self-hosted | Missed activity checks | Browser-side encryption of messages | **Low — dies with the owner's server** | Web | Requires an always-on server the owner pays for and maintains. |
| [Seppuku](https://github.com/Netdex/Seppuku) | OSS, self-hosted (.NET) | Countdown timer requiring resets | None built-in (module system) | **Low — same self-hosting paradox** | N/A | Extensible module actions on fire. |
| Aeterna ([overview](https://www.androidauthority.com/home-server-dead-man-switch-3648903/)) | OSS, self-hosted (Docker) | Missed check-ins | Limited | **Low — same self-hosting paradox** | Web | Marketed for home-server owners. |
| Hosted DMS services (e.g. deadmansswitch.net, [Cipherwill](https://www.cipherwill.com/blog/step-by-step-guide-to-dead-mans-switch-setup-21b6d636261880f49b96ecb548780f09)) | Hosted SaaS | Email check-ins | Varies, mostly server-side | Medium (company lifetime risk) | Good | Operator must be trusted with content or keys; company shutdown risk over decade horizons. |

More OSS examples under the [`dead-mans-switch` GitHub topic](https://github.com/topics/dead-mans-switch).

## Field-observed failure modes (inputs to our design)

1. **The self-hosting paradox.** A DMS hosted on infrastructure the owner pays for (VPS, home server) tends to die *before or with* the owner — power, billing, hardware. The trigger must outlive the owner. This rules out the "自宅サーバ/VPS cron" architecture and motivates using GitHub Actions free tier (no billing tied to owner's continued action) — see ADR-003.
2. **The subscription paradox.** Hosted paid products (Bitwarden Premium) lapse when payments stop after death. Free tiers of very large platforms are more survivable.
3. **Server-side trust.** Google IAM-style products require trusting the operator with plaintext. Our threat model ("第三者サービス運営者" selected by the owner) forbids this: no single third party may hold both ciphertext and enough key material.
4. **False-fire embarrassment vs. missed-fire disaster.** Products with a single reminder channel false-fire during hospital stays/travel. Mitigations adopted: dual check-in channels (CLI + phone-tappable workflow_dispatch), escalating reminders, a recipient "safety-check" phase *before* release, and generous default grace periods.
5. **Recipient-side complexity kills real-world recovery.** Tools that assume the recipient can operate a server or a CLI without guidance fail with non-technical families. Adopted: physically pre-distributed bundle with printed Japanese step-by-step guide, QR-encoded key material, bundled decrypt binaries for 4 platforms, and a standard age-format ciphertext so the Recovery-Sheet (passphrase) path stays decryptable with third-party tools decades later.

## Positioning of keepsake

keepsake occupies a niche none of the above covers:

- **Serverless dead man's switch**: scheduled execution on GitHub Actions (free tier, survives owner inactivity; see research 03).
- **True end-to-end secrecy with split trust**: GitHub holds only one Shamir share; recipients physically hold ciphertext + the other share. No single party can decrypt alone (see research 02, ADR-002/004).
- **Non-technical recipient UX as a first-class requirement**: printed Japanese guide, QR codes, standard-format ciphertext (`age`) that any future tool can decrypt given the Recovery-Sheet passphrase.
- **終活 (end-of-life preparation) framing**: templates for account inventories, wishes, and letters — not just raw secret release.

## Sources

- https://docs.github.com/actions/managing-workflow-runs/disabling-and-enabling-a-workflow
- https://bitwarden.com/help/emergency-access/
- https://alcazarsec.com/deadmanswitch/alternatives/google-inactive-account-manager
- https://alcazarsec.com/deadmanswitch/alternatives/bitwarden-emergency-access
- https://www.linuxtoday.com/blog/lastsignal-is-a-new-open-source-dead-mans-switch-you-can-self-host/
- https://github.com/Netdex/Seppuku
- https://www.androidauthority.com/home-server-dead-man-switch-3648903/
- https://github.com/topics/dead-mans-switch
- https://www.cipherwill.com/blog/step-by-step-guide-to-dead-mans-switch-setup-21b6d636261880f49b96ecb548780f09
