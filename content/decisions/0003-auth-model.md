---
id: ADR-0003
title: Auth model — admin sessions, user sessions, API tokens
status: decided
date: 2026-05-02
---

# ADR-0003 — Auth model

## Context

Today wg-easy has one auth flow: admin username/password (+ optional TOTP) → encrypted cookie. We need:

- **Multi-admin** with roles (superadmin, admin, operator, viewer) and per-router scoping.
- **End-user dashboards** where a non-admin user logs in to see their own VPN clients, usage, and quota.
- **Programmatic access** (API tokens) for CI/CD, monitoring, and the federation agent protocol.
- **Optional SSO** (OIDC/SAML) — P3, must not constrain the design now but should not require a third rewrite later.

## Decision

Three auth surfaces, **two underlying mechanisms**:

| Surface | Mechanism | Cookie / header |
| --- | --- | --- |
| Admin login | Encrypted session cookie (existing) | `wg-session` |
| User dashboard login | Encrypted session cookie, **separate name** | `wg-user-session` |
| API tokens | Bearer token in `Authorization` header (extend existing Basic Auth path) | `Authorization: Bearer <token>` |

User dashboard auth is **password-less**: user proves they hold a WireGuard private key by signing a server-issued challenge (Curve25519). See [[prds/20-user-features/02-qr-key-login]].

SSO (when added) issues admin sessions only — it's another way to acquire the admin cookie, not a parallel auth scheme.

## Reasoning

### Why separate cookies for admin vs user

- Different threat models: admin cookie loss = compromise of the whole control plane; user cookie loss = compromise of one user. Separate names prevent accidental privilege confusion in middleware.
- Different lifetimes: admin sessions short (configurable, default 12h); user sessions can be longer (default 30d) since the underlying credential (WG private key) is itself the long-lived secret.
- Different namespaces in middleware — `auth.global.ts` resolves to one of `{adminUser, dashboardUser, null}`, never both.

### Why password-less user dashboard

- Users already have a credential: their WireGuard private key. Asking them to remember a second password is friction without security gain — if the WG key leaks, the dashboard password is moot anyway.
- Curve25519 signing is cheap and the math is identical to what WireGuard uses, so no new crypto primitives.
- The QR code admins already hand out becomes the login token. Users don't see a "create an account" step.

### Why bearer tokens piggyback on the Basic Auth path

- `session.ts:24-116` already decodes `Authorization` headers and resolves a user. We extend it to recognize `Bearer <token>` in addition to `Basic <b64>`. One code path, two formats.
- Tokens are stored in a new `api_token` table (token hash, owner user_id, scopes, expires_at).
- API tokens map to **a user**, not a separate principal. Permissions follow the user's role and ACL.

### RBAC layering

- `user.role` is a string enum: `superadmin | admin | operator | viewer | client`.
- `client` role = end user with a dashboard, no admin powers.
- For everything except `superadmin` and `client`, scope is intersected with `admin_router_acl(user_id, router_id, permission)`. `superadmin` bypasses ACL.
- Permissions: `read`, `write`, `admin` (router-level admin = manage interfaces, peers, hooks on that router).
- Every state-changing action writes one row to `audit_log`.

### SSO compatibility

- When SSO lands, the OIDC callback resolves a user by email, creates the user if `auto_provision: true`, and issues the admin cookie. Role/ACL come from the local DB or from claims (configurable). The cookie format does not change.
- TOTP becomes mutually exclusive with SSO per user: SSO IdP handles MFA.

## Consequences

- Existing admin login path is preserved; no breaking change for current users.
- A new `auth.global.ts` resolves both cookie names and exposes `event.context.principal: { kind: 'admin'|'user', user, scopes }`.
- Routes declare required permissions via a small helper (`requirePermission(event, 'router:write', router_id)`), removing ad-hoc admin checks scattered across handlers.
- `api_token` table with hashed tokens (Argon2id), never stored in plaintext.

## Alternatives considered

- **One cookie, role-based dispatch** — rejected. Mixing admin and user sessions in one cookie space invites privilege confusion bugs. Also complicates lifetime tuning.
- **JWT instead of encrypted session cookies** — rejected. Existing infrastructure is encrypted cookies; JWTs add revocation complexity (which is exactly what cookies-with-server-side-session avoid).
- **Treat the WG private key as the session token directly** — rejected. The user would need to send the key on every request; a single XSS or accidentally pasted log line leaks the VPN credential, not just a session.

## Related PRDs

- [[prds/00-foundation/03-auth-refactor]] — implements the cookie split, bearer token path, principal context.
- [[prds/00-foundation/02-multi-admin-rbac]] — implements roles, ACL, audit log.
- [[prds/20-user-features/02-qr-key-login]] — implements password-less dashboard login.
- [[prds/50-integrations/02-sso]] — adds OIDC/SAML on top.
