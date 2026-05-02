---
id: PRD-00-04
title: Data model migration — engines, routers, quotas, speed limits, audit
status: shipped
phase: P0
depends_on:
  - "[[prds/00-foundation/01-backend-abstraction]]"
touches:
  - src/server/database/repositories/interface/schema.ts
  - src/server/database/repositories/client/schema.ts
  - src/server/database/repositories/user/schema.ts
  - src/server/database/repositories/router/schema.ts (new)
  - src/server/database/repositories/quota/schema.ts (new)
  - src/server/database/repositories/speedLimit/schema.ts (new)
  - src/server/database/repositories/usageSample/schema.ts (new)
  - src/server/database/repositories/auditLog/schema.ts (new)
  - src/server/database/repositories/adminRouterAcl/schema.ts (new)
  - src/server/database/repositories/exitNode/schema.ts (new)
  - src/server/database/repositories/routePolicy/schema.ts (new)
  - src/server/database/repositories/apiToken/schema.ts (new)
  - src/server/database/migrations/<timestamp>_p0_foundation.sql (new)
  - src/server/database/sqlite.ts
---

# PRD-00-04 — Data model migration

> Spec ref: [[architecture#4-data-model]]

## Why

A single migration up front that adds **every column and table the roadmap will need**. Doing this in one shot avoids piecemeal migrations that ratchet the schema through inconsistent intermediate states, and it lets later PRDs be **schema-additive only at the type level** (no further migrations needed for the listed phases). All new tables are empty until the feature that owns them ships.

## User stories

- As an **engineer**, I can add the quota feature without writing a migration.
- As an **operator**, I run `pnpm migrate` once and the DB has every table the fork needs through P3.

## Scope

### In

- New tables: `router`, `quota`, `speed_limit`, `usage_sample`, `audit_log`, `admin_router_acl`, `exit_node`, `route_policy`, `api_token`.
- New columns on existing tables:
  - `wg_interface.engine_type`, `wg_interface.router_id` (FK to `router`, nullable initially; backfilled to local router id 0).
  - ~~`client.expires_at` (nullable)~~ — already exists in `repositories/client/schema.ts:32` as `text('expires_at')`. Do **not** re-add. If a typed timestamp is desired, that's a separate later migration.
  - `user.role` already exists; widen its accepted enum to `superadmin | admin | operator | viewer | client`.
- A "self" router row (id=0) inserted at migration time pointing at the local box (`engine_type='wireguard'`, `transport='local-shell'`).
- All existing `wg_interface` rows get `router_id = 0` and `engine_type = 'wireguard'` via backfill in the same migration.

### Out

- Wiring services / API to the new tables. That happens in the PRDs that own each table.
- Removing `engine_type` from `wg_interface` if it's redundant with `router.engine_type` — we keep it denormalized for query speed.
- Postgres support. SQLite only for now; Drizzle keeps options open.

## Data model changes

### New tables (Drizzle SQLite)

```ts
// router/schema.ts
export const router = sqliteTable('router', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  name: text('name').notNull().unique(),
  engineType: text('engine_type').$type<EngineType>().notNull(),
  transport: text('transport').$type<TransportType>().notNull(),
  host: text('host'),                  // null for local-shell
  port: integer('port'),
  credentialsEncrypted: text('credentials_encrypted'), // JSON, encrypted at rest
  enabled: integer('enabled', { mode: 'boolean' }).notNull().default(true),
  lastSeen: integer('last_seen', { mode: 'timestamp' }),
  createdAt: integer('created_at', { mode: 'timestamp' }).notNull().$defaultFn(() => new Date()),
  updatedAt: integer('updated_at', { mode: 'timestamp' }).notNull().$defaultFn(() => new Date()),
});
```

```ts
// quota/schema.ts — one row per client when quota is set
export const quota = sqliteTable('quota', {
  clientId: integer('client_id').primaryKey().references(() => client.id, { onDelete: 'cascade' }),
  limitBytes: integer('limit_bytes', { mode: 'number' }).notNull(),  // store as bigint where supported
  period: text('period').$type<'daily'|'weekly'|'monthly'>().notNull(),
  usedBytes: integer('used_bytes').notNull().default(0),
  periodStart: integer('period_start', { mode: 'timestamp' }).notNull(),
  periodEnd: integer('period_end', { mode: 'timestamp' }).notNull(),
  autoDisable: integer('auto_disable', { mode: 'boolean' }).notNull().default(true),
  // null = not currently disabled by quota; date = disabled at
  disabledByQuotaAt: integer('disabled_by_quota_at', { mode: 'timestamp' }),
});
```

```ts
// speedLimit/schema.ts — one row per client when speed limit is set
export const speedLimit = sqliteTable('speed_limit', {
  clientId: integer('client_id').primaryKey().references(() => client.id, { onDelete: 'cascade' }),
  upKbps: integer('up_kbps').notNull().default(0),    // 0 = unlimited
  downKbps: integer('down_kbps').notNull().default(0),
  appliedAt: integer('applied_at', { mode: 'timestamp' }),
});
```

```ts
// usageSample/schema.ts — high write volume
export const usageSample = sqliteTable('usage_sample', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  clientId: integer('client_id').references(() => client.id, { onDelete: 'cascade' }).notNull(),
  rxBytes: integer('rx_bytes').notNull(),
  txBytes: integer('tx_bytes').notNull(),
  ts: integer('ts', { mode: 'timestamp' }).notNull(),
}, (t) => ({
  clientTs: index('usage_sample_client_ts').on(t.clientId, t.ts),
}));
```

```ts
// auditLog/schema.ts
export const auditLog = sqliteTable('audit_log', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  actorUserId: integer('actor_user_id').references(() => user.id),
  action: text('action').notNull(),     // 'router.create', 'client.disable', etc.
  target: text('target'),                // JSON
  result: text('result').$type<'ok'|'error'>().notNull(),
  ts: integer('ts', { mode: 'timestamp' }).notNull().$defaultFn(() => new Date()),
});
```

```ts
// adminRouterAcl/schema.ts
export const adminRouterAcl = sqliteTable('admin_router_acl', {
  userId: integer('user_id').references(() => user.id, { onDelete: 'cascade' }).notNull(),
  routerId: integer('router_id').references(() => router.id, { onDelete: 'cascade' }).notNull(),
  permission: text('permission').$type<'read'|'write'|'admin'>().notNull(),
}, (t) => ({
  pk: primaryKey({ columns: [t.userId, t.routerId] }),
}));
```

```ts
// exitNode/schema.ts
export const exitNode = sqliteTable('exit_node', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  routerId: integer('router_id').references(() => router.id).notNull(),
  label: text('label').notNull(),
  enabled: integer('enabled', { mode: 'boolean' }).notNull().default(true),
});
```

```ts
// routePolicy/schema.ts
export const routePolicy = sqliteTable('route_policy', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  // wg_interface PK is name TEXT (see src/server/database/repositories/interface/schema.ts) — FK must be text, not int
  interfaceId: text('interface_id').references(() => wgInterface.name, { onDelete: 'cascade', onUpdate: 'cascade' }).notNull(),
  clientId: integer('client_id').references(() => client.id, { onDelete: 'cascade' }), // nullable
  matchCidr: text('match_cidr').notNull(),
  exitNodeId: integer('exit_node_id').references(() => exitNode.id).notNull(),
  priority: integer('priority').notNull().default(100),
});
```

```ts
// apiToken/schema.ts
export const apiToken = sqliteTable('api_token', {
  id: integer('id').primaryKey({ autoIncrement: true }),
  userId: integer('user_id').references(() => user.id, { onDelete: 'cascade' }).notNull(),
  tokenHash: text('token_hash').notNull(),     // argon2id
  label: text('label'),
  scopes: text('scopes'),                       // JSON array
  expiresAt: integer('expires_at', { mode: 'timestamp' }),
  lastUsedAt: integer('last_used_at', { mode: 'timestamp' }),
  createdAt: integer('created_at', { mode: 'timestamp' }).notNull().$defaultFn(() => new Date()),
});
```

### Modified tables

- `wg_interface`: add `engineType TEXT NOT NULL DEFAULT 'wireguard'`, `routerId INTEGER NOT NULL DEFAULT 0`.
- `client`: ~~add `expiresAt INTEGER`~~ — column already present as `text`. No-op for this PRD.

### Backfill (in migration)

- Insert one row in `router`: `(id=0, name='self', engine_type='wireguard', transport='local-shell', enabled=true)`.
- All existing `wg_interface` rows already get `router_id=0` and `engine_type='wireguard'` via the column defaults.

## API changes

None — this PRD only touches schema and the `DBService` aggregator.

## UI changes

None.

## Driver / backend changes

- `src/server/database/sqlite.ts` aggregates the 7 new repositories alongside the existing 7. Pattern is mechanical.
- New repositories follow the existing one-folder-per-table convention with `schema.ts`, `service.ts` (skeleton — full service logic lands in feature PRDs), and `types.ts`.

## Migration & rollout

- Single SQL migration file under `src/server/database/migrations/`. Drizzle auto-generates from the schema diff via `pnpm db:generate`.
- Test the migration on a copy of a running DB: data should be preserved, all existing peers continue to work.
- No down-migration is required for shipping (Drizzle supports it but we don't expect to roll back).

## Verification

### Unit tests

- `src/server/database/migrations/p0_foundation.test.ts` — apply migration to a fresh DB and to a populated DB; assert table existence, the self-router row, backfilled `router_id` / `engine_type`.

### Integration test

- `pnpm dev` against a copy of an existing wg-easy DB: app starts, peers visible, peers connectable. No 500s in logs.

### Manual test plan

1. Take a backup of `/etc/wireguard/wg-easy.db` from a running instance.
2. Start the new build pointing at the same DB file.
3. Verify all interfaces and peers visible in admin UI.
4. Verify a real client can still connect.
5. Inspect DB: `sqlite3 wg-easy.db '.schema router'` shows the new table; `SELECT * FROM router` shows the self row.

## Open questions

- [ ] SQLite `bigint` mode vs `number` for `limitBytes`/`usedBytes`. For monthly quotas at multi-TB scale, JS `number` is fine (2^53 bytes >> any plausible quota). Decision: use `number` mode; revisit if anyone hits 8 PB.

---

## Kimi handoff

**Read before implementing:**
- `[[architecture]]` §4
- `[[glossary]]`
- `src/server/database/sqlite.ts`
- One existing repository as a template: `src/server/database/repositories/client/` (read schema.ts, service.ts, types.ts)
- The latest existing migration in `src/server/database/migrations/`

**Modify these files:**
- (See `touches:` frontmatter — exhaustive list.)

**Acceptance tests:**
1. `pnpm db:generate` produces a clean migration with no extra diffs.
2. `pnpm test` includes the migration test and it passes.
3. `pnpm dev` against a populated DB starts cleanly; existing UI flows work.

**Self-test plan:**
```bash
pnpm install
pnpm db:generate
pnpm typecheck
pnpm test src/server/database
# point at a copied prod DB
DATABASE_URL=file:./wg-easy.db.copy pnpm dev
sqlite3 wg-easy.db.copy '.schema' | grep -E 'router|quota|speed_limit|usage_sample|audit_log'
```
