---
id: PRD-10-02
title: MikroTik auto-configuration — bootstrap from zero
status: shipped
phase: P1
depends_on:
  - "[[prds/10-mikrotik/01-mikrotik-driver]]"
touches:
  - src/server/engines/mikrotik/bootstrap.ts (new)
  - src/server/engines/mikrotik/index.ts
  - src/server/api/admin/router/[id]/bootstrap.post.ts (new)
  - src/app/pages/admin/routers/[id]/bootstrap.vue (new)
  - src/app/components/Routers/BootstrapWizard.vue (new)
  - src/server/engines/mikrotik/bootstrap.test.ts (new)
  - src/i18n/locales/en.json
---

# PRD-10-02 — MikroTik auto-configuration

> Spec ref: [[architecture#6-mikrotik-provisioning]]

## Why

[[prds/10-mikrotik/01-mikrotik-driver]] assumes the router already has a WireGuard interface, an API user, firewall rules, and an IP. In practice operators want to point wg-easy at a fresh RouterOS box and have it set itself up. This PRD is the wizard + idempotent bootstrap routine.

## User stories

- As an **admin** with a fresh CHR or RB, I provide SSH credentials, a WG interface name, a CIDR, and a port; wg-easy creates everything.
- As an **admin**, I can re-run bootstrap on an already-configured router and nothing breaks (idempotent).
- As an **admin**, if bootstrap fails halfway, the wizard tells me exactly which step failed and how to recover.

## Scope

### In

- `bootstrap(router, options)` method on `MikrotikEngine`, idempotent.
- A wizard UI: 4 steps — connectivity → identity → interface config → review/apply.
- API endpoint `POST /api/admin/router/[id]/bootstrap` returning a streaming progress log (Server-Sent Events).
- Steps the bootstrap performs:
  1. SSH connect, `/system/identity print` — confirm RouterOS 7.x.
  2. Generate a WireGuard private key on-router (`/interface/wireguard/add private-key=auto` if needed).
  3. Create the WG interface if missing (`/interface/wireguard/add name=<name> listen-port=<port>`).
  4. Assign IP (`/ip/address/add address=<cidr> interface=<name>`).
  5. Firewall: accept input UDP/<port> from any (rule placed before any drop).
  6. NAT: masquerade on out-interface=<wan> for source=<cidr> (auto-detect WAN by default route, allow override).
  7. Create API user `wgeasy` with strong random password, group `full` (or a custom group if user wants least-privilege — see open questions).
  8. Enable API-SSL service on 8729 with a self-signed cert; capture cert fingerprint.
  9. Encrypt `{apiUser, apiPassword, sshUser, sshKey, tlsFingerprint}` into `router.credentialsEncrypted`.
  10. Switch the router record from "ssh-only" to API-primary.
  11. Final test: API connect, list peers (should be 0), success.

### Out

- One-click router *purchase* / supply-chain integration. Operator brings their own RouterOS box.
- Fully unattended (no SSH creds) bootstrap. Decision: SSH creds are required. We do not attempt RouterBOARD default-user/null-password discovery.
- VLAN / advanced firewall topology — only a flat WAN/LAN is auto-configured.

## Data model changes

None. Bootstrap writes to existing `router` and (optionally) `wg_interface` rows.

## API changes

| Method | Path | Permission | Body | Returns |
| --- | --- | --- | --- | --- |
| POST | `/api/admin/router/[id]/bootstrap` | `router:admin` | `{ifaceName, listenPort, ipv4Cidr, ipv6Cidr?, wanInterface?, sshUser, sshKey | sshPassword}` | SSE stream of `{step, status, detail}` |

The router row must already exist (created via the basic POST `/api/admin/router`). Bootstrap upgrades it.

## UI changes

- A "Bootstrap" button on `/admin/routers/[id]` visible when the router has no `apiPassword` yet (or always, for re-runs).
- 4-step wizard component (`BootstrapWizard.vue`).
- Live log panel that streams the SSE updates with green/red per step.

## Driver / backend changes

### Bootstrap orchestrator

```ts
// src/server/engines/mikrotik/bootstrap.ts
export async function bootstrap(router: Router, opts: BootstrapOptions, emit: (e: ProgressEvent) => void) {
  const ssh = new SshTransport({ host: router.host, port: router.port, ...sshCreds(opts) });
  await ssh.connect();
  emit({ step: 'connect', status: 'ok' });

  await assertRouterOs7(ssh);
  emit({ step: 'identity', status: 'ok', detail: { version } });

  await ensureWireguardInterface(ssh, opts);
  emit(...);

  // ... 11 steps, each in its own helper

  await ssh.close();
  emit({ step: 'done', status: 'ok' });
}
```

Each helper is **idempotent**:
- "Add interface" first checks `/interface/wireguard/print where name=<n>`; if exists, skip.
- "Add IP" checks `/ip/address/print where interface=<n>`.
- "Firewall rule" uses a unique `comment="wg-easy:input-allow"`; check by comment before adding.
- "API user" checks `/user/print where name=wgeasy`; if exists, generate new password and `set password=<new>`.

### Failure handling

Each step returns `{ ok: true } | { ok: false, error, recovery }`. On failure, the SSE stream emits `status=error` with a `recovery` string ("Run `/ip/firewall/filter print` and remove the conflicting rule, then retry"). The wizard surfaces this verbatim.

No automatic rollback — partial state on a router is normal and idempotent re-runs converge. We document this clearly to operators.

### WAN detection

```
/ip/route/print where dst-address="0.0.0.0/0"
```
Take `gateway` and resolve `interface`. If multiple defaults, the wizard asks the operator to pick.

## Migration & rollout

- New feature; no migration impact.
- Behind capability check: bootstrap button only appears for `engine_type=mikrotik` routers.

## Verification

### Unit tests

- Mock `SshTransport`. For each step, fixture the SSH responses and assert the right next command is issued.
- Idempotency tests: run bootstrap with state already in place; assert no destructive ops.

### Integration test

- Against a fresh CHR docker container (or VM): run bootstrap from the wizard end-to-end; assert all 11 steps green; create a peer; client connects.
- Re-run bootstrap on the same CHR; assert all steps green and no errors.

### Manual test plan

1. Boot a fresh RouterOS 7.x VM with default config.
2. Set a known SSH password (RouterOS default has empty `admin`).
3. wg-easy: Add router (host = VM IP, transport = ssh, ssh creds).
4. Click "Bootstrap"; in wizard, set ifaceName=wg-easy, port=51820, ipv4Cidr=10.8.0.1/24, wanInterface=ether1.
5. Watch SSE stream — all green.
6. Confirm in UI: router status green, interface wg-easy listed.
7. Add a peer; connect from a real client; data flows.
8. Re-run bootstrap; all green, no duplicates created.

## Open questions

- [ ] Least-privilege API group: instead of `group=full`, define a custom group with only `api,read,write,policy` or narrower. RouterOS group permissions are coarse; verify what the minimum set is. Default to `full` for v1, document the tradeoff.
- [ ] IPv6: optional; if `ipv6Cidr` provided, also add `/ipv6/address` and an IPv6 firewall rule pair.

---

## Kimi handoff

**Read before implementing:**
- `[[architecture]]` §6
- `[[prds/10-mikrotik/01-mikrotik-driver]]` (especially transport definitions)
- `src/server/engines/mikrotik/index.ts`
- `src/server/transports/ssh.ts`, `src/server/transports/routeros-api.ts`
- RouterOS scripting docs: https://help.mikrotik.com/docs/spaces/ROS/pages/47579160/API

**Modify these files:** see `touches:` frontmatter.

**Acceptance tests:**
1. Bootstrap succeeds on a fresh CHR.
2. Re-running bootstrap on the same router produces zero side effects (idempotent).
3. SSE stream emits one event per step; failure events include a `recovery` string.
4. Step "Firewall rule" detects an existing wg-easy-tagged rule via comment and skips.

**Self-test plan:**
```bash
pnpm test src/server/engines/mikrotik/bootstrap.test.ts
# manual: spin a CHR, run bootstrap from UI
```

## Resolution log (2026-05-02)

- **Shipped**: Idempotent 11-step bootstrap orchestrator via SSH. Progress streamed via SSE (POST fetch + ReadableStream).
- **Wizard UI**: 4-step wizard with live log panel and auto-detection of WAN interface.
- **Security**: Creates random API password, enables API-SSL, captures fingerprint.
- **Tests**: 4 unit tests verifying green-path, ROS < 7 rejection, firewall idempotency, and WAN detection failure.
- **Note**: `src/app/pages/admin/routers/[id].vue` was moved to `[id]/index.vue` to accommodate the bootstrap child route.
