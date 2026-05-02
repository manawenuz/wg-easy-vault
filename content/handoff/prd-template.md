---
id: PRD-XX-YY
title: <PRD Title>
status: draft
phase: P0
depends_on: []
touches: []
---

# <PRD Title>

> Status: `draft` · Phase: `P0` · Depends on: —

## Why

One paragraph. The user-visible problem or the engineering need. If there is no clear "why", do not write the PRD yet.

## User stories

- As an **\<actor\>**, I can **\<action\>** so that **\<outcome\>**.

Two to five stories max. If you have more, the PRD is too big — split it.

## Scope

### In

- Bullets. Concrete, verifiable.

### Out

- Bullets. Things this PRD deliberately does NOT do, with a one-line reason each.

## Data model changes

```sql
-- new tables / columns / indexes here, drizzle-style preferred but SQL is fine
```

Include migration direction (up + down).

## API changes

| Method | Path | Auth | Body | Returns |
| --- | --- | --- | --- | --- |

## UI changes

- Pages added/changed (file paths under `src/app/pages/`).
- Components added/changed (file paths under `src/app/components/`).
- Pinia store changes.

## Driver / backend changes

- Which `VpnEngine` methods grow new behavior.
- Which existing helpers change (with file:line refs).
- Any new transports.

## Migration & rollout

- Order of operations (schema migrate → backfill → enable code path).
- Backwards compatibility: how old configs / DBs continue to work.
- Feature flag (if any), and exit criteria for removing it.

## Verification

- **Unit tests**: list of test files + what they assert.
- **Integration tests**: end-to-end scenario, including any Docker compose changes.
- **Manual test plan**: numbered steps a human can follow.

## Open questions

- [ ] Anything unresolved. Resolve before moving `status` to `approved`.

---

## Kimi handoff

> This block is the contract between the PRD and the implementer. Keep it in sync with `touches:` frontmatter.

**Read before implementing:**
- `[[architecture]]` — diagrams §X, §Y
- `[[glossary]]`
- Source files (with line ranges):
  - `src/server/utils/WireGuard.ts` (lines 10-338)
  - …

**Modify these files:**
- `src/server/...`
- `src/app/...`
- `src/server/database/repositories/.../schema.ts`
- New: `src/server/engines/<name>/index.ts`

**Do NOT modify:**
- Anything outside the file lists above without re-opening this PRD.

**Acceptance tests** (Kimi must demonstrate these pass):
1. …
2. …

**Self-test plan** (commands Kimi runs locally):
```bash
pnpm test path/to/added/test
docker compose up -d && curl ...
```
