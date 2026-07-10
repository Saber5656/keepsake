# ADR-001: Implementation language is Go

Status: Accepted · 2026-07-10 · Confirmed with owner

## Context

keepsake is a security-critical CLI + a binary executed inside GitHub Actions for years without maintenance. Implementation work is delegated to lower-capability coding agents, so the language must minimize the ways an implementation can be subtly wrong. Candidates: Go, Rust, TypeScript/Node.

## Decision

Go (latest stable toolchain, pinned via `go.mod` `toolchain` directive). Module `github.com/Saber5656/keepsake`, single static binary per platform (`CGO_ENABLED=0`).

## Rationale

- The reference implementation of age (`filippo.io/age`) is a Go library — we use the exact code most audited against the spec, not a port.
- The de-facto standard audited Shamir implementation (HashiCorp Vault's) is Go and vendorable (ADR-002).
- Static cross-compilation for darwin/windows/linux from one Makefile — required for the recipient `tools/` directory and the vendored trigger-repo binary.
- Simple language semantics: mechanical implementation by weaker agents with fewer footguns than Rust (ownership/lifetimes) and a real type system + single-binary story that TypeScript lacks; Node dependency trees are a long-horizon liability for a tool that must run in 10+ years.

## Consequences

- CLI uses stdlib `flag` + a small router (no cobra) per dependency policy DESIGN §10.6.
- Memory zeroization is best-effort only (GC language) — accepted; documented in DESIGN §5.3; owner machine is a trusted boundary.
- All issues specify Go module paths, package names, and function signatures so agents don't improvise structure.
