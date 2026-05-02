---
title: Glossary
type: reference
---

# Glossary

Precise terminology — used consistently across [[architecture]] and every PRD. When in doubt, link to this page.

## Core concepts

- **Control plane** — the wg-easy app itself: UI, API, DB, scheduler. Does not forward packets.
- **Data plane** — the thing that actually moves packets: a Linux kernel WireGuard interface, a MikroTik router, a BoringTun userspace process, etc.
- **Engine** — a *kind* of VPN data plane. Concrete engines: `wireguard`, `amneziawg`, `boringtun`, `mikrotik`. Stored as an enum on `wg_interface.engine_type`.
- **Driver** — the control-plane code that speaks to a specific engine. One driver per engine. Implements the [[prds/00-foundation/01-backend-abstraction|VpnEngine]] interface.
- **Transport** — *how* a driver talks to its engine. Examples: `local-shell` (current behavior), `ssh`, `routeros-api`. Drivers may use multiple transports (the MikroTik driver uses `routeros-api` for normal ops and `ssh` for bootstrap).
- **Router** — a managed data-plane host. A row in the `router` table. Has connection details (host, port, credentials), an engine type, and zero or more interfaces. The local box itself is `router_id = 0` (the "self" router).
- **Interface** — a WireGuard interface (or its non-Linux equivalent on MikroTik). Belongs to exactly one router. Existing `wg_interface` table.
- **Peer / Client** — same thing: a WireGuard peer entry. We use **client** in user-facing UI ("VPN client") and **peer** in protocol-level prose. Existing `client` table.

## Auth & tenancy

- **Admin** — a user with a non-`client` role who can manage interfaces, routers, or other users. Multi-admin (P0) introduces sub-roles: `superadmin`, `admin`, `operator`, `viewer`. See [[prds/00-foundation/02-multi-admin-rbac|RBAC PRD]].
- **End user** (or just **user**) — a person who has VPN clients but does not log into the admin UI. They may log into the **user dashboard** ([[prds/20-user-features/01-user-dashboard]]).
- **Tenant** — currently 1:1 with router scope. An admin scoped via `admin_router_acl` to routers `{A, B}` is effectively the tenant for those routers. Future SSO/multi-org work may formalize tenants further.
- **Session** — encrypted cookie issued by `useWGSession()`; admin sessions and user-dashboard sessions are the **same mechanism, different role**.
- **API token** — long-lived bearer token for programmatic access. Reuses the existing Basic Auth path in `session.ts` (rather than adding a third auth scheme).

## Quotas & limits

- **Quota** — a *volume* allowance over a *period*: e.g., 50 GB/month. Enforced by disabling the client when exceeded; auto-reset at period end. See [[prds/20-user-features/03-bandwidth-quotas]].
- **Speed limit** — a *rate* cap (KB/s) applied to a client's traffic, up and/or down. Enforced at the data plane (tc/HTB on Linux, queue tree on MikroTik). See [[prds/20-user-features/04-speed-limits]].
- **Usage sample** — a single (client, timestamp, rx_bytes, tx_bytes) row. The quota engine accumulates samples to compute period totals.

## Routing & multi-server

- **Federation** — running >1 wg-easy node and orchestrating them from a single UI. The orchestrator node holds the canonical DB; agent nodes execute. See [[prds/40-multi-server/01-multi-router-federation]].
- **Exit node** — a router that egresses traffic to the public internet. A multi-path setup has multiple exit nodes; clients (or subnets within a client's allowed-IPs) are policy-routed to one.
- **Multi-path** — selecting a different exit node per IP/subnet/client. Implemented via per-router policy routing tables. See [[prds/40-multi-server/03-multi-path-routing]].

## Obfuscation

- **AmneziaWG** — a WireGuard fork that adds packet-shape obfuscation parameters (jC, jMin, jMax, s1-s4, h1-h4, i1-i5). Already a build-flag option upstream; we promote it to a runtime engine. See [[prds/30-multi-engine/01-amneziawg-promotion]].
- **wg-obfuscator** — a separate proxy that wraps WireGuard traffic in a custom obfuscated transport. We integrate it specifically on the MikroTik backend. See [[prds/10-mikrotik/03-mikrotik-obfuscation]].

## Process

- **PRD** — Product Requirements Doc, but in this vault it's closer to an Engineering Design Doc: explicit data-model and API changes, not vague user stories.
- **Kimi** — the implementation model (256K context). Receives one PRD + [[architecture]] + relevant source files per session. See [[handoff/kimi-prompt-template]].
- **Handoff block** — the section at the end of every PRD listing exact files to read, files to modify, and acceptance tests.
