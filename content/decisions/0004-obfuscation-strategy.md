---
id: ADR-0004
title: Obfuscation — AmneziaWG everywhere, wg-obfuscator for MikroTik
status: decided
date: 2026-05-02
---

# ADR-0004 — Obfuscation strategy

## Context

Some users sit behind DPI that fingerprints WireGuard handshakes. We want a credible obfuscation story without inventing a protocol. Two existing options:

- **AmneziaWG** — a fork of WireGuard with packet-shape obfuscation (jC, jMin/Max junk packets, s1-s4 magic-byte stuffing, h1-h4 alternative header types, i1-i5 init-packet variations). Same wire-compatible peer model as WireGuard. Already in upstream wg-easy as a build flag.
- **wg-obfuscator** — a separate UDP-level proxy that wraps WireGuard packets in an opaque stream. Has a published RouterOS integration ([reference](https://github.com/ClusterM/wg-obfuscator/blob/master/docs/MIKROTIK.md)).

## Decision

- **Linux engines** (kernel WireGuard, BoringTun): use **AmneziaWG** when obfuscation is requested. Promote AmneziaWG from a build-time flag to a runtime engine choice ([[prds/30-multi-engine/01-amneziawg-promotion]]).
- **MikroTik engine**: use **wg-obfuscator** wrapping a normal WireGuard interface ([[prds/10-mikrotik/03-mikrotik-obfuscation]]). RouterOS does not have AmneziaWG.

## Reasoning

1. **AmneziaWG is the right primitive when we control the kernel.** It's same-wire as WireGuard, has good params for shape randomization, and its schema fields are *already in our DB* (jC, jMin/Max, s1-s4, h1-h4, i1-i5 on `wg_interface` and `client`). We are not reinventing modeling, just promoting from compile-time switch to runtime engine. Both client and server must run AmneziaWG-aware code; AmneziaWG-aware Android/iOS/desktop clients exist.

2. **MikroTik can't run AmneziaWG.** RouterOS has its own WireGuard implementation; we can't load a kernel module. wg-obfuscator runs as a userspace UDP proxy on RouterOS via the published mikrotik integration. So obfuscation on MikroTik = WireGuard interface + wg-obfuscator wrapper.

3. **Don't build a generic obfuscation framework.** A plugin layer that abstracts "any obfuscator" is the wrong shape — there are two real choices and they land in different places (engine config vs sidecar config). Concrete integrations are simpler than an abstraction we'd grow into.

4. **Capability flags express the difference.** `engine.capabilities.obfuscation` is one of `none | amneziawg-params | wg-obfuscator-sidecar`. The UI shows the right form.

## Shape

- AmneziaWG params live on `wg_interface` and (where they should differ) on `client`. Already there.
- wg-obfuscator config lives in a new table `wg_obfuscator_config(interface_id PK, listen_port, key, dummy_padding, ...)`. Only populated when `engine_type = mikrotik` AND obfuscation enabled.
- Client config generation:
  - AmneziaWG → standard `awg-quick` config with the obfuscation block.
  - wg-obfuscator → standard WireGuard `[Interface]` + a server endpoint that points at the local obfuscator port; users run a small wg-obfuscator client locally. The config download includes setup instructions.

## Consequences

- We commit to two obfuscation surfaces, not one unified one. Future obfuscation methods (e.g., Cloak, Shadowsocks-over-WG) would each be their own integration, gated by a new capability flag.
- The user-facing language is "obfuscation: on/off" plus an engine-dependent detail panel — users don't have to know "AmneziaWG vs. wg-obfuscator" unless they want to.
- Mobile/desktop clients **must support the chosen obfuscation** for AmneziaWG; wg-obfuscator requires the user to install a small companion. The dashboard shows the right download links per engine.

## Alternatives considered

- **AmneziaWG only, drop MikroTik obfuscation** — rejected. MikroTik users are the flagship customer; "no obfuscation on MikroTik" is a regression vs. published mikrotik integration.
- **Build a custom obfuscator** — rejected. Cryptographically and operationally expensive, no benefit over wg-obfuscator/AmneziaWG.
- **One unified plugin layer** — rejected. Two integrations, two surfaces. Premature abstraction; see [[decisions/0002-backend-abstraction]].

## Related PRDs

- [[prds/30-multi-engine/01-amneziawg-promotion]]
- [[prds/10-mikrotik/03-mikrotik-obfuscation]]
