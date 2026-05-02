---
id: PRD-30-01
title: Promote AmneziaWG from build flag to runtime engine
status: shipped
phase: P2
depends_on:
  - "[[prds/00-foundation/01-backend-abstraction]]"
touches:
  - src/server/engines/amneziawg/index.ts (new)
  - src/server/engines/amneziawg/configgen.ts (new)
  - src/server/engines/registry.ts
  - src/server/engines/wireguard/index.ts
  - Dockerfile
  - src/app/components/Interfaces/EngineSelector.vue (new)
  - src/app/pages/admin/interface.vue
  - src/server/engines/wg-like.ts (new)
  - src/server/database/repositories/interface/schema.ts
  - src/server/database/repositories/interface/types.ts
  - src/server/database/sqlite.ts
  - src/server/api/information.get.ts
  - src/i18n/locales/en.json
---

# PRD-30-01 — Promote AmneziaWG to a runtime engine

> ADR: [[decisions/0004-obfuscation-strategy]]

## Why

Today AmneziaWG is selected at build time via `WG_EXECUTABLE=awg` — the whole Docker image runs either WG or AWG, never both. Operators who want WG for some clients and AWG-obfuscated for others can't. This PRD makes engine selection per-interface at runtime.

## User stories

- As an **admin**, I can create one interface running native WireGuard and another running AmneziaWG on the same instance.
- As an **admin**, I can flip an interface's engine type (with a brief downtime + peer config regeneration).
- As a **user** on an AWG interface, my downloaded config has the AWG obfuscation parameters baked in.

## Scope

### In

- `AmneziaWgEngine implements VpnEngine` as a sibling of `WireguardEngine`. Capability flag `obfuscation: 'amneziawg-params'`.
- Both `wg-tools` and `amneziawg-tools` shipped in the Docker image (already true upstream).
- Per-interface engine selection in the admin UI.
- Config generation honors the AmneziaWG params already present on `wg_interface` and `client` (jC, jMin, jMax, s1-s4, h1-h4, i1-i5).
- Migration: existing interfaces have `engine_type='wireguard'`; if `WG_EXECUTABLE=awg` was used pre-fork, a one-time data fix flips them to `engine_type='amneziawg'` (detected by the env var presence at startup).

### Out

- Distinct AWG params per peer (already supported by schema; UI exposure is a follow-up).
- Migration tooling that converts an AWG client config into a WG client config (lossy; out of scope).

## Data model changes

None. `engine_type` and AmneziaWG fields all exist.

## API changes

Existing interface POST/PATCH endpoints accept `engineType: 'wireguard' | 'amneziawg'`. UI sends it; server validates against installed tools at runtime (graceful 400 if `awg` not on PATH).

## UI changes

- Interface create/edit form gains an **Engine** dropdown.
- For AmneziaWG interfaces, an "Obfuscation parameters" collapsible exposes jC/jMin/jMax/s1-s4/h1-h4/i1-i5 with reasonable defaults (the values upstream uses).

## Driver / backend changes

`AmneziaWgEngine` is largely `WireguardEngine` with `wg`→`awg` and `wg-quick`→`awg-quick` substitutions, plus AWG-specific config generation. Implementation: extract a small `WgLikeEngine` helper module shared between `WireguardEngine` and `AmneziaWgEngine` for the parts that *are* identical (peer parsing, `dump` parsing). **Keep it functional, not a parent class** — see [[decisions/0002-backend-abstraction]].

## Verification

- Unit tests: AWG config generator emits the obfuscation block.
- Integration: create an AWG interface; connect using an AWG client (Android `Amnezia VPN` app or `awg-tools` on Linux); handshake completes.
- Manual: also verify a WG-only client *cannot* connect to an AWG interface (sanity).

---

## Kimi handoff

**Read before implementing:**
- `[[architecture]]` §3
- `[[decisions/0004-obfuscation-strategy]]`
- `[[prds/00-foundation/01-backend-abstraction]]`
- `src/server/engines/wireguard/index.ts`, `configgen.ts`
- AmneziaWG schema fields in `repositories/interface/schema.ts` and `repositories/client/schema.ts`
- AmneziaWG tools/proto: https://github.com/amnezia-vpn/amneziawg-go

**Modify these files:** see `touches:` frontmatter.

**Acceptance tests:**
1. AWG interface created, AWG client connects, WG client cannot.
2. WG interface still works (no regression).
3. Migration from `WG_EXECUTABLE=awg` instances flips `engine_type` correctly on first boot.

**Self-test plan:**
```bash
pnpm test src/server/engines/amneziawg
pnpm dev
# manual: stand up AWG client (e.g. `awg-quick up`)
```

## Resolution log (2026-05-02)

- **Shipped**: `AmneziaWgEngine` as a standalone engine implementation.
- **Engine Selection**: Added `EngineSelector` component and integrated it into the interface settings page.
- **Code Reuse**: Extracted `wg-like.ts` functional helpers for shared parsing logic between WG and AWG engines.
- **Migration**: Added `migrateAwgEngineType()` to `sqlite.ts` for smooth transitions from legacy `WG_EXECUTABLE=awg` deployments.
- **API Coverage**: Updated all 25+ API and scheduler files to dynamically resolve the correct engine for each interface.
- **Tests**: 12 new unit tests for the AWG engine and config generator. Total suite: 149 tests pass.
