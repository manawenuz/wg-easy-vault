---
id: PRD-40-03
title: Multi-path routing — exit-node selection per IP / subnet / client
status: draft
phase: P3
depends_on:
  - "[[prds/40-multi-server/01-multi-router-federation]]"
touches:
  - src/server/services/routePolicyService.ts (new)
  - src/server/engines/wireguard/routePolicy.ts (new)
  - src/server/engines/mikrotik/routePolicy.ts (new)
  - src/server/api/admin/interface/[id]/route-policies.get.ts (new)
  - src/server/api/admin/interface/[id]/route-policies.put.ts (new)
  - src/server/api/admin/exit-nodes.get.ts (new)
  - src/server/api/admin/exit-nodes.post.ts (new)
  - src/app/components/RoutePolicies/Editor.vue (new)
  - src/app/pages/admin/exit-nodes.vue (new)
---

# PRD-40-03 — Multi-path routing

> Spec ref: [[architecture#9-multi-path-routing]]

## Why

With federation in place, we have multiple data planes. The natural next ask: **route different traffic out of different exit nodes**. Use cases: route streaming traffic out of a US exit, business traffic out of an EU exit; route certain IPs out of a known-clean ISP to avoid block lists; per-client policy based on plan tier.

## User stories

- As an **admin**, I can declare exit nodes (each backed by a router/interface) and write policies: "for client X, route 1.1.1.0/24 via exit A, 8.8.8.0/24 via exit B, default via exit C."
- As an **admin**, my policy edits propagate to the relevant data planes within a few seconds.
- As a **user**, I don't see the complexity — my client config still points at one interface; the orchestrator sets up the routing inside.

## Scope

### In

- `exit_node` and `route_policy` tables already in [[prds/00-foundation/04-data-model-migration]].
- Policy editor: per-interface, a table of `(client?, match_cidr, exit_node, priority)` rows.
- Engine support:
  - **Linux engines** (WG / AWG / BoringTun): `ip rule` + `ip route` with policy tables. Per policy: a custom routing table that next-hops via the chosen exit's tunnel; an `ip rule` matching `from <client_ip> to <match_cidr>` looking up that table.
  - **MikroTik**: `/ip/route/rule` and `/ip/route` with routing-mark; mangle rules to set the mark.
- Orchestrator pushes policies to the right router on every change; agents apply.

### Out

- BGP / dynamic routing protocols. Static policy only.
- Health-checked failover between exits. Manual priority for v1.
- Per-application routing (would require DPI — out).

## API changes

| Method | Path | Permission | Notes |
| --- | --- | --- | --- |
| GET | `/api/admin/exit-nodes` | `router:read` | list |
| POST | `/api/admin/exit-nodes` | `router:admin` | create from a router |
| GET | `/api/admin/interface/[id]/route-policies` | `router:read` | list |
| PUT | `/api/admin/interface/[id]/route-policies` | `router:write` | replace set |

## Driver / backend changes

`VpnEngine` gains two methods (additive, capability-flagged):

```ts
applyRoutePolicies(iface, policies: RoutePolicy[]): Promise<void>;
clearRoutePolicies(iface): Promise<void>;
```

Capability: `routing: 'none' | 'policy-routing'`. MikroTik and Linux engines = `policy-routing`. BoringTun = same (it's a tun device, same `ip rule` rules apply).

## Verification

- Unit tests per engine: policy → expected command sequence.
- Integration: two-exit federation; policy "send 1.1.1.1 via exit B"; from a connected client, `curl -v https://1.1.1.1` shows source IP = exit B; `curl -v https://example.com` shows source IP = exit A (default).

---

## Kimi handoff

**Read before implementing:**
- `[[architecture]]` §9
- `[[prds/40-multi-server/01-multi-router-federation]]`
- Linux policy routing docs (`ip rule`, `ip route table`)
- RouterOS routing-mark + mangle docs

**Acceptance tests:**
1. Two-exit integration test passes.
2. Removing all policies cleans up `ip rule` / RouterOS routing-mark rules — no orphans.
3. UI editor saves and reloads correctly.

**Self-test plan:**
```bash
pnpm test src/server/services/routePolicyService.test.ts
docker compose -f docker-compose.federation.yml up
# manual: curl with policy on/off, observe source IPs
```
