---
title: wg-easy fork — Obsidian vault
type: index
---

# wg-easy fork — Documentation Vault

This is the design vault for the **manawenuz/wg-easy** fork. It contains PRDs and architecture documentation for turning wg-easy from a single-box WireGuard admin UI into a **router-agnostic, multi-tenant VPN control plane**.

## Open this folder as an Obsidian vault

```
File → Open Folder as Vault → docs/obsidian/
```

Wikilinks (`[[...]]`) and the graph view both work out of the box.

## Start here

1. **[[architecture]]** — system architecture, all mermaid diagrams, the spine of the vault.
2. **[[roadmap]]** — phased delivery plan (foundation → flagship → scale → long-tail).
3. **[[glossary]]** — terminology (engine vs. backend vs. router vs. tenant, etc.).
4. **[[handoff/prd-template]]** — how every PRD is structured.
5. **[[handoff/kimi-prompt-template]]** — how implementation work is handed off to Kimi.

## Quick map

| Area | Folder | Phase |
| --- | --- | --- |
| Foundation refactors | `prds/00-foundation/` | P0 |
| MikroTik backend | `prds/10-mikrotik/` | P1–P2 |
| User-facing features | `prds/20-user-features/` | P1 |
| Multi-engine | `prds/30-multi-engine/` | P2 |
| Multi-server / federation | `prds/40-multi-server/` | P2–P3 |
| Integrations (Tailscale, SSO) | `prds/50-integrations/` | P3 |
| Architectural decisions | `decisions/` | — |

## Authoring rules

- **Every doc has frontmatter.** Status, phase, dependencies, files-touched.
- **Cross-references are wikilinks**, never relative paths. Obsidian resolves by basename.
- **PRDs do not contain implementation code.** They contain interface signatures, data shapes, sequence diagrams, and acceptance tests. Kimi writes the code.
- **One PRD = one Kimi session.** Each PRD must be self-contained enough that Kimi can implement it given only that PRD + [[architecture]] + [[glossary]] + the listed source files.
- **Diagrams live in [[architecture]].** PRDs reference diagrams by anchor; they do not redraw the system.
- **No marketing tone.** Engineers and Kimi are the audience.

## Status legend

- `draft` — being written, not ready to implement
- `approved` — ready for Kimi handoff
- `in-progress` — Kimi is implementing, do not change scope
- `shipped` — merged; the `touches:` list has been verified against the diff

## Repo location

This vault lives at `docs/obsidian/` in the [`manawenuz/wg-easy`](https://github.com/manawenuz/wg-easy) repo. Upstream is [`wg-easy/wg-easy`](https://github.com/wg-easy/wg-easy), tracked as the `upstream` git remote.
