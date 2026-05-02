---
id: ADR-0005
title: Do not use Ansible for MikroTik / external router management
status: decided
date: 2026-05-02
---

# ADR-0005 — No Ansible for runtime router management

## Context

The fork manages MikroTik (and future external routers) as data planes. A reasonable question: should the orchestrator drive these via Ansible (`community.routeros` collection) instead of a direct RouterOS API + SSH transport?

## Decision

**No.** wg-easy talks to MikroTik directly via RouterOS API + SSH ([[prds/10-mikrotik/01-mikrotik-driver]]). Ansible is not in the runtime path.

## Reasoning

1. **Wrong shape.** wg-easy's control loop needs sub-second peer mutations (quota auto-disable, speed-limit changes). `ansible-playbook` startup alone is 200–1000 ms of Python import + inventory parse, per invocation. Persistent RouterOS API connections respond in tens of ms. The latency gap is product-defining, not an optimization detail.

2. **Two layers of indirection.** wg-easy → Ansible → RouterOS API doubles the failure modes and the debugging surface. A failed peer add now means parsing both Ansible output and the underlying RouterOS error.

3. **Image bloat and CVE surface.** Adds Python + `ansible-core` + `community.routeros` + `librouteros` to the Docker image (~150 MB). For functionality already provided by a Node RouterOS client.

4. **Idempotency is our problem either way.** Ansible's idempotency works inside a single playbook run. Our state-converge logic (`syncInterface` diffs against current peers) needs to exist regardless. Ansible would just be a thicker wrapper around the same idea.

5. **Inventory mismatch.** Source of truth is the SQLite `router` + `wg_interface` tables. Ansible expects YAML/INI inventory. Generating inventory on every operation is the same cost as calling the API directly, minus the indirection.

## When this might be revisited

- **Multi-vendor expansion** beyond MikroTik (Cisco, Juniper, Fortinet). At that point, Ansible (or NETCONF/YANG) starts to pay back because the marginal cost of vendor #4 inside Ansible is lower than maintaining four bespoke drivers. Until then, MikroTik-only doesn't justify the runtime weight.
- **If the bootstrap step ([[prds/10-mikrotik/02-mikrotik-autoconfig]]) grows past ~20 idempotent steps**, switch *only the bootstrap* to a generated playbook the operator can run separately. Keep the steady-state path direct.

## What we do instead

- Direct RouterOS API + SSH transports per [[prds/10-mikrotik/01-mikrotik-driver]].
- Persistent connection pooling, per-router.
- Idempotent operations modeled inside the driver, not delegated.

## What's worth doing later (P3+, optional)

- **One-way Ansible export**: a read-only "dump router X's desired state as an Ansible inventory + playbook" feature. Lets ops teams that already run Ansible integrate wg-easy state into existing workflows without us taking on Ansible as a runtime dependency. Pure export, not a control path.

## Consequences

- We commit to writing and maintaining MikroTik integration logic in TypeScript inside `src/server/engines/mikrotik/`.
- Documentation should not imply Ansible is on the roadmap. If a contributor asks, point here.
- The export feature above is captured as a possible P3 follow-up; not a blocker.

## Alternatives considered

- **Ansible as the only path** — rejected for latency, indirection, image weight.
- **Hybrid: Ansible for bootstrap, direct API for steady state** — rejected. Bootstrap is 11 idempotent SSH commands; rewriting them as Ansible tasks is more code, not less, and binds us to the Ansible runtime forever once any operator wires CI on top of it.
- **NETCONF / YANG** — premature; MikroTik's NETCONF support is partial. Revisit with multi-vendor.
