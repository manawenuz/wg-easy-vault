---
id: PRD-30-02
title: BoringTun engine — userspace WireGuard
status: shipped
phase: P2
depends_on:
  - "[[prds/00-foundation/01-backend-abstraction]]"
touches:
  - src/server/engines/boringtun/index.ts (new)
  - src/server/engines/boringtun/process.ts (new)
  - src/server/engines/boringtun/index.test.ts (new)
  - src/server/engines/boringtun/process.test.ts (new)
  - src/server/engines/registry.ts
  - src/server/engines/registry.test.ts
  - src/server/database/repositories/interface/types.ts
  - Dockerfile
---

# PRD-30-02 — BoringTun engine

## Why

Some hosts can't run the kernel WireGuard module (locked-down kernels, unprivileged containers, macOS in dev). BoringTun is a Cloudflare-built userspace WireGuard implementation in Rust. Adding it as an engine widens the deployment surface and double-validates the [[prds/00-foundation/01-backend-abstraction|VpnEngine]] interface against another wire-compatible-but-different-process engine.

## User stories

- As a **deployer**, I can run wg-easy on a host without the WG kernel module by selecting `engine_type=boringtun` on the interface.
- As an **engineer**, BoringTun behaves identically to WG from the outside (same UAPI, same wire protocol, peers don't know the difference).

## Scope

### In

- `BoringtunEngine implements VpnEngine`. Manages a `boringtun-cli` process per interface.
- Communicates via the [WireGuard cross-platform UAPI](https://www.wireguard.com/xplatform/) — same protocol kernel WG exposes, BoringTun also speaks it. The control plane uses a UAPI client, not shell-out to `wg`.
- Bundle `boringtun` binary in the Dockerfile (musl static build).
- Capability flags: `obfuscation: 'none'`, `speedLimit: 'engine-native'` (still tc-based since the iface is a tun device), `multiPeerSync: false`, `livePeerStats: true`.

### Out

- Anything that requires kernel-WG-only features (e.g., specific `wg-quick` PostUp scripts that depend on kernel timing). Document differences.
- Replacing the kernel WG engine. They coexist per-interface.

## Driver / backend changes

### Process management

```ts
// src/server/engines/boringtun/process.ts
class BoringtunProcessManager {
  start(iface): Promise<void>;     // spawns `boringtun-cli <iface>` detached
  stop(iface): Promise<void>;
  uapiSocket(iface): string;        // /var/run/wireguard/<iface>.sock
}
```

UAPI client over the unix socket — small protocol (text key=value lines). Implement directly; no library dependency.

### Sync / sample / peer ops

All via UAPI: `set`, `get`. Lifecycle: BoringTun process owns the tun; we send `set` over UAPI to update peers.

## Verification

- Unit: UAPI client unit-tested against captured fixtures.
- Integration: container with `boringtun` binary, no `wireguard` kernel module loaded; create iface, peer connects.
- Compare to kernel WG engine in CI: run the same test matrix against both, assert behavior parity.

---

## Kimi handoff

**Read before implementing:**
- `[[architecture]]` §3
- `[[prds/00-foundation/01-backend-abstraction]]`
- WireGuard UAPI: https://www.wireguard.com/xplatform/
- BoringTun: https://github.com/cloudflare/boringtun

**Modify these files:** see `touches:` frontmatter.

**Acceptance tests:**
1. BoringTun interface comes up; peer handshakes.
2. UAPI client correctly issues `set` and parses `get`.
3. Process restart on crash (supervisor).

**Self-test plan:**
```bash
pnpm test src/server/engines/boringtun
docker build -t wg-easy-test --build-arg ENGINE=boringtun .
docker run --rm --cap-add NET_ADMIN --device /dev/net/tun wg-easy-test
```

## Resolution log (2026-05-02)

- **Shipped**: `BoringtunEngine` and `BoringtunProcessManager` with UAPI client.
- **Process Supervision**: Spawns `boringtun-cli`, supervises crashes (max 3 restarts).
- **UAPI Protocol**: Implemented directly over Unix socket (set/get); unit-tested with fixtures.
- **Speed Limits**: Reuse existing `wireguard/speedlimit.ts` (tc-based) since iface is a tun device.
- **Firewall/Hooks**: Parasitic behavior matches kernel engines (iptables + manual hook execution).
- **Dockerfile**: Added `rust:alpine` multi-stage build stage for `boringtun-cli`.
- **Gap Fixed**: Updated `InterfaceUpdateSchema` in `types.ts` to allow `boringtun` and `mikrotik` engine types.
