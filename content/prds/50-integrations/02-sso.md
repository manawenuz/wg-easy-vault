---
id: PRD-50-02
title: SSO — OIDC (and SAML, maybe)
status: draft
phase: P3
depends_on:
  - "[[prds/00-foundation/03-auth-refactor]]"
  - "[[prds/00-foundation/02-multi-admin-rbac]]"
touches:
  - src/server/auth/oidc.ts (new)
  - src/server/auth/saml.ts (new, maybe)
  - src/server/api/auth/oidc/login.get.ts (new)
  - src/server/api/auth/oidc/callback.get.ts (new)
  - src/app/pages/login.vue
  - src/app/pages/admin/integrations/sso.vue (new)
---

# PRD-50-02 — SSO

> ADR: [[decisions/0003-auth-model]]

## Why

Org deployments want admin login via their existing identity provider (Google, Okta, Authentik, etc.). Username/password + TOTP is fine for solo, but as soon as a team has >3 admins, SSO becomes non-negotiable. **Admin SSO only** in scope — end-users still log in via the QR/key flow.

## User stories

- As an **admin team**, we configure OIDC against our IdP; admins click "Sign in with SSO" and are dispatched to the IdP, returned with an admin session.
- As a **superadmin**, I configure: provider type, client id/secret, discovery URL, role-claim mapping, and whether to auto-provision unknown users.
- As an **admin** logged in via SSO, I cannot also have a local password (mutually exclusive per user).

## Scope

### In

- **OIDC** via `openid-client` library. Authorization Code + PKCE flow.
- Configurable role mapping: claims-based (e.g., `groups: ['vpn-admins']` → role `admin`).
- Auto-provisioning toggle: if true, an unknown email signing in is created with default role; if false, login fails for unknown users.
- Coexistence with local password login (both available; SSO is *additional*).
- TOTP on user is auto-disabled when they're tied to SSO.

### Out — not in v1

- **SAML**: research-leaning; capture as a follow-up. Build OIDC first; SAML adds significant complexity (cert handling, signed assertions). Most modern IdPs support OIDC.
- SCIM provisioning. Manual / claim-based only.
- Per-router scope mapping from IdP claims. Map to roles only; ACL stays in-app.

## Data model changes

Add to `user`:
- `sso_subject TEXT` — the IdP `sub` claim, unique when set.
- `sso_provider TEXT`.

Add `sso_config` (single row, settings):
- `enabled`, `provider_type`, `client_id`, `client_secret_encrypted`, `discovery_url`, `role_claim_path`, `role_claim_map_json`, `auto_provision`, `default_role`.

Migration is small, self-contained.

## API changes

| Method | Path | Notes |
| --- | --- | --- |
| GET | `/api/auth/oidc/login` | redirects to IdP authorize endpoint |
| GET | `/api/auth/oidc/callback` | handles code exchange, mints admin session |
| GET | `/api/admin/integrations/sso` | get config |
| PUT | `/api/admin/integrations/sso` | update config (superadmin) |

## UI changes

- `/login` page gets a "Sign in with SSO" button when `sso_config.enabled`.
- Admin page `/admin/integrations/sso` for configuration.

## Verification

- Unit: claim mapping logic.
- Integration: stand up an Authentik or `oidc-mock` test IdP; full login flow round-trip.

## Open questions

- [ ] Logout: do we hit the IdP's end-session endpoint or just clear local cookie? Default: clear local cookie; provide an "SSO logout" toggle for orgs that want global logout.
- [ ] Multiple IdPs simultaneously? v1 = one. Multi-IdP is a UX cost (which button to show); revisit.

---

## Kimi handoff

**Read before implementing:**
- `[[decisions/0003-auth-model]]`
- `[[prds/00-foundation/03-auth-refactor]]`
- `openid-client` README
- A reference implementation (e.g., a Nuxt/Nitro OIDC example)

**Acceptance tests:**
1. Round-trip OIDC login against `oidc-mock`.
2. Role mapping creates the right local role.
3. Unknown user with `auto_provision=false` → 403 with audit row.

**Self-test plan:**
```bash
pnpm test src/server/auth/oidc.test.ts
docker run -d -p 9090:9090 ghcr.io/dexidp/dex-mock-idp
# configure wg-easy SSO pointing at it
pnpm dev
```
