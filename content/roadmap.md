---
title: Roadmap
type: index
---

# Roadmap

Foundation-first phasing. Each phase is gated on the previous one shipping (`status: shipped` on every PRD in the phase).

> See [[architecture#10-phasing-dependency-graph]] for the dependency graph.

## P0 — Foundation (no user-visible features; pure refactor + schema) ✅ SHIPPED

Goal: make every later phase *additive* by introducing the seams now.

| Order | PRD | Why it's first | Status |
| --- | --- | --- | --- |
| 1 | [[prds/00-foundation/01-backend-abstraction]] | Defines the `VpnEngine` interface that MikroTik, BoringTun, AmneziaWG plug into. Without this, every later PRD branches the codebase. | ✅ |
| 2 | [[prds/00-foundation/04-data-model-migration]] | One migration that adds `engine_type`, `router`, `quota_*`, `speed_limit_*`, `expires_at`, `audit_log`. Everything below depends on it. | ✅ |
| 3 | [[prds/00-foundation/03-auth-refactor]] | Splits admin auth from user-dashboard auth, introduces API tokens. Required before user-dashboard and multi-admin. | ✅ |
| 4 | [[prds/00-foundation/02-multi-admin-rbac]] | Roles + permissions + audit log. Required before exposing per-user dashboards or per-router scoping. | ✅ |

## P1 — Flagship features (the reason this fork exists) ✅ SHIPPED

Goal: prove the engine abstraction with MikroTik, ship the user-visible features that differentiate the fork.

| Order | PRD | Notes | Status |
| --- | --- | --- | --- |
| 1 | [[prds/10-mikrotik/01-mikrotik-driver]] | First non-Linux engine. Validates the [[prds/00-foundation/01-backend-abstraction\|VpnEngine]] interface. | ✅ |
| 2 | [[prds/10-mikrotik/02-mikrotik-autoconfig]] | Bootstrap a fresh RouterOS device end-to-end. | ✅ |
| 3 | [[prds/20-user-features/01-user-dashboard]] | End-user view: status, expiry, usage graph. | ✅ |
| 4 | [[prds/20-user-features/02-qr-key-login]] | Login by scanning the WireGuard QR or pasting the config — no password. | ✅ |
| 5 | [[prds/20-user-features/03-bandwidth-quotas]] | Daily / weekly / monthly volume caps, auto-disable, auto-reset. | ✅ |
| 6 | [[prds/20-user-features/04-speed-limits]] | Per-client KB/s up/down rate caps. Engine-side: tc/HTB on Linux, queue tree on MikroTik. | ✅ |

## P2 — Multi-engine & federation (In Progress)

Goal: prove the abstraction with two more engines, scale to multi-router.

| Order | PRD | Status |
| --- | --- | --- |
| 1 | [[prds/30-multi-engine/01-amneziawg-promotion]] | ✅ |
| 2 | [[prds/30-multi-engine/02-boringtun-driver]] | ✅ |
| 3 | [[prds/30-multi-engine/03-engine-selection-ux]] | ✅ |
| 4 | [[prds/10-mikrotik/03-mikrotik-obfuscation]] | ✅ |
| 5 | [[prds/40-multi-server/01-multi-router-federation]] | 🌑 |
| 6 | [[prds/40-multi-server/02-admin-router-acl]] | 🌑 |

## P3 — Long tail

| PRD | Notes |
| --- | --- |
| [[prds/40-multi-server/03-multi-path-routing]] | Exit-node selection per IP/subnet/client. |
| [[prds/50-integrations/01-tailscale]] | Tailscale interop. |
| [[prds/50-integrations/02-sso]] | OIDC/SAML. Research-leaning. |

## Decided out of scope

- **Rust rewrite** — see [[decisions/0001-rust-rewrite]]. Recommendation: no. Bottleneck is integration breadth, not language perf.

## Cadence

- One P0 PRD at a time, sequential. P0 has high coupling — parallelism would just create merge conflicts.
- P1 can parallelize **after MikroTik driver ships**: `mikrotik-autoconfig` and `user-dashboard` can run in parallel Kimi sessions because they touch disjoint files.
- P2+ assumes the foundations are stable; multiple PRDs per phase can run concurrently.

## Definition of "phase done"

A phase is done when:

1. Every PRD in the phase has `status: shipped`.
2. Each PRD's `touches:` frontmatter has been verified against the actual diff (manual or scripted).
3. The integration test suite for the phase passes — see each PRD's verification section.
4. [[architecture]] is updated if any diagram drifted.
