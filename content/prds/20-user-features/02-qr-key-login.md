---
id: PRD-20-02
title: User dashboard login by QR or WireGuard key
status: approved
phase: P1
depends_on:
  - "[[prds/20-user-features/01-user-dashboard]]"
  - "[[prds/00-foundation/03-auth-refactor]]"
touches:
  - src/server/api/dashboard/login/challenge.post.ts (new)
  - src/server/api/dashboard/login/verify.post.ts (new)
  - src/server/api/dashboard/logout.post.ts (new)
  - src/server/utils/wgKeyAuth.ts (new)
  - src/app/pages/dashboard/login.vue
  - src/app/components/Dashboard/QrLogin.vue (new)
  - src/app/components/Dashboard/PasteConfigLogin.vue (new)
---

# PRD-20-02 — User dashboard login by QR or key

> ADR: [[decisions/0003-auth-model]] · Spec ref: [[architecture#5b-user-dashboard-login-new-by-qr-or-pubkey]]

## Why

Users already have a credential — their WireGuard private key. Asking them to remember a separate dashboard password is friction without security gain. We let them log in by **proving they hold the private key** for one of their VPN clients: scan the QR they were given, or paste the config file. No password.

## User stories

- As a **user**, I scan my WireGuard QR with my phone camera (or the dashboard's built-in webcam scanner) and I'm logged in.
- As a **user**, I paste my `.conf` file into a textarea and I'm logged in.
- As a **user**, my session lasts long enough that I don't re-authenticate weekly. (Default 30 days; admin-configurable.)
- As an **admin**, I can revoke a user's dashboard sessions (by disabling the corresponding clients).

## Scope

### In

- Server-issued challenge (random 32 bytes, 60-second TTL, single-use) signed client-side by the user's WG private key. Verification uses the same Curve25519 primitives WireGuard uses.
- Two UI flows on `/dashboard/login`:
  1. **Scan QR**: webcam capture (browser MediaDevices API) → decode QR → extract `PrivateKey` and `Endpoint`/`Address` → sign challenge → submit.
  2. **Paste config**: textarea → parse → same flow.
- Both flows happen entirely **client-side** for the private-key handling; only the public key and signature go to the server.
- 30-day session cookie `wg-user-session` (configurable).
- Logout endpoint that clears the cookie and invalidates server-side session record (we add a tiny `user_session` cache in DB or in-memory; optional — see open questions).
- Brute-force defenses: rate limit the verify endpoint per IP (10/min) and per challenge id (1).

### Out

- Email magic links. No email infra in scope.
- Allowing users to log in with **any** key, even one not in our DB. The public key must match an existing `client.public_key` whose owner has role `client`.
- Hardware-key (FIDO) login. Not in scope.

## Data model changes

Optional `user_session` table for revocation:
```ts
export const userSession = sqliteTable('user_session', {
  id: text('id').primaryKey(),         // session id
  userId: integer('user_id').notNull(),
  clientId: integer('client_id').notNull(),  // which client they auth'd as
  createdAt: integer('created_at', { mode: 'timestamp' }).notNull(),
  expiresAt: integer('expires_at', { mode: 'timestamp' }).notNull(),
  revokedAt: integer('revoked_at', { mode: 'timestamp' }),
});
```

Add to [[prds/00-foundation/04-data-model-migration]] as a follow-up if this PRD ships before that one merges. Otherwise: append a small migration in this PRD.

## API changes

| Method | Path | Body | Returns |
| --- | --- | --- | --- |
| POST | `/api/dashboard/login/challenge` | `{ publicKey }` | `{ challengeId, nonce }` (nonce = base64 32 bytes) |
| POST | `/api/dashboard/login/verify` | `{ challengeId, signature }` | sets `wg-user-session` cookie, returns `{ ok }` |
| POST | `/api/dashboard/logout` | — | clears cookie |

Challenges held in-memory with TTL (60s). On verify: lookup challenge → fetch client by public key → verify signature using the public key → check `client.user.role === 'client'` and `client.enabled` → mint session.

## UI changes

- `src/app/pages/dashboard/login.vue` — tabs: "Scan QR" | "Paste config".
- `QrLogin.vue` — uses `qr-scanner` npm lib (BSD-licensed, no Google APIs) on a `<video>` element. No camera roll upload; live capture only (mobile cameras need to be on https).
- `PasteConfigLogin.vue` — textarea, parse on submit, show errors inline.
- Both call `useDashboardAuth().login(privateKeyBase64)`, which:
  1. Derives the public key locally (libsodium / `tweetnacl`).
  2. POSTs `/api/dashboard/login/challenge` with the public key.
  3. Signs the returned nonce with the private key.
  4. POSTs `/api/dashboard/login/verify`.
  5. Discards the private key from memory.

## Driver / backend changes

### `wgKeyAuth.ts`

```ts
export function verifyWgSignature(publicKey: Buffer, message: Buffer, signature: Buffer): boolean
```

WireGuard uses Curve25519 for ECDH and BLAKE2s for hashing — but for *signing* we need Ed25519. Curve25519 keys can be converted to Ed25519 (XEdDSA / a Curve25519-to-Ed25519 mapping). We use **libsodium's `crypto_sign_ed25519_pk_to_curve25519` in reverse**: the client converts its X25519 private key to an Ed25519 keypair (deterministic mapping) and signs; the server applies the same conversion to the public key and verifies.

Alternative if XEdDSA conversion turns out to be lossy in practice: have the client perform an HMAC-style proof — `signature = BLAKE2s(privateKey || nonce)`. This isn't a real signature scheme cryptographically, but for a session-binding token it's adequate and uses primitives WireGuard already uses. **Decision pending implementation experiment** — see open questions.

### Session minting

Reuse `useWGSession()`'s cookie infra but with cookie name `wg-user-session`, max-age = 30d. On verify: `session.update({ kind: 'user', userId, clientId, sessionId })`.

## Migration & rollout

- Feature is gated behind `ENABLE_USER_DASHBOARD` from the previous PRD.
- Brute-force rate limit configurable; defaults conservative.

## Verification

### Unit tests

- `wgKeyAuth.test.ts` — known keypair + nonce + signature → verify true; tampered signature → false; tampered nonce → false.
- `login/challenge.test.ts` — issues nonce, marks single-use, expires after 60s.
- `login/verify.test.ts` — wrong public key → 401; right key but client disabled → 403; right key and enabled → 200 + cookie.

### Integration test

- Spin a real client config, derive keys client-side in the test, run the full handshake against the API, assert cookie is set.

### Manual test plan

1. Admin creates user "bob" with role=client and one client "phone".
2. Bob opens `/dashboard/login` on his phone.
3. Bob scans his QR with the in-page scanner → he's logged in within ~2s.
4. Refresh `/dashboard` → still logged in.
5. Admin disables "phone" client → bob's existing session is invalidated (next request → 401).
6. Bob pastes config in textarea (different device) → also works.

## Open questions

- [ ] Curve25519-to-Ed25519 signing: prove with a test that XEdDSA round-trip works against `tweetnacl-js` and Node's `crypto`. If not, fall back to BLAKE2s-HMAC proof. The whole question is "what crypto primitive runs both in browser and Node and binds a key the user already has." Document the chosen path in the implementation diff.
- [ ] Should webcam access be optional? Yes — if the browser denies camera, fall back to "paste config" automatically with a friendly message.

---

## Kimi handoff

**Read before implementing:**
- `[[architecture]]` §5b
- `[[decisions/0003-auth-model]]`
- `[[prds/20-user-features/01-user-dashboard]]`
- `[[prds/00-foundation/03-auth-refactor]]` (cookie setup, principal model)
- `src/server/utils/session.ts`
- `tweetnacl-js` and `libsodium-wrappers` docs
- A reference implementation of XEdDSA in JS to validate the path

**Modify these files:** see `touches:` frontmatter.

**Acceptance tests:**
1. End-to-end login via paste-config in a unit test (no browser).
2. Single-use nonce: replay attempt → 401.
3. Disabled client cannot log in.
4. Rate limiting: 11th request in a minute → 429.

**Self-test plan:**
```bash
pnpm test src/server/utils/wgKeyAuth.test.ts
pnpm test src/server/api/dashboard/login
pnpm dev
# manual: scan QR on a phone connected to the dev origin (use a self-signed cert + ngrok if needed for https)
```
