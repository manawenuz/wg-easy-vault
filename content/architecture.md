---
title: Architecture
type: spine
---

# Architecture

The spine of this vault. All PRDs reference diagrams here by anchor (`[[architecture#5-auth-flows]]`). When the design changes, this file changes first; PRDs follow.

## Table of contents

1. [System context](#1-system-context)
2. [Control plane layers](#2-control-plane-layers)
3. [VpnEngine driver interface](#3-vpnengine-driver-interface)
4. [Data model](#4-data-model)
5. [Auth flows](#5-auth-flows)
6. [MikroTik provisioning](#6-mikrotik-provisioning)
7. [Quota enforcement loop](#7-quota-enforcement-loop)
8. [Multi-router federation](#8-multi-router-federation)
9. [Multi-path routing](#9-multi-path-routing)
10. [Phasing dependency graph](#10-phasing-dependency-graph)

---

## 1. System context

```mermaid
graph TD
    subgraph Browsers
        AdminUI[Admin UI<br/>Vue/Nuxt]
        UserUI[User Dashboard<br/>Vue/Nuxt]
    end

    subgraph "Control plane (wg-easy app)"
        API[Nitro HTTP API]
        Sched[Scheduler<br/>quota reset / usage poll]
        Engines[Engine drivers<br/>VpnEngine impls]
        DB[(SQLite<br/>Drizzle)]
    end

    subgraph "Data plane (routers / hosts)"
        Local[Local kernel WG<br/>wg / awg]
        Boring[BoringTun<br/>userspace]
        MT[MikroTik<br/>RouterOS]
    end

    AdminUI -->|cookie / token| API
    UserUI -->|cookie / token| API
    API --> DB
    API --> Engines
    Sched --> DB
    Sched --> Engines
    Engines -->|exec| Local
    Engines -->|exec| Boring
    Engines -->|RouterOS API + SSH| MT
```

The **control plane** is one process. The **data plane** is one or more routers it manages. The local box is just `router_id = 0` — the same code path that drives a remote MikroTik drives the local kernel WireGuard. This is the central simplification of the fork.

PRDs that elaborate this view: [[prds/00-foundation/01-backend-abstraction]], [[prds/40-multi-server/01-multi-router-federation]].

---

## 2. Control plane layers

```mermaid
flowchart LR
    HTTP[HTTP route<br/>src/server/api/...] --> Service[Service layer<br/>src/server/services/...]
    Service --> Repo[Repository<br/>src/server/database/repositories/...]
    Service --> Engine[VpnEngine<br/>src/server/engines/...]
    Repo --> DB[(SQLite)]
    Engine --> Transport[Transport<br/>local-shell / ssh / routeros-api]
    Transport --> DataPlane[Data plane]
```

Today wg-easy mixes service logic into route handlers and into the `WireGuard` class. We introduce an explicit **service layer** to mediate between HTTP and (engine + repo) so the same operation (e.g., "create client") works identically whether triggered by the admin UI, the user dashboard, an API token, or the scheduler.

A **transport** is the thing that physically delivers a command. The current code has one transport: `child_process.exec` (`src/server/utils/cmd.ts`). We add `ssh` and `routeros-api` as siblings. Drivers compose transports — the MikroTik driver uses `routeros-api` for steady state and `ssh` for bootstrap.

PRDs: [[prds/00-foundation/01-backend-abstraction]], [[prds/10-mikrotik/01-mikrotik-driver]].

---

## 3. VpnEngine driver interface

```mermaid
classDiagram
    class VpnEngine {
        <<interface>>
        +id: EngineType
        +capabilities: EngineCapabilities
        +healthCheck(iface: InterfaceType) Promise~Health~
        +syncInterface(iface: InterfaceType, peers: Client[]) Promise~void~
        +createPeer(iface: InterfaceType, peer: Client) Promise~void~
        +updatePeer(iface: InterfaceType, peer: Client) Promise~void~
        +removePeer(iface: InterfaceType, peerPublicKey: string) Promise~void~
        +enablePeer(iface: InterfaceType, peerPublicKey: string) Promise~void~
        +disablePeer(iface: InterfaceType, peerPublicKey: string) Promise~void~
        +applySpeedLimit(iface, peerPublicKey, upKbps, downKbps) Promise~void~
        +sampleUsage(iface: InterfaceType) Promise~UsageSample[]~
        +bringUp(iface: InterfaceType) Promise~void~
        +bringDown(iface: InterfaceType) Promise~void~
    }

    class WireguardEngine {
        -transport: LocalShellTransport
        +generateConfig(iface, peers) string
    }
    class AmneziaWgEngine {
        -transport: LocalShellTransport
        +generateConfig(iface, peers) string
    }
    class BoringtunEngine {
        -transport: LocalShellTransport
        -process: ChildProcess
    }
    class MikrotikEngine {
        -api: RouterOSClient
        -ssh: SshTransport
        +bootstrap(router) Promise~void~
    }

    VpnEngine <|.. WireguardEngine
    VpnEngine <|.. AmneziaWgEngine
    VpnEngine <|.. BoringtunEngine
    VpnEngine <|.. MikrotikEngine
```

One interface, four implementations. The interface is **deliberately narrow**: every method maps to a single user-visible operation. Capability flags (`supportsObfuscation`, `supportsSpeedLimit`) let the UI gracefully degrade — e.g., the speed-limit input is disabled if the selected engine doesn't support it.

`sampleUsage()` returns per-peer rx/tx counters. The scheduler polls this on an interval (default 60s) and writes `usage_sample` rows; the quota engine reads those.

`bringUp` / `bringDown` exist because some engines (BoringTun, MikroTik) have a meaningful "interface state" that can't be re-derived from config alone.

PRD: [[prds/00-foundation/01-backend-abstraction]].

---

## 4. Data model

```mermaid
erDiagram
    USER ||--o{ CLIENT : owns
    USER ||--o{ ADMIN_ROUTER_ACL : "scoped to"
    ROUTER ||--o{ WG_INTERFACE : hosts
    WG_INTERFACE ||--o{ CLIENT : "has peers"
    CLIENT ||--o| QUOTA : has
    CLIENT ||--o| SPEED_LIMIT : has
    CLIENT ||--o{ USAGE_SAMPLE : generates
    USER ||--o{ AUDIT_LOG : actor
    WG_INTERFACE ||--o{ ROUTE_POLICY : has
    ROUTE_POLICY ||--|| EXIT_NODE : routes_to

    USER {
        int id PK
        string username
        string password_hash
        string role  "superadmin|admin|operator|viewer|client"
        string email
        bool enabled
        bool totp_verified
        string totp_secret
        timestamp created_at
    }
    ROUTER {
        int id PK
        string name
        string engine_type "wireguard|amneziawg|boringtun|mikrotik"
        string transport "local-shell|ssh|routeros-api"
        string host
        int port
        json credentials_encrypted
        bool enabled
        timestamp last_seen
    }
    WG_INTERFACE {
        int id PK
        int router_id FK
        string name
        string engine_type "denormalized from router"
        int port
        string ipv4_cidr
        string ipv6_cidr
        string public_key
        string private_key
        json amnezia_params "jC,jMin,jMax,s1-s4,h1-h4,i1-i5"
        bool enabled
    }
    CLIENT {
        int id PK
        int user_id FK
        int interface_id FK
        string name
        string public_key
        string private_key
        string preshared_key
        string ipv4_address
        string ipv6_address
        bool enabled
        timestamp expires_at
        timestamp created_at
    }
    QUOTA {
        int client_id PK_FK
        bigint limit_bytes
        string period "daily|weekly|monthly"
        bigint used_bytes
        timestamp period_start
        timestamp period_end
        bool auto_disable
    }
    SPEED_LIMIT {
        int client_id PK_FK
        int up_kbps
        int down_kbps
    }
    USAGE_SAMPLE {
        int id PK
        int client_id FK
        bigint rx_bytes
        bigint tx_bytes
        timestamp ts
    }
    ADMIN_ROUTER_ACL {
        int user_id FK
        int router_id FK
        string permission "read|write|admin"
    }
    AUDIT_LOG {
        int id PK
        int actor_user_id FK
        string action
        json target
        timestamp ts
    }
    EXIT_NODE {
        int id PK
        int router_id FK
        string label
        bool enabled
    }
    ROUTE_POLICY {
        int id PK
        int interface_id FK
        int client_id FK_NULL
        string match_cidr
        int exit_node_id FK
        int priority
    }
```

Two tables already exist (`user`, `wg_interface`, `client`); the rest are added in [[prds/00-foundation/04-data-model-migration]]. `engine_type` is denormalized from `router` to `wg_interface` for query efficiency.

`USAGE_SAMPLE` is the highest-volume table. It is partitioned by retention: keep raw 60s samples for 7 days, then roll up to hourly aggregates. The quota engine queries the rollup, not raw samples, except for "current period in progress."

`ROUTE_POLICY.client_id` is nullable: a policy may match on CIDR alone (subnet routing) or be scoped to a specific client.

---

## 5. Auth flows

### 5a. Admin login (existing, lightly extended)

```mermaid
sequenceDiagram
    participant U as Admin
    participant FE as Admin UI
    participant API as /api/session
    participant DB as user table
    U->>FE: enters username/password (+ TOTP)
    FE->>API: POST {username, password, totp?}
    API->>DB: lookup user, verify hash, verify totp
    DB-->>API: user{id, role}
    API-->>FE: Set-Cookie: wg-session (encrypted)
    FE->>API: subsequent requests with cookie
    API->>API: hasPermissions(user, action)
```

### 5b. User dashboard login (NEW — by QR or pubkey)

```mermaid
sequenceDiagram
    participant U as End user
    participant FE as User Dashboard
    participant API as /api/user-session
    participant DB
    Note over U: User has WG config or QR from admin
    U->>FE: scans QR / pastes config
    FE->>FE: extract private_key from config
    FE->>FE: derive public_key
    FE->>API: POST {public_key, signed_challenge}
    API->>DB: lookup client by public_key
    DB-->>API: client + user
    API->>API: verify signature with WG curve25519
    API-->>FE: Set-Cookie: wg-user-session
```

The user does **not** have a password. They prove ownership of their WireGuard private key by signing a server-issued challenge. The dashboard is read-only by default — view usage, expiry, status, download a fresh config — with no admin powers.

### 5c. Multi-admin RBAC check

```mermaid
sequenceDiagram
    participant Req as Request
    participant MW as auth middleware
    participant ACL as admin_router_acl
    Req->>MW: resource = router_id 7, action = "write"
    MW->>MW: load session.user
    alt user.role == "superadmin"
        MW-->>Req: allow
    else
        MW->>ACL: SELECT permission WHERE user_id=? AND router_id=7
        ACL-->>MW: "read" | "write" | "admin" | none
        MW->>MW: compare to required permission
        MW-->>Req: allow / 403
    end
```

PRDs: [[prds/00-foundation/03-auth-refactor]], [[prds/00-foundation/02-multi-admin-rbac]], [[prds/20-user-features/02-qr-key-login]].

---

## 6. MikroTik provisioning

```mermaid
sequenceDiagram
    participant Admin
    participant App as wg-easy
    participant SSH as SSH transport
    participant API as RouterOS API
    participant MT as MikroTik

    Admin->>App: Add router (host, ssh creds)
    App->>SSH: connect, /system identity print
    SSH-->>App: RouterOS version
    App->>SSH: enable API service, create api-user
    App->>SSH: /interface/wireguard add (if missing)
    App->>API: connect with api-user
    API-->>App: connected
    App->>API: /ip/address add (CIDR)
    App->>API: /ip/firewall/filter add (allow input UDP/port)
    App->>API: /ip/firewall/nat add (masquerade)
    App->>App: store router with engine=mikrotik
    Admin->>App: Create peer
    App->>API: /interface/wireguard/peers add (pubkey, allowed-ips)
    API-->>App: ok
    App->>API: /queue/tree add (if speed limit)
```

Two transports per MikroTik: **SSH for bootstrap** (creates the API user, enables the API service, sets up the WireGuard interface and firewall rules), **RouterOS API for steady state** (peer CRUD, queue tree updates, usage polling). After bootstrap, SSH is only re-used for upgrades and disaster recovery.

The `bootstrap` step is idempotent: re-running it on an already-configured router should produce no changes (modulo credential rotation).

PRDs: [[prds/10-mikrotik/01-mikrotik-driver]], [[prds/10-mikrotik/02-mikrotik-autoconfig]].

---

## 7. Quota enforcement loop

```mermaid
stateDiagram-v2
    [*] --> Active
    Active --> Active: usage poll<br/>(used_bytes += delta)
    Active --> OverQuota: used_bytes >= limit_bytes
    OverQuota --> Disabled: engine.disablePeer()
    Disabled --> Active: scheduler @ period_end<br/>(reset used_bytes, period_start)
    Active --> Active: admin manual reset
    Disabled --> Active: admin manual override<br/>(audit logged)
```

**Sample → Accumulate → Compare → Act → Reset.**

- Scheduler ticks every N seconds (default 60). For each enabled interface, it calls `engine.sampleUsage()`, diffs against the previous sample, writes to `usage_sample`, increments `quota.used_bytes`.
- When `used_bytes >= limit_bytes`, the scheduler calls `engine.disablePeer()` and writes an audit log entry.
- A second scheduler (cron-like) runs at midnight UTC for daily, Monday 00:00 for weekly, 1st of month 00:00 for monthly. It resets `used_bytes`, advances `period_start`/`period_end`, and re-enables peers that were *only* disabled by quota (not manually disabled).

Edge case: if the user is disabled by both quota AND manual admin action, period reset re-enables only if the manual disable was lifted. Track this via `audit_log` reasons rather than a separate `disable_reason` column.

PRD: [[prds/20-user-features/03-bandwidth-quotas]].

---

## 8. Multi-router federation

```mermaid
graph TB
    subgraph "Orchestrator (wg-easy with role=orchestrator)"
        UI[Admin UI]
        DB[(Canonical DB<br/>routers, clients, quotas)]
        Disp[Dispatcher]
    end

    subgraph "Agent A (wg-easy with role=agent)"
        AgentA[Agent worker]
        LocalA[Local WG]
    end

    subgraph "Agent B"
        AgentB[Agent worker]
        MT[MikroTik]
    end

    subgraph "Agent C"
        AgentC[Agent worker]
        LocalC[Local WG]
    end

    UI --> DB
    UI --> Disp
    Disp -->|mTLS gRPC or HTTP+token| AgentA
    Disp -->|...| AgentB
    Disp -->|...| AgentC
    AgentA -->|VpnEngine| LocalA
    AgentB -->|VpnEngine| MT
    AgentC -->|VpnEngine| LocalC
```

The **orchestrator** is the only node with the canonical DB and the admin UI. **Agents** are wg-easy installations running in agent mode: no UI, no DB beyond a small local cache, just a worker that takes commands from the orchestrator and executes them via the same `VpnEngine` drivers.

A "router" in the data model can be:
- `transport=local-shell` on the orchestrator (= "local interface, no agent")
- `transport=ssh` or `transport=routeros-api` (= "remote device, no agent")
- `transport=agent` (= "remote wg-easy agent, which then talks to its local data plane via local-shell")

Why agents? Because some deployments can't expose RouterOS API or SSH externally; the agent makes an outbound mTLS connection to the orchestrator instead.

PRD: [[prds/40-multi-server/01-multi-router-federation]].

---

## 9. Multi-path routing

```mermaid
graph LR
    Client[Client peer<br/>10.8.0.42] --> IFace[wg0]
    IFace --> Policy{ROUTE_POLICY<br/>match by client / CIDR}
    Policy -->|dst 1.1.1.0/24| ENA[Exit node A<br/>router_id=2]
    Policy -->|dst 8.8.8.0/24| ENB[Exit node B<br/>router_id=3]
    Policy -->|default| ENC[Exit node C<br/>router_id=0]
    ENA --> Internet1[Internet via ISP1]
    ENB --> Internet2[Internet via ISP2]
    ENC --> Internet3[Internet via ISP3]
```

A `ROUTE_POLICY` row says: "for traffic from `client_id` (or any client matching `match_cidr`) destined for `match_dst_cidr`, send it via `exit_node_id`." The control plane translates these into `ip rule` + `ip route` on Linux exits and `/ip/route/rule` on MikroTik exits.

This is the most complex feature in the roadmap and is gated behind multi-router federation (you can't have multiple exits without multiple routers). Hence P3.

PRD: [[prds/40-multi-server/03-multi-path-routing]].

---

## 10. Phasing dependency graph

```mermaid
graph TD
    BA[backend-abstraction]
    DM[data-model-migration]
    AR[auth-refactor]
    RBAC[multi-admin-rbac]

    MD[mikrotik-driver]
    MA[mikrotik-autoconfig]
    UD[user-dashboard]
    QR[qr-key-login]
    BQ[bandwidth-quotas]
    SL[speed-limits]

    AWG[amneziawg-promotion]
    BT[boringtun-driver]
    ESU[engine-selection-ux]
    MRF[multi-router-federation]
    ARA[admin-router-acl]
    MO[mikrotik-obfuscation]

    MPR[multi-path-routing]
    TS[tailscale]
    SSO[sso]

    BA --> DM
    DM --> AR
    AR --> RBAC

    BA --> MD
    DM --> MD
    MD --> MA
    AR --> UD
    UD --> QR
    DM --> BQ
    BA --> BQ
    BA --> SL
    DM --> SL

    BA --> AWG
    BA --> BT
    BA --> ESU
    AWG --> ESU
    BT --> ESU
    DM --> MRF
    RBAC --> MRF
    MRF --> ARA
    MD --> MO

    MRF --> MPR
    BA --> TS
    AR --> SSO
    RBAC --> SSO

    classDef p0 fill:#1f77b4,color:#fff
    classDef p1 fill:#2ca02c,color:#fff
    classDef p2 fill:#ff7f0e,color:#fff
    classDef p3 fill:#888,color:#fff

    class BA,DM,AR,RBAC p0
    class MD,MA,UD,QR,BQ,SL p1
    class AWG,BT,ESU,MRF,ARA,MO p2
    class MPR,TS,SSO p3
```

Read this as: every arrow is "must ship before". The four P0 PRDs form a chain (no parallelism). P1 fans out from P0. P2 mostly depends on P0 + at least one P1. P3 is the long tail.

If you want to know whether a PRD is unblocked, check that every node it points back to has `status: shipped`.
