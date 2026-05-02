---
id: PRD-40-02
title: Admin–router ACL — scope admins to specific routers
status: draft
phase: P2
depends_on:
  - "[[prds/40-multi-server/01-multi-router-federation]]"
  - "[[prds/00-foundation/02-multi-admin-rbac]]"
touches:
  - src/server/utils/permissions.ts
  - src/app/components/Users/AclEditor.vue (new)
  - src/app/pages/admin/users/[id].vue
---

# PRD-40-02 — Admin–router ACL enforcement

## Why

[[prds/00-foundation/02-multi-admin-rbac]] introduced the `admin_router_acl` table and stub enforcement. Federation makes ACLs concretely valuable: admin Alice manages routers in region A, Bob manages region B, neither sees the other's clients. This PRD finalizes the enforcement and adds the editor UI.

## Scope

### In

- `requirePermission` honors `admin_router_acl` exactly as specified in [[prds/00-foundation/02-multi-admin-rbac]] (it was implemented in stub form). Tighten:
  - List endpoints filter by ACL (e.g., `GET /api/admin/router` returns only routers the user has `read` on).
  - Listing interfaces / clients filters by router ACL.
  - The audit log filters to actions on routers the user can read (superadmin sees all).
- `AclEditor.vue` — multi-select of routers with permission radio per router, used inside `/admin/users/[id]`.
- Bulk-edit support: assign multiple users to one router at once via `/admin/routers/[id]/admins` (small admin page).

### Out

- Per-interface ACL (more fine-grained than per-router). Not needed in v1.
- Group-based ACL. Roles already serve this purpose at coarse granularity.

## Verification

- Unit tests: matrix of (user, router, permission, action) → allow/deny.
- Integration: create three users with disjoint ACLs; confirm each only sees their scope.
- Audit: actions trying to cross scope produce a 403 + audit entry.

---

## Kimi handoff

**Read before implementing:**
- `[[prds/00-foundation/02-multi-admin-rbac]]`
- `[[prds/40-multi-server/01-multi-router-federation]]`
- `src/server/utils/permissions.ts`

**Acceptance tests:**
1. Cross-scope reads return filtered lists, not 403s (don't leak existence).
2. Cross-scope writes return 403 with audit entry.
3. AclEditor saves and reloads correctly.

**Self-test plan:**
```bash
pnpm test src/server/utils/permissions.test.ts
pnpm dev
# manual: three-user matrix walkthrough
```
