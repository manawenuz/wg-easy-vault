---
id: PRD-20-04
title: Speed limits — per-client KB/s up/down
status: approved
phase: P1
depends_on:
  - "[[prds/00-foundation/01-backend-abstraction]]"
  - "[[prds/00-foundation/04-data-model-migration]]"
touches:
  - src/server/engines/wireguard/speedlimit.ts (new)
  - src/server/engines/wireguard/index.ts
  - src/server/engines/mikrotik/speedlimit.ts
  - src/server/services/speedLimitService.ts (new)
  - src/server/api/admin/clients/[id]/speed-limit.put.ts (new)
  - src/server/api/admin/clients/[id]/speed-limit.delete.ts (new)
  - src/app/components/Clients/SpeedLimitForm.vue (new)
  - src/app/pages/admin/clients/[id].vue
---

# PRD-20-04 — Per-client speed limits

## Why

Quotas cap volume; speed limits cap rate. A speed limit lets operators offer tiered service (e.g., a "free" tier capped at 1 MB/s) without burning through quotas. It's also the right hammer for fairness on shared exit links. **This is feature #14 from the original brief.**

## User stories

- As an **admin**, I can set up/down KB/s caps on a client.
- As an **admin**, I can clear the limit and the client returns to unrestricted.
- As a **user**, my dashboard shows the active limit (read-only).
- As an **admin** on MikroTik, the cap is enforced at the engine (queue tree). On Linux WG, the cap is enforced via `tc` HTB.

## Scope

### In

- One `speed_limit` row per client (table from [[prds/00-foundation/04-data-model-migration]]) with `up_kbps` and `down_kbps`. `0 = unlimited`.
- Engine-side enforcement:
  - **WireGuard / AmneziaWG / BoringTun (Linux)**: `tc qdisc add dev <iface> root handle 1: htb` once per interface; per-client classes + filters keyed on the client's IP.
  - **MikroTik**: queue tree (already specified in [[prds/10-mikrotik/01-mikrotik-driver|MikroTik driver]]).
- Admin form on client edit page.
- Read-only badge on user dashboard.
- Capability flag honored: if `engine.capabilities.speedLimit === 'none'` (none planned, but future-proof), the form is disabled with a tooltip.

### Out

- Time-of-day rate adjustments. Constant rate only.
- Burst configuration / per-class burst tuning. We use sensible defaults (`tc burst = 1.5kb`, MikroTik `burst-limit` = 1.5 × `max-limit`).
- Aggregate (per-tenant) speed limits. Client-level only.

## Data model changes

`speed_limit` table from [[prds/00-foundation/04-data-model-migration]].

## API changes

| Method | Path | Permission | Body | Returns |
| --- | --- | --- | --- | --- |
| PUT | `/api/admin/clients/[id]/speed-limit` | `client:write` | `{ upKbps, downKbps }` | speedLimit |
| DELETE | `/api/admin/clients/[id]/speed-limit` | `client:write` | — | `{ ok }` |

## UI changes

- `Clients/SpeedLimitForm.vue` — two number inputs with KB/s ↔ MB/s helper text; "Unlimited" checkbox per direction.
- Dashboard badge: small chip "↓ 1024 KB/s · ↑ 512 KB/s" on the client card.

## Driver / backend changes

### `VpnEngine.applySpeedLimit` / `clearSpeedLimit`

Already in the interface from [[prds/00-foundation/01-backend-abstraction]]. This PRD provides real implementations for two engines.

### WireGuard (Linux) — tc HTB

Once per interface:
```bash
tc qdisc add dev <iface> root handle 1: htb default 9999
tc class add dev <iface> parent 1: classid 1:1 htb rate 10gbit
tc qdisc add dev <iface> ingress           # for download (ingress shaping via ifb)
# Plus an ifb device for ingress shaping; create on first use.
```

Per-client (where `<id>` = client.id, `<ip>` = client.ipv4_address):
```bash
# Egress (from server towards client) = "download" from client perspective
tc class add dev <iface> parent 1:1 classid 1:<id> htb rate <downKbps>kbit ceil <downKbps>kbit
tc filter add dev <iface> parent 1: protocol ip u32 match ip dst <ip> flowid 1:<id>

# Ingress (from client towards server) = "upload" — needs ifb redirection
tc filter add dev <iface> parent ffff: protocol ip u32 match ip src <ip>
  action mirred egress redirect dev ifb-<iface>
tc class add dev ifb-<iface> parent 1:1 classid 1:<id> htb rate <upKbps>kbit ceil <upKbps>kbit
tc filter add dev ifb-<iface> parent 1: protocol ip u32 match ip src <ip> flowid 1:<id>
```

Module: `src/server/engines/wireguard/speedlimit.ts`. Idempotent: `clearSpeedLimit` does `tc class del classid 1:<id> 2>/dev/null` and friends, swallowing "no such file" errors.

The tc setup commands run as root inside the wg-easy container (which already has `NET_ADMIN`).

### MikroTik

Already specified in [[prds/10-mikrotik/01-mikrotik-driver]] §applySpeedLimit. This PRD adds the service-layer plumbing that calls it.

### Service-layer

```ts
// src/server/services/speedLimitService.ts
export async function setSpeedLimit(clientId, upKbps, downKbps) {
  const client = await db.client.find(clientId);
  const iface = await db.wgInterface.find(client.interfaceId);
  const engine = getEngine(iface.engineType);
  if (engine.capabilities.speedLimit === 'none') throw new Error('not supported');
  await db.speedLimit.upsert({ clientId, upKbps, downKbps });
  await engine.applySpeedLimit(iface, client.publicKey, upKbps, downKbps);
  await audit.logAction(event, 'client.speedLimit.set', { clientId, upKbps, downKbps });
}
```

Re-apply on:
- Engine restart (interface bring-up): iterate `speed_limit` rows for the interface, call `applySpeedLimit` on each.
- Client IP change: clear old, apply new.

## Migration & rollout

- Opt-in per client.
- On first use, the WG engine creates the qdisc/ifb infrastructure on the interface; verify the container has `NET_ADMIN` and the `ifb` kernel module is loadable. Document this in the upgrade notes.

## Verification

### Unit tests

- `wireguard/speedlimit.test.ts` — given (iface, ip, up, down), produces the expected tc command sequence (mock LocalShellTransport).
- `mikrotik/speedlimit.test.ts` — RouterOS API command sequence matches.
- `speedLimitService.test.ts` — capability check, audit, IP change re-applies.

### Integration test

- Linux: bring up wg interface, set limit 1000 KB/s down on a peer, run iperf3 from peer → server, observe ~1000 KB/s.
- MikroTik: same idea against CHR.

### Manual test plan

1. Set limit 512 KB/s up, 1024 KB/s down on a client.
2. From the client, run a speed test or `iperf3 -c <server>` (download) and `iperf3 -c <server> -R` (upload).
3. Observe rates capped within ±5%.
4. Clear limit; rerun; uncapped.

## Open questions

- [ ] ifb module availability: container needs `modprobe ifb`. If host kernel doesn't have it, ingress shaping fails. Decision: log a warning at startup if `ifb` not loadable; egress (download) still works; document the fallback.

---

## Kimi handoff

**Read before implementing:**
- `[[architecture]]` §3
- `[[prds/00-foundation/01-backend-abstraction]]`
- `[[prds/10-mikrotik/01-mikrotik-driver]]` (queue tree commands)
- `src/server/engines/wireguard/index.ts`
- `src/server/engines/mikrotik/index.ts`
- Linux `tc` HTB docs

**Modify these files:** see `touches:` frontmatter.

**Acceptance tests:**
1. Linux: iperf3 between client and server respects the cap within 5%.
2. MikroTik: queue tree applied; iperf3 respects.
3. Clearing limit removes all tc / queue-tree entries (no orphans).
4. Re-applies on interface restart.

**Self-test plan:**
```bash
pnpm test src/server/engines/wireguard/speedlimit.test.ts
pnpm test src/server/engines/mikrotik/speedlimit.test.ts
pnpm dev
# manual: iperf3
```
