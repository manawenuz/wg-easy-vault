---
title: Orchestration handoff — resume here
type: handoff
last_updated: 2026-05-02
---

# Orchestration handoff

This document exists so a future Claude session (or a human) can pick up the orchestration role without re-deriving everything. **Read this first when resuming.**

## What this fork is

`manawenuz/wg-easy` is a fork of `wg-easy/wg-easy` (Nuxt 4 + Nitro + Drizzle/SQLite WireGuard admin UI). The fork's strategic bet: **turn it into a router-agnostic, multi-tenant VPN control plane** with MikroTik as the flagship non-Linux engine, plus quotas, speed limits, user dashboards, multi-admin, federation, and obfuscation.

Upstream is tracked as the `upstream` git remote (`https://github.com/wg-easy/wg-easy`). Origin is `git@github.com:manawenuz/wg-easy.git`.

## Operating model — orchestrator + Kimi

- **Claude (orchestrator role)** writes PRDs and architecture docs. Does NOT write production code.
- **Kimi (implementer)** consumes one PRD at a time and produces a unified diff + self-test plan. Does NOT write or amend PRDs.
- **The vault is the contract.** `docs/obsidian/` is loaded as an Obsidian vault. Wikilinks resolve by basename.

This separation is a deliberate constraint, not a limitation. PRDs deliberately contain interface signatures, data shapes, and acceptance tests — never implementation code. Kimi gets exactly the context it needs (PRD + [[architecture]] + [[glossary]] + listed source files). See [[handoff/kimi-prompt]] for the operational checklist.

## Vault state as of last session

```
docs/obsidian/
├── README.md                                  ✅
├── architecture.md                            ✅ (10 mermaid diagrams, the spine)
├── roadmap.md                                 ✅ (foundation-first phasing)
├── glossary.md                                ✅
├── decisions/
│   ├── 0001-rust-rewrite.md                   ✅ (decided: no)
│   ├── 0002-backend-abstraction.md            ✅ (driver/strategy pattern)
│   ├── 0003-auth-model.md                     ✅ (split admin/user/token)
│   ├── 0004-obfuscation-strategy.md           ✅ (AmneziaWG + wg-obfuscator/MikroTik)
│   └── 0005-no-ansible.md                     ✅ (decided: no Ansible at runtime)
├── prds/
│   ├── 00-foundation/   (P0, sequential)
│   │   ├── 01-backend-abstraction.md          ✅ status: shipped
│   │   ├── 02-multi-admin-rbac.md             ✅ status: shipped
│   │   ├── 03-auth-refactor.md                ✅ status: shipped
│   │   └── 04-data-model-migration.md         ✅ status: shipped
│   ├── 10-mikrotik/     (P1–P2)
│   │   ├── 01-mikrotik-driver.md              ✅
│   │   ├── 02-mikrotik-autoconfig.md          ✅
│   │   └── 03-mikrotik-obfuscation.md         ✅ (P2)
│   ├── 20-user-features/ (P1)
│   │   ├── 01-user-dashboard.md               ✅ status: approved
│   │   ├── 02-qr-key-login.md                 ✅ status: approved
│   │   ├── 03-bandwidth-quotas.md             ✅ status: approved
│   │   └── 04-speed-limits.md                 ✅ status: approved (feature #14)
│   ├── 30-multi-engine/ (P2)
│   │   ├── 01-amneziawg-promotion.md          ✅
│   │   ├── 02-boringtun-driver.md             ✅
│   │   └── 03-engine-selection-ux.md          ✅
│   ├── 40-multi-server/ (P2–P3)
│   │   ├── 01-multi-router-federation.md      ✅
│   │   ├── 02-admin-router-acl.md             ✅
│   │   └── 03-multi-path-routing.md           ✅ (P3)
│   └── 50-integrations/ (P3)
│       ├── 01-tailscale.md                    ✅ (research-leaning)
│       └── 02-sso.md                          ✅ (research-leaning)
└── handoff/
    ├── prd-template.md                        ✅
    ├── kimi-prompt-template.md                ✅ (rationale + budget)
    ├── kimi-prompt.md                         ✅ (copy-paste)
    └── orchestration-handoff.md               ✅ ← this file
```

**Total**: 28 markdown files. Every PRD has frontmatter, a "Kimi handoff" block, and a `touches:` list.

## Decisions locked with the user

These were chosen explicitly in the planning conversation. **Don't re-litigate without a reason.**

- **Vault location**: `docs/obsidian/` in the repo. (Not `/vault/`, not a sibling repo.)
- **Phasing**: foundation-first. P0 ships before any user-visible feature. P0 is sequential (high coupling).
- **Rust rewrite**: no, captured as [[decisions/0001-rust-rewrite|ADR-0001]].
- **Obfuscation**: AmneziaWG for Linux engines, wg-obfuscator for MikroTik. [[decisions/0004-obfuscation-strategy|ADR-0004]].
- **Ansible**: not in runtime path. [[decisions/0005-no-ansible|ADR-0005]] (decided in same session as this handoff was written).

## Original 14-feature scope from the user

Captured as a checklist for traceability. Each maps to a PRD or ADR.

1. ✅ MikroTik backend → [[prds/10-mikrotik/01-mikrotik-driver]]
2. ✅ Bandwidth quotas (daily/weekly/monthly + auto-disable + auto-reset) → [[prds/20-user-features/03-bandwidth-quotas]]
3. ✅ Rust rewrite (decision memo) → [[decisions/0001-rust-rewrite]] (no)
4. ✅ Multi-server, multi-engine → [[prds/40-multi-server/01-multi-router-federation]] + multi-engine PRDs
5. ✅ BoringTun → [[prds/30-multi-engine/02-boringtun-driver]]
6. ✅ User dashboard with QR/key login → [[prds/20-user-features/01-user-dashboard]] + [[prds/20-user-features/02-qr-key-login]]
7. ✅ Multi-admin → [[prds/00-foundation/02-multi-admin-rbac]]
8. ✅ Tailscale → [[prds/50-integrations/01-tailscale]] (research-leaning)
9. ✅ MikroTik auto-configuration → [[prds/10-mikrotik/02-mikrotik-autoconfig]]
10. ✅ Multi-router federation, admins per router → [[prds/40-multi-server/01-multi-router-federation]] + [[prds/40-multi-server/02-admin-router-acl]]
11. ✅ Multi-path routing → [[prds/40-multi-server/03-multi-path-routing]]
12. ✅ SSO → [[prds/50-integrations/02-sso]] (research-leaning)
13. ✅ Obfuscation → [[decisions/0004-obfuscation-strategy]] + [[prds/30-multi-engine/01-amneziawg-promotion]] + [[prds/10-mikrotik/03-mikrotik-obfuscation]]
14. ✅ Per-client speed limits (KB/s up/down) → [[prds/20-user-features/04-speed-limits]]

## Where to resume

Phase 0 (Foundation) is **fully implemented and shipped**. The codebase now has a pluggable engine architecture, multi-admin RBAC, and a complete foundation schema.

### 1. Dry run with the first P1 PRD (User Features)

The user has requested to prioritize **P1: User Features** over MikroTik.
Suggested next action: hand `[[prds/20-user-features/01-user-dashboard]]` to Kimi using the prompt at [[handoff/kimi-prompt]].

### 2. Sequential P1 implementation

Order for P1 (User Features branch):
1. user-dashboard
2. qr-key-login
3. bandwidth-quotas || speed-limits (can be parallelized)

## Known issues in PRDs not yet sent to Kimi

- **PRD-10-03 (mikrotik-obfuscation) — same `interfaceId integer(...)` bug** in the `wg_obfuscator_config` schema example. Fix before sending (note: P0-04 and 10-03 were updated in vault already).

## Open threads that may come up

- **PRD review pass**: the user may want to tighten scope on individual PRDs before approving. Common edits: cut "out of scope" items further, add explicit `priority` annotations within a phase, split a PRD if `touches:` grows past ~10 files.
- **Obsidian vault config (`.obsidian/`)**: not yet created. Open the folder once in Obsidian to generate it; commit `.obsidian/` if you want shared graph/canvas state, gitignore it if you don't.
- **CI for doc linting**: nice-to-have. Could lint mermaid renders, validate `depends_on:` cycles, check `touches:` paths exist. Captured as backlog, not blocking.
- **PRD assembly script**: implemented at `scripts/assemble-kimi-prompt.sh`. Usage: `./scripts/assemble-kimi-prompt.sh <phase> <index> [-o file.md]`. Examples: `0 1` → backend-abstraction, `10 1` → mikrotik-driver, `20 4` → speed-limits. It reads the PRD's `touches:` frontmatter and the "Read before implementing:" block, pastes everything (PRD + architecture + glossary + read-only files + modify-target files) into a single delimited prompt, and prints stats + a token estimate to stderr. Use this to generate the Kimi prompt instead of pasting by hand.
- **Tailscale and SSO PRDs are research-leaning**: their first deliverable is a 1-page feasibility report, not code. Keep this in mind when approving — they should NOT be sent to Kimi as implementation tasks until the report exists.
- **MikroTik obfuscation least-privilege API group**: open question in [[prds/10-mikrotik/02-mikrotik-autoconfig]]. Default to `group=full` for v1.
- **XEdDSA vs. BLAKE2s-HMAC for QR-key login signing**: open question in [[prds/20-user-features/02-qr-key-login]]. Decide during implementation.

## How to brief a fresh Claude session

If a future Claude session needs to pick this up:

1. Open `/Users/manwe/CascadeProjects/wg-easy-fork/`.
2. Read **this file first** (`docs/obsidian/handoff/orchestration-handoff.md`).
3. Read [[README]], then [[roadmap]], then skim [[architecture]].
4. Check git status — has the vault been committed yet? Has any PRD moved past `draft`?
5. Resume at "Where to resume" above.

Briefing template for the user to use:

> "Continuing the wg-easy fork orchestration work. The Obsidian vault is at `docs/obsidian/`. Read `handoff/orchestration-handoff.md` first, then tell me what the current state is and what you'd suggest as the next step."

## Tone and style notes for the orchestrator role

Carried forward from how this work was actually done; future sessions should match.

- **PRDs are spec, not story.** Concrete data shapes, exact API tables, file paths with line numbers. No marketing tone, no "users will love…", no executive summaries.
- **Engineers are the audience.** Kimi too — and Kimi follows precise instructions better than vibes. Be specific.
- **Out-of-scope sections matter.** Each PRD's `### Out` block prevents scope creep. Be aggressive about putting things there.
- **Decisions get ADRs.** If the user makes a non-obvious choice, capture it in `decisions/` as an ADR with the reasoning, not just the conclusion. The reasoning is what lets future-us re-evaluate when conditions change.
- **Wikilinks resolve by basename.** Use `[[architecture]]`, not `[[architecture.md]]` or `[[../architecture]]`.

## Files referenced from this handoff

- [[README]] · [[architecture]] · [[roadmap]] · [[glossary]]
- [[handoff/kimi-prompt]] · [[handoff/kimi-prompt-template]] · [[handoff/prd-template]]
- [[decisions/0001-rust-rewrite]] · [[decisions/0002-backend-abstraction]] · [[decisions/0003-auth-model]] · [[decisions/0004-obfuscation-strategy]] · [[decisions/0005-no-ansible]]
- All 17 PRDs under `prds/`.
