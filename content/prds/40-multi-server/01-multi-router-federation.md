---
id: PRD-40-01
title: Multi-router federation — orchestrator + agents
status: draft
phase: P2
depends_on:
  - "[[prds/00-foundation/04-data-model-migration]]"
  - "[[prds/00-foundation/02-multi-admin-rbac]]"
touches:
  - src/server/agent/index.ts (new)
  - src/server/agent/protocol.ts (new)
  - src/server/agent/dispatcher.ts (new)
  - src/server/transports/agent.ts (new)
  - src/server/api/agent/connect.ts (new)
  - src/server/api/agent/heartbeat.post.ts (new)
  - src/server/cli/agent.ts (new)
  - Dockerfile (add agent mode entrypoint)
---

# PRD-40-01 — Multi-router federation

> Spec ref: [[architecture#8-multi-router-federation]]

## Why

Beyond "one wg-easy with multiple drivers", real deployments have multiple boxes — a flagship in one region, a satellite in another, each running a local WG. Today they're managed independently. Federation makes one **orchestrator** the canonical UI/DB and turns the others into **agents** that take commands over an outbound mTLS connection. This unlocks per-tenant fleets, multi-region routing, and ([[prds/40-multi-server/03-multi-path-routing|later]]) per-traffic exit selection.

## User stories

- As an **operator**, I deploy a wg-easy agent on a remote VM with one env var (`ORCHESTRATOR_URL`) + a join token; it shows up in the orchestrator UI within seconds.
- As an **admin** in the orchestrator, I see all agents and their interfaces in one list, do peer ops in any of them.
- As an **operator**, agents survive orchestrator restarts and reconnect; they survive their own restarts and resync.

## Scope

### In

- Two run modes in one binary: `mode=orchestrator` (default — current behavior + agent dispatch) and `mode=agent` (no UI, no DB beyond a tiny local cache).
- Agents speak to the orchestrator over **outbound mTLS WebSocket** to `wss://orchestrator/api/agent/connect`. Outbound only — agents don't need inbound connectivity.
- A simple JSON RPC protocol over the WS: `{ id, type: 'request'|'response'|'event', method, params|result|error }`.
- `transport=agent` in the `router` table — when an interface's router has this transport, all engine ops dispatch via the WS.
- Agents implement `VpnEngine` locally for their own engine type and proxy commands through.
- Heartbeat every 30s; orchestrator marks `router.lastSeen`. UI shows a red dot if heartbeat is stale.
- Join flow: operator runs `wg-easy agent --join <orchestrator-url> --token <one-time>`; the orchestrator mints a long-lived agent cert; agent stores it locally and reconnects with it.

### Out

- Multi-orchestrator HA. One orchestrator per fleet for now.
- Cross-agent peer state mirroring. The DB stays canonical on the orchestrator.
- Replacing the local-shell transport for the orchestrator's own router. The orchestrator can still drive a local WG directly without an agent.

## Data model changes

Add to `router` (extend [[prds/00-foundation/04-data-model-migration]]):
- `agentCertFingerprint TEXT` — to pin the agent's cert.
- `agentLastConnectAt INTEGER` — separate from `lastSeen` which is heartbeat-based.

Add `agent_join_token` table:
```ts
{ id, token_hash, created_by_user_id, expires_at, used_at, used_by_router_id }
```
One-time-use, 15-minute TTL.

## API changes

| Method | Path | Notes |
| --- | --- | --- |
| GET | `/api/agent/connect` | WS upgrade, mTLS required (cert pinned) |
| POST | `/api/agent/heartbeat` | optional fallback for environments without WS |
| POST | `/api/admin/router/[id]/join-token` | mints a one-time join token (admin) |

## Driver / backend changes

### Protocol

```ts
// src/server/agent/protocol.ts
type Request = { id: string; type: 'request'; method: AgentMethod; params: unknown };
type Response = { id: string; type: 'response'; result?: unknown; error?: { code, message } };
type Event = { type: 'event'; name: string; payload: unknown };

type AgentMethod =
  | 'engine.healthCheck'
  | 'engine.bringUp' | 'engine.bringDown'
  | 'engine.syncInterface'
  | 'engine.createPeer' | 'engine.updatePeer' | 'engine.removePeer'
  | 'engine.enablePeer' | 'engine.disablePeer'
  | 'engine.sampleUsage'
  | 'engine.applySpeedLimit' | 'engine.clearSpeedLimit';
```

Methods 1:1 with `VpnEngine`. The agent transport class on the orchestrator implements `VpnEngine` by serializing each call into a request and awaiting the response.

### Agent worker

```ts
// src/server/agent/index.ts
async function runAgent(opts: { orchestrator, cert, key }) {
  const ws = await connectWithBackoff(opts);
  ws.on('message', async (msg) => {
    const req = parse(msg);
    const engine = getEngine(localEngineType());
    try {
      const result = await dispatch(req.method, req.params, engine, localIfaceFor(req.params));
      ws.send(success(req.id, result));
    } catch (e) { ws.send(error(req.id, e)); }
  });
  setInterval(() => ws.send(heartbeat()), 30_000);
}
```

### Local cache

Agent has a small SQLite (separate from orchestrator's) with: its own router id, last-known interface configs, last-known peer set. Used to converge after a disconnect — orchestrator pushes a `syncInterface` on every reconnect, so the agent doesn't strictly need persistence, but having it lets the agent self-heal if its data plane drifted.

### CLI

```bash
wg-easy agent --join https://orchestrator.example.com --token <t>
wg-easy agent  # subsequent runs, uses stored cert
```

## Migration & rollout

- Existing single-instance deployments unaffected (no agents, no `transport=agent` routers).
- Add a feature flag `ENABLE_FEDERATION` defaulting to true; can be disabled to lock down an instance to single-router operation.

## Verification

### Unit tests

- `protocol.test.ts` — request/response round-trip, error mapping.
- `dispatcher.test.ts` — orchestrator-side `VpnEngine` proxy serializes calls correctly.

### Integration test

- Two-container compose: orchestrator + agent. Join with a token. Create an interface on the agent's router. Add peers. Real WG client connects through the agent. Restart orchestrator; agent reconnects. Restart agent; orchestrator sees red dot, then green; peer config converges.

### Manual test plan

1. Bring up orchestrator on host A.
2. Mint join token in UI.
3. On host B: `docker run -e MODE=agent -e ORCHESTRATOR_URL=... -e JOIN_TOKEN=... wg-easy`.
4. Within 10s, orchestrator UI shows a new router (host B).
5. Create iface on it, add peer, real client on host B connects.
6. Kill agent container; orchestrator shows red dot. Restart; green.
7. Kill orchestrator; agent retries with backoff. Restart orchestrator; agent reconnects.

## Open questions

- [ ] mTLS bootstrap: the orchestrator itself needs a TLS cert the agent trusts. Use Let's Encrypt with HTTP-01 if the orchestrator is on a public domain; document a self-signed fallback (operator copies fingerprint to agent at join time).

---

## Kimi handoff

**Read before implementing:**
- `[[architecture]]` §8
- `[[prds/00-foundation/01-backend-abstraction]]`
- `[[prds/00-foundation/02-multi-admin-rbac]]` (audit log for agent commands)
- `src/server/engines/registry.ts`
- `ws` and `node:tls` docs

**Modify these files:** see `touches:` frontmatter.

**Acceptance tests:**
1. Two-container compose passes the integration test.
2. Backoff and reconnect loops are bounded; no thundering herd on orchestrator restart.
3. Join token is single-use.

**Self-test plan:**
```bash
pnpm test src/server/agent
docker compose -f docker-compose.federation.yml up
# manual: see test plan
```
