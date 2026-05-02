---
id: ADR-0002
title: Driver/strategy pattern for VPN engines (not class inheritance)
status: decided
date: 2026-05-02
---

# ADR-0002 — Driver/strategy pattern for VPN engines

## Context

Today wg-easy has a single `WireGuard` class (`src/server/utils/WireGuard.ts:10-338`) with shell-out helpers in `wgHelper.ts`. We need to support multiple engines: native WireGuard, AmneziaWG, BoringTun, MikroTik. Two natural shapes:

- **Inheritance**: extract `WireGuard` into a base class, subclass per engine.
- **Strategy / driver pattern**: a narrow `VpnEngine` interface, concrete implementations live in `src/server/engines/<name>/`.

## Decision

**Strategy pattern with a narrow interface.** No inheritance. The interface is defined in [[architecture#3-vpnengine-driver-interface]].

## Reasoning

1. **MikroTik is not a subclass of WireGuard.** It doesn't shell out to `wg-quick`. It doesn't write `/etc/wireguard/*.conf`. It doesn't manage local kernel interfaces. Forcing it under a `WireGuardBase` would mean a base class that exists only to be overridden into uselessness. Fragile base class anti-pattern.

2. **The interface is small.** ~12 methods, all I/O. There's no shared algorithm to factor out — config generation differs per engine, peer ops differ per engine, lifecycle differs per engine. Any "shared" code would be ad-hoc utilities, which can live as plain functions.

3. **Capability flags > method overrides.** Some engines support obfuscation, some don't. Some support speed limits at the engine level, some need control-plane fallback. Capability flags on the interface (`supportsObfuscation: boolean`) keep this declarative; the UI gracefully degrades. Inheritance with `throw new NotSupportedError()` is brittle.

4. **Testability.** A driver is a class behind an interface — trivial to mock per test. An inheritance hierarchy entangles tests with parent class state.

5. **Future Rust agent.** Per [[decisions/0001-rust-rewrite|ADR-0001]], if we ever rewrite the agent in Rust, an interface translates cleanly to a Rust trait. An inheritance hierarchy doesn't.

## Shape

```ts
// src/server/engines/types.ts
export interface VpnEngine {
  readonly id: EngineType;
  readonly capabilities: EngineCapabilities;

  healthCheck(): Promise<Health>;
  syncInterface(iface: WgInterface, peers: Client[]): Promise<void>;
  createPeer(iface: WgInterface, peer: Client): Promise<void>;
  // ...
}

// src/server/engines/wireguard/index.ts
export class WireguardEngine implements VpnEngine { ... }

// src/server/engines/registry.ts
export function getEngine(type: EngineType): VpnEngine { ... }
```

## Consequences

- The existing `WireGuard` class is **renamed and moved** to `src/server/engines/wireguard/index.ts`, implementing `VpnEngine`. Most logic is preserved; the class loses its global-singleton role.
- `wgHelper.ts` config-generation functions move into the WireGuard / AmneziaWG engines as private methods. They are not shared with MikroTik or BoringTun.
- Service-layer code calls `getEngine(iface.engineType).createPeer(...)` — no `if (engine === 'wireguard')` branches in routes/services.
- New engines are added by dropping a folder under `src/server/engines/` and registering it. No changes to callers.

## Alternatives considered

- **Single class with a big switch on `engine_type`** — rejected. Becomes a god class quickly; hostile to the "third engine" addition.
- **Plugin loader (dynamic import)** — overkill. We have ~4 engines for the foreseeable future; static registry is simpler and tree-shakeable.
- **One class hierarchy per protocol family (e.g., `WireGuardLikeEngine` parent)** — rejected. AmneziaWG is the only obvious sibling of native WireGuard, and even there the obfuscation params already live as fields, not behavior. Not enough shared code to justify a parent.

## Migration

Done as the first PRD: [[prds/00-foundation/01-backend-abstraction]]. The PR introducing this pattern only refactors the existing `WireGuard` class into a `WireguardEngine` and adds the registry. No new engines yet — those come after.
