---
id: PRD-20-01
title: User dashboard — usage graph, status, expiry
status: shipped
phase: P1
depends_on:
  - "[[prds/00-foundation/03-auth-refactor]]"
  - "[[prds/00-foundation/04-data-model-migration]]"
touches:
  - src/app/pages/dashboard/index.vue (new)
  - src/app/pages/dashboard/clients/[id].vue (new)
  - src/app/pages/dashboard/login.vue (new)
  - src/app/layouts/dashboard.vue (new)
  - src/app/middleware/auth.global.ts
  - src/app/stores/dashboard.ts (new)
  - src/server/api/dashboard/me.get.ts (new)
  - src/server/api/dashboard/clients/index.get.ts (new)
  - src/server/api/dashboard/clients/[id]/usage.get.ts (new)
  - src/server/api/dashboard/clients/[id]/configuration.get.ts (new)
  - src/server/api/dashboard/clients/[id]/qrcode.svg.get.ts (new)
  - src/server/api/dashboard/me.get.test.ts (new)
  - src/server/api/dashboard/clients/index.get.test.ts (new)
  - src/server/api/dashboard/clients/[id]/usage.get.test.ts (new)
---

# PRD-20-01 — User dashboard

> ADR: [[decisions/0003-auth-model]]

## Why

Users today have no self-service. They ping the admin for a config, again to check why VPN dropped, again when they hit a quota. A dashboard that shows status, usage, expiry, and lets them re-download configs eliminates most of that ticket volume. It's also the prerequisite UI for the QR/key login flow ([[prds/20-user-features/02-qr-key-login]]).

## User stories

- As a **user**, I can log in (with QR or key — flow in next PRD; this PRD assumes the session exists) and see all my VPN clients.
- As a **user**, I can see for each client: connected/disconnected, last handshake, current rx/tx, this period's usage vs. quota, expiry date, speed limit (if any).
- As a **user**, I can re-download my client config or QR code.
- As a **user**, I see a usage graph for the last 24h / 7d / 30d.
- As a **user**, I cannot do anything that affects other users or admin settings.

## Scope

### In

- `/dashboard` route family with its own layout (no admin sidebar).
- API namespace `/api/dashboard/*` that strictly returns/affects only the requesting user's data.
- Components: client status card, usage chart (line, two series rx/tx), period quota progress bar, expiry countdown, "download config" + "show QR" buttons.
- Read-only by default. The only mutation is "download/regenerate config" — which we expose as **download only** (no key regeneration in v1, to keep the threat model simple).
- Empty-state: if user has no clients, show "Contact your administrator."

### Out

- Login flow itself ([[prds/20-user-features/02-qr-key-login]]).
- Letting users create new clients themselves. Decision: admins create clients; users consume.
- Letting users edit DNS, allowed-IPs, etc. — admin-only.
- Notifications / email on quota approach. Captured as a follow-up after P1.

## Data model changes

None. This PRD reads from existing tables.

## API changes

All endpoints require `event.context.principal.kind === 'user'` and scope by `principal.user.id`.

| Method | Path | Returns |
| --- | --- | --- |
| GET | `/api/dashboard/me` | `{ user: { id, name, email }, clientsCount }` |
| GET | `/api/dashboard/clients` | `[{ id, name, enabled, ipv4, lastHandshakeAt, rxBytes, txBytes, expiresAt, quota?, speedLimit? }]` |
| GET | `/api/dashboard/clients/[id]/usage?range=24h\|7d\|30d` | `{ buckets: [{ ts, rxBytes, txBytes }] }` |
| GET | `/api/dashboard/clients/[id]/configuration` | text/plain WG config |
| GET | `/api/dashboard/clients/[id]/qrcode.svg` | image/svg |

The usage endpoint reads `usage_sample` and bucket-rolls server-side: 24h → 5min buckets, 7d → hourly, 30d → daily.

## UI changes

- `src/app/layouts/dashboard.vue` — header with username, logout, no admin sidebar.
- `src/app/pages/dashboard/index.vue` — list of client cards (one card per client).
- `src/app/pages/dashboard/clients/[id].vue` — detail view with usage chart and metadata.
- Reuse existing `Clients/QRCodeDialog.vue` if structure allows; otherwise a thin wrapper.
- Charting: small dependency choice. Recommendation: `chart.js` via `vue-chartjs` (already commonly used in Nuxt). If a simpler lib is preferred (e.g. uPlot), justify in the diff message.

## Driver / backend changes

None to engines. This is a UI + read API PRD.

Service-layer additions:
- `dashboardService.getMyClients(userId)` — joins `client`, `quota`, `speed_limit`, latest `usage_sample`.
- `dashboardService.getUsageBuckets(userId, clientId, range)` — server-side aggregation query.

## Migration & rollout

- New routes; no migration risk.
- Feature flag `ENABLE_USER_DASHBOARD` (env var, default `true` in dev, gradual rollout in prod) so an admin can hide the dashboard until the login flow ships.

## Verification

### Unit tests

- `dashboard/clients.get.test.ts` — returns only the requester's clients; another user's clients are not leaked.
- `dashboard/usage.get.test.ts` — bucket aggregation correctness against a seeded `usage_sample` set.
- `dashboard/me.get.test.ts` — returns the right user; rejects admin sessions (admins don't use the dashboard, they have the admin UI).

### Integration test

- Seed a user with two clients; log in (mock the user-session cookie); load `/dashboard`; assert both clients render, the inactive one is gray, the active one shows last-handshake.

### Manual test plan

1. As an admin, create a `client`-role user "bob" with two VPN clients.
2. Manually set a `wg-user-session` cookie for bob (or wait until [[prds/20-user-features/02-qr-key-login]] ships).
3. Visit `/dashboard` → see both clients.
4. Click one → usage graph renders for last 24h.
5. Click "Download config" → file downloads.
6. Try `/admin` while signed in as user → redirected to dashboard with "no permission" toast.

## Open questions

- [ ] Time zone for "this period" displays. Decision: server is authoritative (UTC), UI shows in user's browser TZ, with a tooltip showing UTC.
- [ ] Should we let users **revoke** their own client? Decision: no in v1. Hand off via admin; reconsider with usage data.

---

## Kimi handoff

**Read before implementing:**
- `[[architecture]]` §1, §5b
- `[[decisions/0003-auth-model]]`
- `[[prds/00-foundation/03-auth-refactor]]`
- `src/app/middleware/auth.global.ts` (after auth-refactor lands)
- Existing components: `src/app/components/Clients/List.vue`, `Clients/QRCodeDialog.vue`
- Existing API: `src/server/api/client/[clientId]/configuration.get.ts`, `qrcode.svg.get.ts`

**Modify these files:** see `touches:` frontmatter.

**Acceptance tests:**
1. User-scoped APIs never leak other users' data (write a test that asserts this).
2. Usage endpoint returns correctly bucketed data.
3. Visiting `/admin` as a `user` principal redirects out with a toast.
4. `ENABLE_USER_DASHBOARD=false` hides routes (404) and the login link.

**Self-test plan:**
```bash
pnpm test src/server/api/dashboard
pnpm dev
# manual: see test plan
```

## Resolution log (2026-05-02)

- All dashboard routes, APIs, and tests implemented per spec.
- All APIs enforce `principal.kind === 'user'` explicitly (the existing `requirePermission('dashboard:self')` also allows admin principals; added guard to match PRD intent).
- Usage bucketing: 24h → 5min, 7d → hourly, 30d → daily, computed in-memory from `usage_sample` rows.
- Charting uses existing ApexCharts (vue3-apexcharts) — no new dependency.
- `ENABLE_USER_DASHBOARD=false` hides dashboard UI routes via middleware; APIs remain callable (feature-flag is UI-only).
- Tests: 8 new assertions across 3 test files; all 78 unit tests pass.
- i18n keys for dashboard UI are referenced but not yet translated — deferred to translation sweep.
- Dashboard login is a placeholder (public-key paste → `/api/user-session`); full QR/key challenge flow ships in PRD-20-02.
