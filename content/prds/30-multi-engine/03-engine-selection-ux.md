---
id: PRD-30-03
title: Engine selection UX
status: draft
phase: P2
depends_on:
  - "[[prds/30-multi-engine/01-amneziawg-promotion]]"
  - "[[prds/30-multi-engine/02-boringtun-driver]]"
touches:
  - src/app/components/Interfaces/EngineSelector.vue
  - src/app/components/Interfaces/EngineCapabilityHints.vue (new)
  - src/server/api/admin/engines.get.ts (new)
---

# PRD-30-03 — Engine selection UX

## Why

After the engine roster grows to 4 (wg / awg / boringtun / mikrotik), operators need a guided way to pick one. The form should explain tradeoffs inline (obfuscation availability, speed-limit support, where it runs).

## Scope

### In

- `GET /api/admin/engines` returns the capability matrix — engines actually installed/usable on this control plane, with capability flags + a one-line description each.
- `EngineSelector.vue` consumes the matrix and renders a card-style picker (not a dropdown) so capabilities are visible at decision time.
- Capability hints update live as the user toggles router selection (because some engines are router-specific: MikroTik only valid for `engine_type=mikrotik` routers; Linux engines only valid for the self router or other Linux agents).

### Out

- Cross-engine peer migration UI. (If you change an interface's engine, you regenerate configs; users redownload. We surface this as a confirmation modal, not an automated flow.)

---

## Kimi handoff

**Read before implementing:**
- `[[architecture]]` §3
- The three engine PRDs above

**Acceptance tests:**
1. The form disables MikroTik option when router is "self".
2. Changing engine on an existing interface shows a "this will require client config regeneration" confirm.

**Self-test plan:**
```bash
pnpm dev
# manual: create interfaces against each engine
```
