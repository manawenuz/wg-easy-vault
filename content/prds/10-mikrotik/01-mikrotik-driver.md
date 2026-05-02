---
id: PRD-10-01
title: MikroTik driver — RouterOS API + SSH
status: draft
phase: P1
depends_on:
  - "[[prds/00-foundation/01-backend-abstraction]]"
  - "[[prds/00-foundation/04-data-model-migration]]"
touches:
  - src/server/engines/mikrotik/index.ts (new)
  - src/server/engines/mikrotik/configgen.ts (new)
  - src/server/engines/mikrotik/usage.ts (new)
  - src/server/engines/mikrotik/speedlimit.ts (new)
  - src/server/transports/routeros-api.ts (new)
  - src/server/transports/ssh.ts (new)
  - src/server/engines/registry.ts
  - src/server/api/admin/router/index.get.ts (new)
  - src/server/api/admin/router/index.post.ts (new)
  - src/server/api/admin/router/[id]/index.patch.ts (new)
  - src/server/api/admin/router/[id]/index.delete.ts (new)
  - src/server/api/admin/router/[id]/test.post.ts (new)
  - src/app/pages/admin/routers/index.vue (new)
  - src/app/pages/admin/routers/[id].vue (new)
  - package.json (add: routeros-client, ssh2)
---

# PRD-10-01 — MikroTik driver

> Spec ref: [[architecture#3-vpnengine-driver-interface]], [[architecture#6-mikrotik-provisioning]]

## Why

The flagship feature of the fork: control a MikroTik router as a VPN data plane through the same UI that manages a local WireGuard interface. Validates the [[prds/00-foundation/01-backend-abstraction|VpnEngine abstraction]] and proves the router-agnostic thesis. Auto-bootstrap of a fresh device is handled in [[prds/10-mikrotik/02-mikrotik-autoconfig]] — this PRD assumes the router is already reachable via API.

## User stories

- As an **admin**, I can add a MikroTik router by host/port/credentials and have wg-easy create/manage WireGuard peers on it.
- As an **admin**, I see live peer status (handshake, rx/tx) for MikroTik peers in the same UI as local peers.
- As an **admin**, I can disable a peer on MikroTik from the UI and the router-side rule is removed within seconds.

## Scope

### In

- `MikrotikEngine implements VpnEngine` using `RouterOsApiTransport` for normal ops and `SshTransport` for fallback / future bootstrap.
- Router CRUD admin pages and API.
- Connectivity test endpoint (`POST /api/admin/router/[id]/test`).
- Peer CRUD on MikroTik via `/interface/wireguard/peers`.
- Usage sampling via `/interface/wireguard/peers/print stats`.
- Speed limit application via `/queue/tree` (engine-native).
- Capability flags: `obfuscation: 'wg-obfuscator-sidecar'`, `speedLimit: 'engine-native'`, `livePeerStats: true`, `multiPeerSync: false` (RouterOS API doesn't support batch peer set; we serialize).

### Out

- Bootstrap flow (creating WG interface from scratch on a virgin router) — [[prds/10-mikrotik/02-mikrotik-autoconfig]].
- wg-obfuscator integration — [[prds/10-mikrotik/03-mikrotik-obfuscation]].
- Multi-router federation — [[prds/40-multi-server/01-multi-router-federation]].

## Data model changes

None new. Uses `router` and `wg_interface` already shipped in [[prds/00-foundation/04-data-model-migration]].

`router.credentialsEncrypted` JSON shape for MikroTik:
```json
{
  "apiUser": "wgeasy",
  "apiPassword": "<encrypted>",
  "sshUser": "admin",
  "sshKey": "<base64 private key>",
  "tlsFingerprint": "<sha256>"
}
```

Encrypted at rest using a server-side key (existing `sessionPassword` / a new dedicated encryption key — pick the latter; rotate independently from sessions).

## API changes

| Method | Path | Permission | Body | Returns |
| --- | --- | --- | --- | --- |
| GET | `/api/admin/router` | `router:read` | — | list |
| POST | `/api/admin/router` | `router:admin` | `{name, engineType, transport, host, port, credentials}` | router |
| PATCH | `/api/admin/router/[id]` | `router:admin` | partial | router |
| DELETE | `/api/admin/router/[id]` | `router:admin` | — | `{ok}` (rejects if interfaces still attached) |
| POST | `/api/admin/router/[id]/test` | `router:admin` | — | `{ok, version, peersCount}` |

## UI changes

- `/admin/routers` — list, status indicator (green/red dot from last health check), "Add router" button.
- `/admin/routers/[id]` — detail, edit, "Test connection", and the list of interfaces hosted on this router (wired to existing interface admin via `routerId` filter).
- Existing interface forms gain a **router selector** (defaults to "self"; selection filters available engine types by `router.engineType`).

## Driver / backend changes

### Transports

```ts
// src/server/transports/routeros-api.ts
export class RouterOsApiTransport {
  constructor(private opts: { host, port, user, password, tlsFingerprint? }) {}
  async connect(): Promise<void>;
  async write(path: string, params: Record<string, string | number | boolean>): Promise<RouterOsResponse>;
  async print(path: string, query?: object): Promise<RouterOsRow[]>;
  async close(): Promise<void>;
}
```

Use `routeros-client` from npm (or `node-routeros`). Pin a known-good version. Connection pooling: one persistent connection per router, lazy reconnect with backoff.

```ts
// src/server/transports/ssh.ts
export class SshTransport {
  constructor(private opts: { host, port, user, key | password }) {}
  async exec(cmd: string): Promise<{ stdout, stderr, code }>;
  async close(): Promise<void>;
}
```

Use `ssh2` from npm. Key-based auth preferred; password auth supported.

### MikrotikEngine

Method-by-method:

- **healthCheck**: API `/system/identity print`. Updates `router.lastSeen`.
- **bringUp** / **bringDown**: API `/interface/enable` / `/interface/disable` on the WG iface. (Iface must already exist — bootstrap PRD.)
- **syncInterface**: full reconciliation — list current peers, diff against desired peers, add/update/remove. Idempotent.
- **createPeer**:
  ```
  /interface/wireguard/peers/add interface=<name> public-key=<pk>
    allowed-address=<ipv4>,<ipv6> preshared-key=<psk>
    comment=<client_id>:<name>
  ```
  We embed `client_id` in the comment to map back. Disabled peers use `disabled=yes`.
- **updatePeer**: `/interface/wireguard/peers/set` by `.id` resolved via `find comment="<client_id>:*"`.
- **removePeer**: `/interface/wireguard/peers/remove` by `.id`.
- **enable/disablePeer**: `/interface/wireguard/peers/set disabled=no|yes`.
- **sampleUsage**: `/interface/wireguard/peers/print stats` → parse `rx`, `tx`, `last-handshake`. Public key field maps to `peer.publicKey`.
- **applySpeedLimit**: queue tree. For peer with allowed-address `10.8.0.42/32`:
  ```
  /queue/tree/add name=wg-<client_id>-up parent=global packet-mark=wg-<client_id>-up max-limit=<upKbps>k
  /queue/tree/add name=wg-<client_id>-down parent=global packet-mark=wg-<client_id>-down max-limit=<downKbps>k
  /ip/firewall/mangle/add chain=forward src-address=10.8.0.42 action=mark-packet new-packet-mark=wg-<client_id>-up
  /ip/firewall/mangle/add chain=forward dst-address=10.8.0.42 action=mark-packet new-packet-mark=wg-<client_id>-down
  ```
  `clearSpeedLimit` removes the four entries by name/comment.

### Mapping

Client identity on MikroTik = comment field `<client_id>:<name>`. Never use peer name on the device for identity (users may rename clients in the UI).

## Migration & rollout

- New routers begin as `engine_type=mikrotik`. Existing self router untouched.
- A failed `healthCheck` does not block UI: the router shows red but read paths still work from cached state. Mutations require a successful API connection.
- Connection pool reconnects with exponential backoff up to 5 minutes; after that, status is `unreachable`.

## Verification

### Unit tests

- `mikrotik/configgen.test.ts` — given (iface, peers), produces the correct sequence of API ops.
- `mikrotik/usage.test.ts` — parses sample API responses correctly (use captured fixtures).
- `mikrotik/speedlimit.test.ts` — queue tree commands are correct, `clearSpeedLimit` is idempotent.
- `transports/routeros-api.test.ts` — against a mocked tcp server (or stub the library), connect/reconnect/error paths.

### Integration test

- A docker-compose service running the official MikroTik CHR (Cloud Hosted Router) docker image (or RouterOS-on-qemu via a CI helper). Tests: add router, create iface (assume bootstrap is manual or run a fixture script), add peer, connect from a real WG client, see usage in UI.
- If CHR in CI is too heavy, gate this behind `--integration-mikrotik` and document a manual run step.

### Manual test plan

1. Spin up MikroTik CHR (cloud or local VM, RouterOS 7.x).
2. SSH in, manually create a basic WG interface (`/interface/wireguard add name=wg1 listen-port=51820`), assign IP, accept input on UDP/51820, enable API service, create user `wgeasy` with full perms.
3. In wg-easy: add router (host = CHR IP, API user = wgeasy). Click "Test" → green.
4. Create interface in UI scoped to this router (selecting existing wg1).
5. Add a peer via UI → verify `/interface/wireguard/peers/print` on CHR shows it.
6. Connect from a real WG client → handshake.
7. Set speed limit 1000 / 2000 KB/s → verify queue tree on CHR.
8. Disable peer via UI → CHR `disabled=yes` set; client disconnects.

## Open questions

- [ ] RouterOS 6 vs 7. Decision: target 7.x only. RouterOS 6 WireGuard support is partial / via add-on; not worth the test matrix.
- [ ] TLS for API. RouterOS API supports TLS on port 8729. Default to TLS-on; require fingerprint pinning on first connect (TOFU).

---

## Kimi handoff

**Read before implementing:**
- `[[architecture]]` §3, §6
- `[[glossary]]`
- `[[prds/00-foundation/01-backend-abstraction]]`
- `src/server/engines/types.ts`, `src/server/engines/registry.ts`
- `src/server/engines/wireguard/index.ts` (as the reference implementation)
- `src/server/transports/local-shell.ts` (transport shape reference)
- RouterOS API docs: https://help.mikrotik.com/docs/spaces/ROS/pages/47579160/API
- `routeros-client` README for the chosen npm package

**Modify these files:** see `touches:` frontmatter.

**Acceptance tests:**
1. Unit tests pass against captured RouterOS API fixtures.
2. Against a real CHR: end-to-end peer creation, sampling, and speed limiting works.
3. Speed limit removed when client speed limit is cleared (no orphan queue tree entries).
4. Disabling a peer in the UI takes effect on CHR within 5 seconds.

**Self-test plan:**
```bash
pnpm install
pnpm typecheck
pnpm test src/server/engines/mikrotik src/server/transports
# integration (manual): see test plan
pnpm dev
```
