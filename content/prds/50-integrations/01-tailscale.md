---
id: PRD-50-01
title: Tailscale integration
status: draft
phase: P3
depends_on:
  - "[[prds/00-foundation/01-backend-abstraction]]"
touches:
  - src/server/integrations/tailscale/index.ts (new)
  - src/server/integrations/tailscale/api.ts (new)
  - src/server/api/admin/integrations/tailscale.get.ts (new)
  - src/server/api/admin/integrations/tailscale.put.ts (new)
  - src/app/pages/admin/integrations/tailscale.vue (new)
---

# PRD-50-01 — Tailscale integration

## Why

Many users have an existing Tailscale tailnet for east-west traffic and want wg-easy as the **public-facing VPN** that bridges into the tailnet. Or they want wg-easy peers to appear as Tailscale nodes for ACL purposes. Either way, integration is valuable but **not architecturally central** — it's an integration layer, not a new engine.

## User stories

- As an **admin**, I can register wg-easy as an OAuth client in Tailscale and let it manage a "wg-easy" namespace inside the tailnet.
- As an **admin**, when I create a wg-easy client, the system can optionally provision a corresponding Tailscale auth-key and associate the two.
- As a **user** with a wg-easy client, my traffic to tailnet IPs is routed via the Tailscale exit node that wg-easy controls (subnet routing in Tailscale terms).

## Scope

### In (research-leaning, document only initially)

- Two integration modes:
  1. **Subnet router**: wg-easy advertises `10.8.0.0/24` (its WG subnet) to the tailnet via a co-located `tailscaled`. Users on Tailscale reach wg-easy clients; wg-easy clients reach tailnet hosts (one-hop).
  2. **Auth-key issuance**: for each wg-easy client, also mint a Tailscale auth-key, install both configs in the user-dashboard download bundle.
- Admin page to configure: Tailscale OAuth client id/secret, which integration mode, default ACL tags.
- Health check + last-sync indicator.

### Out

- Replacing wg-easy's own VPN with Tailscale (different product).
- Tailscale ACL editing from wg-easy (use Tailscale admin UI).
- Tailscale SSH / Funnel / Serve features.

## Status

P3, **research-heavy**. The first deliverable for this PRD is a 1-page experiment report:
- What does the Tailscale OAuth API actually let us do today?
- What does subnet router provisioning require operationally (does it need root on the wg-easy host)?
- What's the user-side experience tradeoff vs. just pointing a Tailscale exit at the wg-easy box manually?

Only after that report is the implementation scope locked.

## Open questions

- [ ] Whether to bundle `tailscaled` in the wg-easy Docker image (size cost) or assume it runs alongside (operational cost).
- [ ] Whether the integration is per-router (each agent advertises its own subnet) or orchestrator-only.

---

## Kimi handoff

**Phase 1 (research):**
- Read Tailscale OAuth + subnet router docs.
- Produce a 1-page report at `docs/obsidian/research/tailscale-feasibility.md` covering the open questions.
- Do NOT write production code in this phase.

**Phase 2 (after report approved):**
- Re-open this PRD with concrete scope; then implement.
