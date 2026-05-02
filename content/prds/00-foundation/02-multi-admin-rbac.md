---
id: PRD-00-02
title: Multi-admin RBAC — roles, ACL, audit log
status: shipped
phase: P0
depends_on:
  - "[[prds/00-foundation/03-auth-refactor]]"
  - "[[prds/00-foundation/04-data-model-migration]]"
touches:
  - src/shared/utils/permissions.ts
  - src/server/utils/permissions.ts
  - src/server/utils/audit.ts (new)
  - src/server/api/admin/users/index.get.ts (new)
  - src/server/api/admin/users/index.post.ts (new)
  - src/server/api/admin/users/[id]/index.get.ts (new)
  - src/server/api/admin/users/[id]/index.patch.ts (new)
  - src/server/api/admin/users/[id]/index.delete.ts (new)
  - src/server/api/admin/users/[id]/acl.get.ts (new)
  - src/server/api/admin/users/[id]/acl.put.ts (new)
  - src/server/api/admin/audit-log/index.get.ts (new)
  - src/app/pages/admin.vue
  - src/app/pages/admin/users/index.vue (new)
  - src/app/pages/admin/users/[id].vue (new)
  - src/app/pages/admin/audit-log.vue (new)
  - src/app/components/Admin/RoleBadge.vue (new)
  - src/app/middleware/auth.global.ts
  - src/server/database/sqlite.ts
---

# PRD-00-02 — Multi-admin RBAC

> ADR: [[decisions/0003-auth-model]]

> **Blocked on**: [[prds/00-foundation/03-auth-refactor]] (the `requirePermission` stub and `event.context.principal`). **Do not hand this PRD to an implementer until P0-03 is `status: shipped`.** P0-04 (`admin_router_acl`, `audit_log`, `router` tables) is now shipped. The `user.role` enum widening was punted from P0-04 and is explicitly in scope here.

## Why

Today there's one admin account. Real deployments need:
- A team where some members can manage everything, some can only manage clients but not server config, some can only view.
- **Per-router scoping** (P2 federation prep): admin Alice manages routers {A, B}, admin Bob manages router {C}.
- An **audit log** so destructive actions are attributable.

This PRD turns the `requirePermission` stub from [[prds/00-foundation/03-auth-refactor]] into a real RBAC check, adds user CRUD, and writes audit log entries for state-changing actions.

## User stories

- As a **superadmin**, I can invite a new admin, set their role, and scope them to specific routers.
- As an **admin scoped to router B**, I cannot see or modify router A's interfaces.
- As a **viewer**, I can read everything in my scope but cannot mutate.
- As an **operator** (client-only role), I can create/edit/delete clients on my routers but cannot edit interface or server settings.
- As a **superadmin**, I can review the audit log and filter by actor / action / target.

## Scope

### In

- Roles: `superadmin | admin | operator | viewer | client`. Definition table:

| Role | Scope | Capabilities |
| --- | --- | --- |
| superadmin | global | everything; bypasses ACL |
| admin | per-router (via ACL) | manage interfaces, clients, hooks, users *for routers in ACL* |
| operator | per-router (via ACL) | manage clients only |
| viewer | per-router (via ACL) | read-only |
| client | self | dashboard only |

- `admin_router_acl` rows carry `permission ∈ {read, write, admin}`. The role × ACL matrix decides allow/deny.
- User CRUD admin pages (`/admin/users`, `/admin/users/:id`).
- Audit log writer (`audit.ts:logAction`) called from every state-changing handler.
- Audit log viewer page (`/admin/audit-log`).
- Bootstrap rule: the first admin (existing single admin) is auto-promoted to `superadmin`. Migration handles this.

### Out

- Org/tenant model beyond per-router ACL.
- Fine-grained permissions inside a router (e.g., "can edit client X but not Y"). Out of scope; revisit if asked.
- LDAP/SCIM provisioning. SSO is later.

## Data model changes

P0-04 shipped `admin_router_acl`, `audit_log`, `router`, and the other foundation tables. This PRD writes/reads them.

`user.role` widening was punted from P0-04 because `src/shared/utils/permissions.ts` (which defines the branded `Role` type and constants) was outside P0-04's `touches:`. **This PRD extends `src/shared/utils/permissions.ts` with `SUPERADMIN=3`, `OPERATOR=4`, `VIEWER=5` and updates the `ROLES` matrix.** The DB column stays `int().$type<Role>()`; no column-type migration is needed.

Bootstrap step: a one-time data fix in `src/server/database/sqlite.ts` (called after `migrate()` on every startup, idempotent) promotes the existing single admin to `superadmin`. Detect: there is exactly one user with `role === roles.ADMIN`; set them to `roles.SUPERADMIN`. If multiple admin-role users exist, leave them alone (we already migrated).

## API changes

| Method | Path | Permission | Body | Notes |
| --- | --- | --- | --- | --- |
| GET | `/api/admin/users` | `admin:users` | — | list users |
| POST | `/api/admin/users` | `admin:users` | `{username, password, role, email?}` | create |
| GET | `/api/admin/users/[id]` | `admin:users` | — | detail |
| PATCH | `/api/admin/users/[id]` | `admin:users` | `{role?, enabled?, email?, password?}` | update |
| DELETE | `/api/admin/users/[id]` | `admin:users` (cannot delete self) | — | delete |
| GET | `/api/admin/users/[id]/acl` | `admin:users` | — | list ACL rows |
| PUT | `/api/admin/users/[id]/acl` | `admin:users` | `[{routerId, permission}]` | replace ACL |
| GET | `/api/admin/audit-log` | `admin:settings` | query: actor, action, target, since, until, limit | paginated |

Existing admin endpoints get explicit `requirePermission` calls. Examples:
- `POST /api/admin/interface/...` → `requirePermission(event, 'router:write', { routerId })`.
- `POST /api/client` → `requirePermission(event, 'client:write', { routerId, clientId })`.

## UI changes

- `src/app/pages/admin/users/index.vue` — table of users, role badge, "invite" button.
- `src/app/pages/admin/users/[id].vue` — detail / edit, ACL editor (router multi-select with permission per row).
- `src/app/pages/admin/audit-log.vue` — paginated table, filter chips.
- `src/app/components/Admin/RoleBadge.vue` — small reusable badge.
- Sidebar: show "Users" and "Audit log" menu items only if `principal.user.role === roles.SUPERADMIN` or has `admin:users`/`admin:settings`.

## Driver / backend changes

### Real `requirePermission` implementation

```ts
// src/server/utils/permissions.ts
import { roles, type Role } from '#shared/utils/permissions';

export type Permission =
  | 'router:read' | 'router:write' | 'router:admin'
  | 'client:read' | 'client:write'
  | 'admin:users' | 'admin:settings'
  | 'dashboard:self';

const ROLE_PERMS: Record<Role, Permission[]> = {
  [roles.SUPERADMIN]: ['*'],
  [roles.ADMIN]:    ['router:read', 'router:write', 'router:admin', 'client:read', 'client:write', 'admin:users'],
  [roles.OPERATOR]: ['router:read', 'client:read', 'client:write'],
  [roles.VIEWER]:   ['router:read', 'client:read'],
  [roles.CLIENT]:   ['dashboard:self'],
};

export async function requirePermission(event, perm: Permission, resource?) {
  const p = event.context.principal;
  if (!p) throw createError({ statusCode: 401 });
  if (p.kind === 'token' && !p.scopes.includes(perm)) throw createError({ statusCode: 403 });

  const u = p.user;
  if (u.role === roles.SUPERADMIN) return;

  const rolePerms = ROLE_PERMS[u.role];
  if (!rolePerms?.includes(perm)) throw createError({ statusCode: 403 });

  // Router-scoped permissions require an ACL row
  if (resource?.routerId !== undefined && (perm.startsWith('router:') || perm.startsWith('client:'))) {
    const acl = await Database.adminRouterAcls.find(u.id, resource.routerId);
    if (!acl) throw createError({ statusCode: 403 });
    if (perm.endsWith(':write') && acl.permission === 'read') throw createError({ statusCode: 403 });
    if (perm.endsWith(':admin') && acl.permission !== 'admin') throw createError({ statusCode: 403 });
  }
}
```

### Audit helper

```ts
// audit.ts
export async function logAction(event, action: string, target: object, result: 'ok'|'error' = 'ok'): Promise<void> {
  const p = event.context.principal;
  await db.auditLog.insert({
    actorUserId: p?.user.id ?? null,
    action, target: JSON.stringify(target), result,
    ts: new Date(),
  });
}
```

Call sites: all NEW handlers introduced in this PRD (`/api/admin/users/**`, `/api/admin/audit-log`). Existing handlers in `src/server/api/admin/**` and `src/server/api/client/**` are instrumented incrementally in a follow-up PRD to keep the review surface bounded.

## Migration & rollout

- Schema already shipped in P0-04. This PRD writes the auto-promotion step as a one-shot bootstrap in `src/server/database/sqlite.ts` (called after `migrate()` on every startup, idempotent).
- Roll out behind no flag; the default state (one superadmin, no ACL rows) preserves current behavior because superadmin bypasses ACL.

## Verification

### Unit tests

- `permissions.test.ts` — matrix: each role × each permission × with/without ACL row × action-perm.
- `audit.test.ts` — every wrapper writes a row.
- `api/admin/users.test.ts` — CRUD; cannot delete self; cannot demote last superadmin.

### Integration test

- Create three users (admin, operator, viewer), each scoped to router 0.
- For each, attempt the matrix: list interfaces, mutate interface, list clients, mutate client, view audit log. Assert correct allow/deny.

### Manual test plan

1. Log in as existing admin → confirm promoted to superadmin.
2. Create a new user "alice" with role `operator`, ACL `{router_id: 0, permission: write}`.
3. Log out; log in as alice; verify she cannot see "Server settings" or "Users" menu items but can manage clients.
4. Try to disable a client as alice → succeeds; audit log shows her as actor.
5. As superadmin, view audit log; filter by `action=client.disable`; row visible.

## Open questions

- [ ] Should `operator` be allowed to disable a client (which is a client mutation but with security implications)? Decision: yes, `client:write` covers enable/disable. Quota auto-disables already happen as a system action (actor null) — those audit rows have actor null and `action=quota.auto_disable`.

---

## Kimi handoff

**Read before implementing:**
- `[[architecture]]` §5c
- `[[decisions/0003-auth-model]]`
- `[[prds/00-foundation/03-auth-refactor]]` (especially `requirePermission` stub)
- `[[prds/00-foundation/04-data-model-migration]]` (`admin_router_acl`, `audit_log`, `user.role`)
- `src/server/utils/permissions.ts` (the P0-03 stub)
- `src/shared/utils/permissions.ts` (extend with SUPERADMIN / OPERATOR / VIEWER)
- `src/server/database/repositories/user/`
- `src/server/database/repositories/adminRouterAcl/`
- `src/server/database/repositories/auditLog/`
- `src/server/database/repositories/router/`
- `src/app/pages/admin/general.vue` (as a layout reference for the new admin pages)

**Modify these files:** see `touches:` frontmatter.

**Acceptance tests:**
1. Permission matrix tests pass.
2. Audit log entries present for all state-changing actions in new handlers introduced by this PRD (assert via DB query in test).
3. UI: a viewer-role user sees no mutation buttons.
4. The existing single-admin instance auto-upgrades to superadmin on first start.

**Self-test plan:**
```bash
pnpm test src/server/utils/permissions.test.ts
pnpm test src/server/utils/audit.test.ts
pnpm test src/server/api/admin/users
pnpm dev
# manual: follow test plan above
```
