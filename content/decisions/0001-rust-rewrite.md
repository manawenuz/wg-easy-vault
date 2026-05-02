---
id: ADR-0001
title: Do not rewrite wg-easy in Rust (now)
status: decided
date: 2026-05-02
deciders: [manawenuz]
---

# ADR-0001 — Do not rewrite wg-easy in Rust (now)

## Context

The fork's roadmap is large: MikroTik driver, multi-engine, multi-tenant, multi-router federation, quotas, speed limits, dashboards, obfuscation. A reasonable question: should we rewrite the codebase in Rust before piling this on?

The argument for Rust is roughly:
- WireGuard's reference implementation and BoringTun are Rust.
- Quota accounting and per-peer rate-limiting touch hot paths.
- A long-running control plane benefits from memory safety and predictable latency.

## Decision

**No rewrite.** Stay on Nuxt 4 / Nitro / TypeScript / Drizzle / SQLite.

## Reasoning

1. **The bottleneck is integration breadth, not language performance.** Every feature on the roadmap is an integration: RouterOS API, SSH, kernel WG, BoringTun process, AmneziaWG, Tailscale, OIDC. None of these is CPU-bound at the control-plane layer. We're shuffling small JSON payloads, not crunching packets.

2. **The data plane is already in C/Rust.** WireGuard kernel module = C. BoringTun = Rust. AmneziaWG = Go. RouterOS = closed-source on a router. The control plane's job is to *talk to* fast things, not to be one.

3. **Ecosystem reuse is huge.** Nuxt gives us SSR, the admin UI, the routing layer, Pinia, Tailwind, the auth plumbing, mkdocs, Drizzle migrations — all already wired up in upstream wg-easy. Rewriting in Rust means rebuilding the front end (Yew/Leptos are not at parity with Vue/Nuxt for this kind of dashboard) or splitting into a Rust backend + JS frontend, which doubles the surface area.

4. **Upstream merging.** As long as we stay on the upstream stack, we can pull MikroTik-agnostic improvements from `wg-easy/wg-easy` via `git fetch upstream && git merge`. A Rust rewrite forks us irreversibly.

5. **TypeScript is fine for what we do.** The hot paths we *do* care about (usage sampling loop, quota accumulation) touch SQLite rows in batches of <1000 per minute. Even a sloppy TS implementation has 100x headroom.

6. **Talent and velocity.** The implementer (Kimi) is more productive in TS than in Rust at codebase scale, and the orchestrator (Claude) reasons about TS/Vue/Nuxt with less ambiguity. Rewriting would slow every subsequent PRD.

## When this decision should be revisited

- If the usage-sampling loop becomes a measured (not theorized) bottleneck above ~50 routers × ~1000 peers each.
- If we want to ship the orchestrator as a single static binary for restricted deployments (no Node runtime).
- If the agent (in [[prds/40-multi-server/01-multi-router-federation|federation]]) needs to run on tiny edge boxes where Node is too heavy. Even then, **rewrite the agent only**, not the orchestrator. The agent is a small surface (a few hundred lines) — that's the right wedge for Rust if we ever take it.

## What we do instead

- Profile-driven optimization where (and only where) the TS path is actually hot.
- Native modules for any specific piece that genuinely needs them (e.g., wireguard-tools bindings via `node-ffi-napi` — if exec'ing `wg` ever shows up in profiles).
- Keep the engine driver interface narrow ([[architecture#3-vpnengine-driver-interface]]) so a future Rust agent can implement it without touching the orchestrator.

## Consequences

- We accept Node.js runtime as a deployment requirement.
- We accept SQLite's single-writer model. If we outgrow it, switch to Postgres (Drizzle supports both) — that's a data-store change, not a rewrite.
- Documentation should never imply Rust is on the roadmap. If a contributor asks, point them here.
