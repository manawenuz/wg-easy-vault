---
id: PRD-00-03
title: Auth refactor — admin/user split, bearer tokens, principal context
status: shipped
phase: P0
depends_on:
  - "[[prds/00-foundation/04-data-model-migration]]"
touches:
  - src/server/utils/session.ts
  - src/server/utils/principal.ts (new)
  - src/server/utils/permissions.ts (new)
  - src/server/api/session.post.ts
  - src/server/api/session.get.ts
  - src/server/api/session.delete.ts
  - src/server/api/user-session.post.ts (new)
  - src/server/api/user-session.delete.ts (new)
  - src/server/api/api-tokens/index.get.ts (new)
  - src/server/api/api-tokens/index.post.ts (new)
  - src/server/api/api-tokens/[id].delete.ts (new)
  - src/app/middleware/auth.global.ts
  - src/app/stores/auth.ts
---

# PRD-00-03 — Auth refactor

> ADR: [[decisions/0003-auth-model]] · Spec ref: [[architecture#5-auth-flows]]

## Why

Today the codebase has a single auth mechanism (admin session cookie) and every route handler does ad-hoc `hasPermissions` checks. Multi-admin (next PRD), user dashboards, API tokens, and SSO all stack on top of auth — the ad-hoc model doesn't scale. We replace it with a **principal context** that the middleware builds once per request, plus a small `requirePermission` helper.

This PRD does **not** add roles or ACLs (that's [[prds/00-foundation/02-multi-admin-rbac]]). It only restructures auth so RBAC can land cleanly.

## User stories

- As an **engineer**, I write `requirePermission(event, 'router:write', router_id)` instead of cobbling together checks.
- As an **admin**, my existing cookie continues to work. No re-login.
- As an **operator**, I can mint an API token with a label and revoke it later.
- As an **end user**, I can hit `/dashboard` and get a separate session that doesn't carry admin powers (login flow itself is in [[prds/20-user-features/02-qr-key-login]]).

## Scope

### In

- Two cookie names: `wg-session` (admin, existing) and `wg-user-session` (new).
- `event.context.principal: Principal` — `{ kind: 'admin' | 'user' | 'token', user, scopes? }`.
- `requirePermission(event, action, resourceId?)` helper. **Without RBAC**, it stubs to: admin → allow; user → allow only for routes tagged `dashboard`; token → check scope. RBAC fills it in next.
- API token CRUD endpoints under `/api/api-tokens` (admin only).
- `Authorization: Bearer <token>` recognized alongside existing `Basic` in `session.ts`. Tokens hashed at rest with argon2id.
- Backwards-compat: admin cookie continues to work; existing `useWGSession()` calls continue to resolve.

### Out

- Role enum expansion / ACL / audit log → next PRD.
- User-session login itself (challenge/sign flow with WG keys) → P1 PRD.
- SSO callback → P3.
- TOTP changes — leave admin TOTP path alone.

## Data model changes

`api_token` table already exists from [[prds/00-foundation/04-data-model-migration]]. No further schema changes.

## API changes

| Method | Path | Auth | Body | Returns |
| --- | --- | --- | --- | --- |
| POST | `/api/session` | none | `{username, password, totp?}` | sets `wg-session` cookie |
| GET | `/api/session` | cookie | — | current admin or null |
| DELETE | `/api/session` | cookie | — | clears cookie |
| POST | `/api/user-session` | none | `{publicKey, signature, challenge}` (placeholder; full impl in QR/key PRD) | sets `wg-user-session` |
| DELETE | `/api/user-session` | user cookie | — | clears |
| GET | `/api/api-tokens` | admin | — | `[{id, label, scopes, expires_at, last_used_at}]` |
| POST | `/api/api-tokens` | admin | `{label, scopes[], expiresAt?}` | `{id, token}` (token shown ONCE) |
| DELETE | `/api/api-tokens/[id]` | admin | — | `{ok}` |

`Authorization: Bearer <token>` works on any endpoint that accepts a session cookie, with permissions intersected by token scopes.

## UI changes

- `src/app/stores/auth.ts` extended: `principal` instead of just `user`. UI keeps showing admin info for admin sessions.
- A new admin page `/admin/api-tokens` for token CRUD (Vue page, simple list + create modal that surfaces the secret once).

## Driver / backend changes

### Principal type

```ts
// src/server/utils/principal.ts
export type Principal =
  | { kind: 'admin'; user: UserType }
  | { kind: 'user'; user: UserType }   // dashboard user
  | { kind: 'token'; user: UserType; tokenId: number; scopes: string[] };

export async function resolvePrincipal(event: H3Event): Promise<Principal | null> {
  // 1. Authorization: Bearer <token> → token principal (lookup api_token by hash)
  // 2. Authorization: Basic <b64> → admin via existing path
  // 3. wg-session cookie → admin
  // 4. wg-user-session cookie → user
  // 5. null
}
```

### Permission helper (stub)

```ts
// src/server/utils/permissions.ts
export type Permission =
  | 'router:read' | 'router:write' | 'router:admin'
  | 'client:read' | 'client:write'
  | 'admin:users' | 'admin:settings'
  | 'dashboard:self';

export async function requirePermission(
  event: H3Event,
  perm: Permission,
  resource?: { routerId?: number; userId?: number; clientId?: number },
): Promise<void> {
  const p = event.context.principal as Principal | null;
  if (!p) throw createError({ statusCode: 401 });

  // Stub — full RBAC in PRD-00-02
  if (p.kind === 'admin') return;
  if (p.kind === 'token') {
    if (!p.scopes.includes(perm)) throw createError({ statusCode: 403 });
    return;
  }
  if (p.kind === 'user') {
    if (perm === 'dashboard:self') return;
    if (resource?.userId === p.user.id) return;  // user's own resources
    throw createError({ statusCode: 403 });
  }
}
```

### Middleware

`src/app/middleware/auth.global.ts` — run `resolvePrincipal` once, attach to `event.context.principal`, redirect to `/login` (admin) or `/dashboard/login` (user) based on the path namespace.

## Migration & rollout

- Existing `wg-session` cookie continues to work (no key change).
- Endpoints adopt `requirePermission` incrementally; the stub keeps current behavior. RBAC PRD ratchets the stub to real checks.
- API tokens are opt-in: nothing breaks if no tokens exist.

## Verification

### Unit tests

- `principal.test.ts` — for each input (cookie, bearer, basic, none), resolves the right `Principal` or null.
- `permissions.test.ts` — admin always allowed; token with matching scope allowed; user denied admin perms; user allowed `dashboard:self`.
- `api-tokens.test.ts` — create returns plaintext token once; subsequent GET hides it; DELETE removes it; expired token rejected.

### Integration test

- Existing admin login → still works.
- Mint API token via UI → use it with `curl -H 'Authorization: Bearer <t>' /api/client` → succeeds.
- Same token after `DELETE /api/api-tokens/<id>` → 401.

### Manual test plan

1. Log into admin UI — works as before.
2. Visit `/admin/api-tokens` → create token "ci-readonly" with scope `client:read`, copy secret.
3. `curl -H 'Authorization: Bearer <t>' http://localhost:51821/api/client` → returns clients.
4. `curl -H 'Authorization: Bearer <t>' -X POST http://localhost:51821/api/client ...` → 403 (no `client:write` scope).
5. Revoke token; same curl → 401.

## Open questions

- [ ] Token format: random 32-byte URL-safe string with a prefix `wgep_` for grep-ability in logs. Acceptable? Going with yes unless objection.

---

## Kimi handoff

**Read before implementing:**
- `[[architecture]]` §5
- `[[decisions/0003-auth-model]]`
- `[[prds/00-foundation/04-data-model-migration]]` (api_token schema)
- `src/server/utils/session.ts` (full)
- `src/server/api/session.post.ts`, `src/server/api/session.get.ts`, `src/server/api/session.delete.ts`
- `src/app/middleware/auth.global.ts`
- `src/app/stores/auth.ts`
- One existing repository as a template (e.g. `repositories/user/`)

**Modify these files:** see `touches:` frontmatter.

**Acceptance tests:**
1. Existing admin auth flows untouched (regression).
2. API token CRUD works end-to-end.
3. Bearer token with scope works; without scope returns 403.
4. `wg-user-session` cookie path resolves a `user` principal (the actual login that *sets* this cookie ships in [[prds/20-user-features/02-qr-key-login]]; for this PRD test by manually crafting a session for an existing client-role user).

**Self-test plan:**
```bash
pnpm test src/server/utils/principal.test.ts src/server/utils/permissions.test.ts
pnpm test src/server/api/api-tokens
pnpm dev
# manual: see test plan
```
