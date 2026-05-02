---
id: PRD-10-03
title: MikroTik obfuscation via wg-obfuscator sidecar
status: draft
phase: P2
depends_on:
  - "[[prds/10-mikrotik/01-mikrotik-driver]]"
touches:
  - src/server/engines/mikrotik/obfuscator.ts (new)
  - src/server/engines/mikrotik/index.ts
  - src/server/database/repositories/wgObfuscatorConfig/schema.ts (new)
  - src/server/api/admin/interface/[id]/obfuscation.put.ts (new)
  - src/app/components/Interfaces/ObfuscationForm.vue (new)
---

# PRD-10-03 — MikroTik obfuscation via wg-obfuscator

> ADR: [[decisions/0004-obfuscation-strategy]] · Reference: [wg-obfuscator MikroTik docs](https://github.com/ClusterM/wg-obfuscator/blob/master/docs/MIKROTIK.md)

## Why

RouterOS doesn't run AmneziaWG. For MikroTik backends that need DPI evasion, the practical answer is `wg-obfuscator` — a small UDP proxy that wraps WireGuard packets. It runs on the MikroTik via the published integration. This PRD adds first-class support: enable a checkbox on the interface, the orchestrator deploys/configures the obfuscator, and client config download includes setup instructions for the user side.

## User stories

- As an **admin** managing a MikroTik interface, I toggle "Obfuscation: on" and the orchestrator configures the device end.
- As a **user** with an obfuscated interface, the dashboard tells me I need a small wg-obfuscator client locally; download includes instructions and parameters.
- As an **admin**, I can rotate the obfuscation key; clients re-download to receive the new key.

## Scope

### In

- New table `wg_obfuscator_config` (one row per interface, present only when obfuscation is enabled): `interface_id PK`, `listen_port`, `wg_target_port`, `key`, `dummy_padding_min`, `dummy_padding_max`.
- Server-side automation: SSH to the MikroTik, install the wg-obfuscator container/script per the upstream docs, configure it to listen on `listen_port` and forward to the local WG `wg_target_port`.
- Adjusted client config: `Endpoint` points at `<router>:<listen_port>` (the obfuscator), and the download includes a wg-obfuscator client config snippet plus a link/instructions.
- UI: per-interface "Obfuscation" toggle for MikroTik interfaces, with the parameters editable in an advanced section.

### Out

- AmneziaWG-style obfuscation on MikroTik (RouterOS doesn't support it).
- Obfuscation for non-MikroTik engines (use AmneziaWG instead — [[prds/30-multi-engine/01-amneziawg-promotion]]).
- Generic obfuscation framework — see [[decisions/0004-obfuscation-strategy]].

## Data model changes

```ts
export const wgObfuscatorConfig = sqliteTable('wg_obfuscator_config', {
  // wg_interface PK is name TEXT — FK must be text, not int
  interfaceId: text('interface_id').primaryKey().references(() => wgInterface.name, { onDelete: 'cascade', onUpdate: 'cascade' }),
  listenPort: integer('listen_port').notNull(),
  wgTargetPort: integer('wg_target_port').notNull(),
  key: text('key').notNull(),  // base64; treat as secret
  dummyPaddingMin: integer('dummy_padding_min').notNull().default(8),
  dummyPaddingMax: integer('dummy_padding_max').notNull().default(64),
});
```

This is a P2 schema add — small enough to ship as its own migration on top of the foundation migration.

## Verification

- Unit: deploy script generation matches the upstream docs.
- Integration: a real MikroTik with wg-obfuscator deployed; a wg-obfuscator client on a Linux box; handshake succeeds through the obfuscator and not without it.
- Negative test: with obfuscation on, a vanilla WireGuard client to the obfuscator port fails to handshake (proves the wrapping is in place).

---

## Kimi handoff

**Read before implementing:**
- `[[decisions/0004-obfuscation-strategy]]`
- `[[prds/10-mikrotik/01-mikrotik-driver]]`
- wg-obfuscator MikroTik integration docs (URL above)
- `src/server/engines/mikrotik/index.ts`
- `src/server/transports/ssh.ts`

**Acceptance tests:**
1. Toggling obfuscation on installs and configures the obfuscator on the MikroTik.
2. Download bundle includes user-side instructions + a wg-obfuscator client config snippet.
3. Toggling off cleanly removes the obfuscator install (idempotent).

**Self-test plan:**
```bash
pnpm test src/server/engines/mikrotik/obfuscator.test.ts
# manual: against a CHR + a Linux test client
```
