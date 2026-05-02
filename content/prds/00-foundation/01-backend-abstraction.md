---
id: PRD-00-01
title: VpnEngine driver interface
status: shipped
phase: P0
depends_on: []
touches:
  # Sources to delete / port
  - src/server/utils/WireGuard.ts
  - src/server/utils/wgHelper.ts
  - src/server/utils/cmd.ts
  # New engine layer
  - src/server/engines/types.ts (new)
  - src/server/engines/registry.ts (new)
  - src/server/engines/wireguard/index.ts (new)
  - src/server/engines/wireguard/configgen.ts (new)
  - src/server/transports/local-shell.ts (new)
  # Test
  - src/server/engines/wireguard/index.test.ts (new)
  # Schema (the existing dir is `interface/`, not `wgInterface/`)
  - src/server/database/repositories/interface/schema.ts
  # Callers — refactor in place (see "Discovered scope" below)
  - src/server/utils/Database.ts
  - src/server/plugins/manager.ts
  - src/server/api/admin/interface/index.post.ts
  - src/server/api/admin/interface/cidr.post.ts
  - src/server/api/admin/interface/restart.post.ts
  - src/server/api/admin/userconfig.post.ts
  - src/server/api/admin/hooks.post.ts
  - src/server/api/client/index.get.ts
  - src/server/api/client/index.post.ts
  - src/server/api/client/[clientId]/index.get.ts
  - src/server/api/client/[clientId]/index.post.ts
  - src/server/api/client/[clientId]/index.delete.ts
  - src/server/api/client/[clientId]/enable.post.ts
  - src/server/api/client/[clientId]/disable.post.ts
  - src/server/api/client/[clientId]/configuration.get.ts
  - src/server/api/client/[clientId]/qrcode.svg.get.ts
  - src/server/routes/cnf/[oneTimeLink].ts
  - src/server/routes/metrics/json.get.ts
  - src/server/routes/metrics/prometheus.get.ts
---

# PRD-00-01 — VpnEngine driver interface

> Status: `approved` · Phase: `P0` · Depends on: —
> Spec ref: [[architecture#3-vpnengine-driver-interface]] · ADR: [[decisions/0002-backend-abstraction]]

## Resolution log (2026-05-02)

Three ambiguities were raised on the first Kimi handoff and resolved by amending this PRD before Kimi proceeds:

1. **`api/admin/interface/index.ts` / `api/client/index.ts` don't exist as single files.** The route handlers are split per HTTP verb (`index.get.ts`, `index.post.ts`, etc.). The `touches:` list now enumerates each real file. **Do not** create `index.ts` aggregators.
2. **Callers of `WireGuard.*` exist beyond the original list.** `touches:` now includes every file the grep below returns. The "Discovered scope" rule below is the authoritative override of hard-rule #3 for *this PRD only*.
3. **Schema directory is `interface/`, not `wgInterface/`.** Path corrected throughout. The Drizzle table identifier `wgInterface` (camelCase TS export) is unchanged — that's correct as written.

## Discovered scope (overrides hard-rule #3 for this PRD)

This PRD is a refactor whose call-site set is discovered, not enumerated. The authoritative caller list is:

```bash
grep -rl 'WireGuard\.' src/server/
```

**Every file that grep returns is in scope** for this PRD, even if not pre-listed in `touches:`. If the grep returns a file not in `touches:`, **add it to your diff and proceed** — do not stop. The acceptance test `grep -rl 'WireGuard\.' src/server/` returning zero hits is the contract; the `touches:` list is a best-effort snapshot of what that grep returned at PRD authoring time.

This override applies **only to this PRD**. All later PRDs follow the strict-allowlist rule.

## Why

Every later PRD assumes pluggable VPN engines. Today the codebase has **one** engine (kernel WireGuard, with AmneziaWG as a build-time flag). Without a clean interface seam, MikroTik / BoringTun / multi-engine all become switch statements scattered across the codebase. This PRD introduces the seam **without adding any new engine** — pure refactor. Risk-controlled foundation work.

## User stories

- As an **engineer**, I can add a new VPN engine by dropping a folder under `src/server/engines/<name>/` and registering it, without touching any HTTP handler or service caller.
- As an **engineer**, I can write a unit test against a mock `VpnEngine` without standing up real WireGuard.
- As a **product owner**, I can say "engine = wireguard" and have a single place that decides what code runs.

## Scope

### In

- Define `VpnEngine` interface and `EngineType`/`EngineCapabilities` types.
- Implement `WireguardEngine` by relocating logic from `WireGuard.ts` and `wgHelper.ts`.
- Introduce a `LocalShellTransport` wrapping `cmd.ts:exec`.
- Introduce an engine registry (`getEngine(type)`).
- Refactor service/route callers to go through the registry.
- Preserve AmneziaWG behavior: when `WG_EXECUTABLE=awg`, the `WireguardEngine` constructs with the `awg` binary set. AmneziaWG is **not yet** a separate engine — that's [[prds/30-multi-engine/01-amneziawg-promotion]].

### Out

- Adding any new engine. (Each new engine is its own PRD.)
- Adding `engine_type` column to `wg_interface`. That lands in [[prds/00-foundation/04-data-model-migration]] — for now everything implicitly resolves to `wireguard`.
- Changing API or UI shape. This is invisible to users.
- Removing the legacy `WireGuard` class instantiation in `app.ts` until the registry is wired up; do both atomically.

## Data model changes

None in this PRD. `engine_type` is added in [[prds/00-foundation/04-data-model-migration]]. Until then, `getEngine(...)` defaults to `'wireguard'`.

## API changes

None observable. Internally, handlers replace direct `WireGuard.<method>` calls with `getEngine(iface.engineType ?? 'wireguard').<method>(iface, ...)`.

## UI changes

None.

## Driver / backend changes

### New types

```ts
// src/server/engines/types.ts
export type EngineType = 'wireguard' | 'amneziawg' | 'boringtun' | 'mikrotik';

export interface EngineCapabilities {
  obfuscation: 'none' | 'amneziawg-params' | 'wg-obfuscator-sidecar';
  speedLimit: 'none' | 'engine-native' | 'control-plane-fallback';
  multiPeerSync: boolean;       // can sync N peers in one call
  livePeerStats: boolean;       // sampleUsage returns counters in real time
}

export interface UsageSample {
  publicKey: string;
  rxBytes: bigint;
  txBytes: bigint;
  lastHandshakeAt: Date | null;
}

export interface Health {
  ok: boolean;
  details?: string;
}

export interface VpnEngine {
  readonly id: EngineType;
  readonly capabilities: EngineCapabilities;

  healthCheck(iface: WgInterface): Promise<Health>;

  bringUp(iface: WgInterface): Promise<void>;
  bringDown(iface: WgInterface): Promise<void>;

  // Sync: idempotent — called after any peer change. Implementations decide
  // whether to do diff-based or full reload.
  syncInterface(iface: WgInterface, peers: Client[]): Promise<void>;

  createPeer(iface: WgInterface, peer: Client): Promise<void>;
  updatePeer(iface: WgInterface, peer: Client): Promise<void>;
  removePeer(iface: WgInterface, peerPublicKey: string): Promise<void>;
  enablePeer(iface: WgInterface, peerPublicKey: string): Promise<void>;
  disablePeer(iface: WgInterface, peerPublicKey: string): Promise<void>;

  // Usage poll. Returns one row per peer.
  sampleUsage(iface: WgInterface): Promise<UsageSample[]>;

  // Speed limit. Engine returns NotSupportedError if capabilities.speedLimit === 'none'.
  applySpeedLimit(iface: WgInterface, peerPublicKey: string, upKbps: number, downKbps: number): Promise<void>;
  clearSpeedLimit(iface: WgInterface, peerPublicKey: string): Promise<void>;
}
```

### New transport

```ts
// src/server/transports/local-shell.ts
export class LocalShellTransport {
  async exec(cmd: string): Promise<{ stdout: string; stderr: string }>;
}
```

Wraps existing `cmd.ts:exec`. Future SSH and RouterOS API transports implement the same shape (where applicable).

### Engine registry

```ts
// src/server/engines/registry.ts
const engines = new Map<EngineType, VpnEngine>();
engines.set('wireguard', new WireguardEngine(new LocalShellTransport()));
export function getEngine(type: EngineType): VpnEngine { ... }
```

### WireguardEngine

Implements `VpnEngine` by porting:
- Lifecycle: `WireGuard.ts:Startup` → `bringUp`, `Shutdown` → `bringDown`.
- Sync: `WireGuard.ts:saveConfig` + `wgHelper.ts:sync` → `syncInterface`.
- Peer ops: read/write through the existing config-write + `wg syncconf` mechanism.
- `sampleUsage`: parse `wg show <iface> dump` output (existing `wgHelper.ts:dump`).
- `applySpeedLimit`: shells out to `tc qdisc` / `tc class` / `tc filter` (HTB). New code; small.

The legacy `WireGuard` class is **deleted**. All callers go through `getEngine('wireguard')`.

### Caller refactor

Find sites:
```bash
rg -n 'WireGuard\.' src/server
```

Each site becomes:
```ts
const engine = getEngine(iface.engineType ?? 'wireguard');
await engine.createPeer(iface, peer);
```

## Migration & rollout

- Single PR, single commit ideally. The refactor must be atomic to keep `master` working.
- No feature flag — there's no behavior change.
- `WG_EXECUTABLE` env continues to select `wg` vs `awg` inside `WireguardEngine`. (This collapses into the AmneziaWG engine in [[prds/30-multi-engine/01-amneziawg-promotion]].)

## Verification

### Unit tests

- `src/server/engines/wireguard/index.test.ts` — given a fake `LocalShellTransport`, assert that `createPeer` issues the expected `wg set` command, `sampleUsage` parses `wg show dump` correctly, `syncInterface` writes the expected config string.
- `src/server/engines/registry.test.ts` — `getEngine('wireguard')` returns a `WireguardEngine`; `getEngine('mikrotik')` throws (until P1 lands).

### Integration test

- The existing `docker compose up` flow still produces a working WireGuard interface, can create a peer via the UI, and the peer connects from a client.

### Manual test plan

1. `pnpm build && docker compose up`.
2. Open admin UI → create interface → add client → download config.
3. Connect from a real WireGuard client. Verify handshake.
4. Disable client → verify connection drops.
5. Re-enable → verify reconnect.

## Open questions

- [ ] Should `syncInterface` be the only mutation method (peer ops dispatched through it) or can engines do per-peer ops directly? **Tentative answer**: keep both for engines that support live updates (kernel WG via `wg set peer`), fall back to full sync for engines that don't (BoringTun reload).

---

## Kimi handoff

**Read before implementing:**
- `[[architecture]]` §2, §3
- `[[glossary]]`
- `[[decisions/0002-backend-abstraction]]`
- `src/server/utils/WireGuard.ts` (full file)
- `src/server/utils/wgHelper.ts` (full file)
- `src/server/utils/cmd.ts` (full file)
- `src/server/database/repositories/interface/schema.ts` (the existing schema you'll preserve)
- `src/server/database/repositories/client/schema.ts` (read for type shapes only)
- Find all callers: `grep -rl 'WireGuard\.' src/server/` — read each, all are in scope per "Discovered scope" above.

**Modify these files:**
- The full `touches:` frontmatter list is authoritative. Summary:
  - **Delete**: `src/server/utils/WireGuard.ts` (logic ported to engine).
  - **Modify**: `src/server/utils/wgHelper.ts` — extract config-gen functions into `src/server/engines/wireguard/configgen.ts`. If `wgHelper.ts` ends up empty after extraction, delete it; if other modules still import from it, keep the remaining exports.
  - **Modify**: every caller of `WireGuard.*` per the grep above. The caller list in `touches:` was a snapshot; if the grep on your tree returns additional files, add them and proceed.
  - **Modify**: `src/server/database/repositories/interface/schema.ts` only if a type/import touches this PRD's surface. No schema changes here — `engine_type` lands in [[prds/00-foundation/04-data-model-migration]]. (Listed in `touches:` because the type imports may need adjustment; do not add columns.)
  - **Create**: the five new files under `src/server/engines/` and `src/server/transports/` plus the test.

**Acceptance tests:**
1. `pnpm test src/server/engines` passes.
2. Existing test suite still passes (no regressions).
3. `docker compose up` produces a working WireGuard interface; UI smoke test passes.
4. `grep -rl 'WireGuard\.' src/server/` returns zero files (callers fully migrated). The only file allowed to contain the string `WireGuard.` is the deleted-file mention in commit messages — none in source.

**Self-test plan:**
```bash
pnpm install
pnpm typecheck
pnpm test
pnpm build
docker compose up -d
# wait for health
curl -fsSL http://localhost:51821/api/information
```
