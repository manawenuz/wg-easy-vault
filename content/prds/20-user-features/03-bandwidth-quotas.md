---
id: PRD-20-03
title: Bandwidth quotas — daily / weekly / monthly with auto-disable
status: shipped
phase: P1
depends_on:
  - "[[prds/00-foundation/01-backend-abstraction]]"
  - "[[prds/00-foundation/04-data-model-migration]]"
touches:
  - src/server/scheduler/index.ts (new)
  - src/server/scheduler/usagePoller.ts (new)
  - src/server/scheduler/quotaEvaluator.ts (new)
  - src/server/scheduler/periodResetter.ts (new)
  - src/server/scheduler/usageRollup.ts (new)
  - src/server/services/quotaService.ts (new)
  - src/server/api/admin/clients/[id]/quota.get.ts (new)
  - src/server/api/admin/clients/[id]/quota.put.ts (new)
  - src/server/api/admin/clients/[id]/quota.delete.ts (new)
  - src/app/components/Clients/QuotaForm.vue (new)
  - src/app/components/Clients/QuotaProgress.vue (new)
  - src/app/components/Clients/QuotaProgressBar.vue (new)
  - src/app/pages/clients/[id].vue
---

# PRD-20-03 — Bandwidth quotas

> Spec ref: [[architecture#7-quota-enforcement-loop]]

## Why
...
### Manual test plan
...

## Resolution log (2026-05-02)

- **Backend Workers**: Implemented four scheduler workers (UsagePoller, QuotaEvaluator, PeriodResetter, UsageRollup) started on app init.
- **Counter Reset Detection**: UsagePoller correctly handles `delta < 0` scenarios by treating the new sample as absolute fresh usage.
- **Admin APIs**: Scoped admin endpoints for quota CRUD implemented under `/api/admin/clients/[id]/quota`.
- **UI Integration**: Added `QuotaForm`, `QuotaProgress`, and `QuotaProgressBar` components.
- **Follow-up Fix**: Merged the Quota form directly into the existing `src/app/pages/clients/[id].vue` instead of creating a separate admin page, ensuring all client settings remain in one place.
- **Tests**: 8 unit tests for admin quota APIs verified.


Many deployments need volume caps per user — e.g., 50 GB/month for a basic plan, hard limit. Today there's nothing. This PRD adds **periodic volume quotas** with **auto-disable on exceed** and **automatic reset/re-enable at period end**.

## User stories

- As an **admin**, I can set a quota on a client: 10 GB/day, 50 GB/week, or 200 GB/month.
- As an **admin**, when a client exceeds, the engine disables their peer; an audit log entry records why.
- As an **admin**, at period end the client is auto re-enabled and the counter resets.
- As a **user**, my dashboard shows current usage vs. quota and a countdown to reset.
- As an **admin**, I can manually reset / override (re-enable mid-period) with the action recorded.

## Scope

### In

- One quota per client (table from [[prds/00-foundation/04-data-model-migration]]).
- Period types: `daily` (UTC midnight rollover), `weekly` (Monday 00:00 UTC), `monthly` (1st of month, 00:00 UTC).
- Scheduler workers (4):
  - **Usage poller**: every 60s (configurable). For each enabled interface, calls `engine.sampleUsage()`, diffs against the previous sample, writes to `usage_sample`, updates `quota.used_bytes`.
  - **Quota evaluator**: triggered after each poll (in-process). For each quota where `used_bytes >= limit_bytes` and `auto_disable=true` and `disabled_by_quota_at IS NULL`: call `engine.disablePeer()`, set `disabled_by_quota_at = now()`, audit log.
  - **Period resetter**: cron-like (runs on minute boundaries; checks each quota's `period_end < now`). On reset: zero `used_bytes`, advance `period_start`/`period_end`, if `disabled_by_quota_at` set then re-enable peer (only if no manual disable still in effect).
  - **Usage rollup**: hourly. Compress raw `usage_sample` rows older than 24h into hourly aggregates; raw rows older than 7d are deleted.
- Admin UI: quota form on the client edit page (limit + period + auto_disable toggle).
- User dashboard: progress bar with bytes used / limit and "resets in 4d 3h".

### Out

- Multiple quotas per client (e.g., 5GB/day AND 50GB/month). Use one period; if needed, add second tier later.
- Per-engine quota rate-shaping (slow-down on approaching). Speed limits are a separate PRD.
- Different rollover anchors (e.g., billing cycle starting on the 15th). Could be added with a `period_anchor` field; out for v1.
- Notifications when approaching quota. Captured as follow-up.

## Data model changes

`quota` table from [[prds/00-foundation/04-data-model-migration]]. No further schema changes.

## API changes

| Method | Path | Permission | Body | Returns |
| --- | --- | --- | --- | --- |
| GET | `/api/admin/clients/[id]/quota` | `client:read` | — | quota or null |
| PUT | `/api/admin/clients/[id]/quota` | `client:write` | `{ limitBytes, period, autoDisable }` | quota |
| DELETE | `/api/admin/clients/[id]/quota` | `client:write` | — | `{ ok }` (also re-enables if disabled by quota) |

Existing GET `/api/dashboard/clients` already includes `quota` per [[prds/20-user-features/01-user-dashboard]].

## UI changes

- `Clients/QuotaForm.vue` — under client edit, three inputs: limit (with unit picker MB/GB/TB), period dropdown, auto-disable toggle. "Reset now" button.
- `Dashboard/QuotaProgress.vue` — bar with used/limit bytes, ETA-to-reset, color shifts at 80% / 100%.

## Driver / backend changes

### Usage poller

```ts
// scheduler/usagePoller.ts
export async function pollUsage(): Promise<void> {
  for (const iface of await db.wgInterface.findEnabled()) {
    const engine = getEngine(iface.engineType);
    const samples = await engine.sampleUsage(iface);
    for (const s of samples) {
      const client = await db.client.findByPublicKey(s.publicKey);
      if (!client) continue;
      const last = await db.usageSample.lastForClient(client.id);
      const dRx = s.rxBytes - (last?.rxBytes ?? 0n);
      const dTx = s.txBytes - (last?.txBytes ?? 0n);
      // Counter reset detection: if delta < 0, take the new value as fresh
      const rxDelta = dRx >= 0 ? dRx : s.rxBytes;
      const txDelta = dTx >= 0 ? dTx : s.txBytes;
      await db.usageSample.insert({ clientId: client.id, rxBytes: s.rxBytes, txBytes: s.txBytes, ts: now });
      const quota = await db.quota.find(client.id);
      if (quota) await db.quota.addUsage(client.id, rxDelta + txDelta);
    }
  }
  await evaluateQuotas();
}
```

Counter reset detection (peer reconfigured / interface restarted): if `delta < 0`, treat `s.rxBytes` as new traffic.

### Quota evaluator

Runs in-process after `pollUsage()`. SQL: `SELECT clientId FROM quota WHERE used_bytes >= limit_bytes AND auto_disable AND disabled_by_quota_at IS NULL`. For each, call `engine.disablePeer`, set `disabled_by_quota_at`, audit.

### Period resetter

Cron on `*/1 * * * *` (every minute). Query: quotas where `period_end <= now()`. For each:
1. Compute new `period_start`/`period_end` based on `period`. Anchor: UTC midnight (daily), Monday 00:00 UTC (weekly), 1st of month 00:00 UTC (monthly).
2. Zero `used_bytes`.
3. If `disabled_by_quota_at IS NOT NULL`: check audit log for any subsequent manual disable; if none, re-enable via `engine.enablePeer`, audit, clear `disabled_by_quota_at`.

### Manual override

`DELETE /api/admin/clients/[id]/quota` removes the quota row; if `disabled_by_quota_at` was set, re-enable peer. `POST /api/admin/clients/[id]/enable` (existing) clears `disabled_by_quota_at` — meaning a manual re-enable wins until next exceed.

### Usage rollup

```ts
// scheduler/usageRollup.ts
// every hour: rows older than 24h get aggregated into hourly buckets in
// `usage_sample` itself (insert one bucket row, delete the raw rows).
// rows older than 7d: delete (the hourly bucket suffices for charts).
```

Mark rolled-up rows with a `bucket_size` column? Simpler: keep one table, the rollup just inserts a single row representing the bucket and deletes the raw rows, so all rows are "samples" and the chart is timestamp-aware.

## Migration & rollout

- New scheduler service starts with the app. Single instance per process. (Multi-instance orchestrator considerations are deferred to [[prds/40-multi-server/01-multi-router-federation]].)
- No flag — quotas are opt-in per client (no `quota` row = no enforcement).
- Existing clients are unaffected.

## Verification

### Unit tests

- `usagePoller.test.ts` — counter reset detected; deltas correct; quota incremented.
- `quotaEvaluator.test.ts` — disables when over; doesn't re-disable if already disabled; respects `auto_disable=false`.
- `periodResetter.test.ts` — daily/weekly/monthly rollover times; re-enable only if not manually disabled.
- `usageRollup.test.ts` — hourly aggregation correctness; 7d cleanup.

### Integration test

- Seed a client with quota=1MB, daily. Generate 1MB of usage_sample rows. Assert peer disabled, audit log written. Advance clock to next UTC midnight. Run resetter. Assert peer re-enabled, used_bytes=0.

### Manual test plan

1. Set quota of 100 MB/daily on a test client.
2. Use the VPN to download a 200 MB file.
3. Within ~1 poll cycle, the client is disabled.
4. UI shows "Disabled by quota" badge.
5. Audit log shows the auto-disable.
6. Manually click "Reset now" → quota cleared, peer re-enabled.
7. Set quota again. Wait until UTC midnight (or set clock); verify auto-reset and re-enable.

## Open questions

- [ ] Counter precision: WireGuard kernel counters are u64 bytes but exposed as decimal in `wg show dump`. We store as numeric. MikroTik returns counters as well. Both fit in JS `number` (2^53 bytes ≈ 8 PB). OK.
- [ ] Should daily reset honor a per-tenant timezone? v1: UTC only. Document; revisit.

---

## Kimi handoff

**Read before implementing:**
- `[[architecture]]` §7
- `[[prds/00-foundation/01-backend-abstraction]]` (`sampleUsage`, `disablePeer`, `enablePeer`)
- `[[prds/00-foundation/04-data-model-migration]]` (`quota`, `usage_sample`, `audit_log`)
- `src/server/engines/types.ts`
- An existing service file as a pattern: `src/server/services/...`

**Modify these files:** see `touches:` frontmatter.

**Acceptance tests:**
1. Quota auto-disable triggers within one poll cycle of exceed.
2. Period reset re-enables iff no manual disable in effect.
3. Counter reset detection works.
4. Hourly rollup compresses without changing chart correctness (compare chart series before/after rollup against expected).

**Self-test plan:**
```bash
pnpm test src/server/scheduler
pnpm test src/server/services/quotaService.test.ts
pnpm dev
# manual: see test plan
```
