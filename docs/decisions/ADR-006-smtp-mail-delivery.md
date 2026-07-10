# ADR-006: Provider-agnostic SMTP delivery from within the keepsake binary

Status: Accepted · 2026-07-10

## Context

The switch must email the owner (reminders), recipients (safety-check alert, release). Delivery runs inside GitHub Actions. Options: third-party marketplace mail actions, provider REST APIs (Resend/SendGrid), or SMTP from our own binary. See research 03.

## Decision

- Mail is sent **from within the `keepsake monitor` binary** using [`wneessen/go-mail`](https://github.com/wneessen/go-mail) (actively maintained, stdlib-first).
- Endpoint + credentials come from a single Actions secret **`KEEPSAKE_SMTP_URL`** (`smtps://user:pass@host:port` or `smtp+starttls://user:pass@host:port`), optional **`KEEPSAKE_SMTP_URL_SECONDARY`** as an automatic fallback provider. Secrets are set manually by the owner (operator policy).
- Plain-text mails only (Japanese primary + English footer): best deliverability, no HTML-rendering variance, nothing to phish-clone pixel-perfectly.
- Per-recipient send outcomes are recorded in state.json; failures retry on subsequent daily runs; release mail re-confirms on a schedule (DESIGN §9 T5).

## Alternatives rejected

| Alternative | Why rejected |
|---|---|
| Marketplace mail actions (e.g. action-send-mail) | third-party action in the most security-critical workflow = supply-chain exposure; logic untestable in unit tests |
| Provider REST APIs | binds the design to one company's API lifetime; SMTP is the 40-year-stable interface any provider (Gmail app password, SES, Mailgun, Resend SMTP) exposes |
| GitHub Issues/notifications as the channel | recipients are non-technical and have no GitHub accounts |

## Consequences

- Deliverability to JP carrier domains is a known unknown (KU-2): config lints against carrier addresses; the annual drill sends real end-to-end mail to every recipient.
- SMTP URL parsing + header-injection prevention are explicit validation boundaries (DESIGN §10.4) with dedicated tests.
- If both providers are down, retries continue daily — acceptable because release timing tolerates multi-day delay by design.
