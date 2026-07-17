# 🏭 Anest (آنست) — Enterprise Warehouse Management System (WMS)

> **Enterprise Architectural Case Study & Documentation**  
> **Built For:** Mashhad Railway Facilities Supervision Group (گروه نظارت ساختمان و تاسیسات راه آهن مشهد)  
> *Due to proprietary compliance, source code is hosted in a private repository. This document serves as a comprehensive architectural blueprint and engineering showcase.*

[🇮🇷 نسخه فارسی](README_FA.md)

![Status](https://img.shields.io/badge/Status-Production%20Live-success?style=for-the-badge)
![Architecture](https://img.shields.io/badge/Architecture-Clean%20Architecture%20%2B%20Event%20Sourcing-blue?style=for-the-badge)
![Tech Stack](https://img.shields.io/badge/Stack-Flutter%20%7C%20.NET%208%20%7C%20PostgreSQL%2016-informational?style=for-the-badge)

---

## 📖 Executive Summary

**Anast** is a production-live, enterprise-grade, full-stack Warehouse Management System custom-engineered to manage and track inventory across a strict multi-tiered facility infrastructure. The platform guarantees absolute data integrity by utilizing **Event Sourcing** to maintain an immutable audit trail of every physical stock movement, backed by robust database-level constraint enforcement.

Deployed within a hardened Linux VPS environment via Docker containers, utilizing an Nginx reverse proxy with automated SSL termination, active brute-force prevention, and decoupled real-time background health monitoring.

---

## 📸 Production Showcase

| | | | |
|:---:|:---:|:---:|:---:|
| <img width="260" height="564" alt="Dashboard" src="https://github.com/user-attachments/assets/c5b92be3-d4e4-4371-b50a-6afede1b1e6b" /> | <img width="260" height="566" alt="Warehouses" src="https://github.com/user-attachments/assets/0590cb17-ba52-497b-b1f9-b700ca2eab7e" /> | <img width="260" height="562" alt="Transfers" src="https://github.com/user-attachments/assets/a9e692fc-e8fe-4835-94cb-1317276b28f6" /> | <img width="260" height="564" alt="Settings" src="https://github.com/user-attachments/assets/2fa43ef6-677f-4a7c-8d1b-5253dc5dd3ba" /> |
---

## 🚀 The Business Problem vs. The Engineering Solution

### The Complexities of the Domain

The Mashhad Railway Facilities Group required a foolproof inventory mechanism across physical sites mapped to a strict hierarchy (**Central → Regional → Local**). Standard off-the-shelf CRUD architectures were disqualified due to several strict business realities:

1. **Directional Flow Control:** Downward allocation is standard, but upward inventory transfers (e.g., a Local site transferring inventory back to a Central Hub) must be strictly impossible.
2. **The "Consumption" Ledger Leak:** Tracking consumed or worn-out infrastructure components without destroying the mathematically closed loop of the 3-level ledger system.
3. **Absolute Non-Repudiation:** Total accountability of physical material custody — knowing exactly *who* authorized a mutation, *what* specific serials shifted, and *why*.
4. **Non-Blocking Analytics:** Generating historical cost, burn rates, and financial reports without putting table locks on hot operational transactional rows.

### The Architectural Blueprint

To build a system resilient against invalid states, I decoupled state modification from state reading using a customized **Event-Sourced Engine Layer**.

Instead of mutating a traditional static `quantity` column, the master state of stock balance is treated as a derivative mathematical projection. Stock balances are derived dynamically by replaying sequential, atomic historic blocks (`StockIncreased`, `StockDecreased`).

To preserve the invariant rules of the 3-tier transfer hierarchy, I introduced a virtualized 4th tier: a **"Disposed"** sink. All business logic and spatial boundaries were pushed directly into the engine's core relational engine via custom triggers and stored procedures, neutralizing invalid operations regardless of API bypass attempts.

---

## 🏗️ System Architecture

```text
┌─────────────────────────────────────────────────────────────┐
│                     FLUTTER MOBILE CLIENT                   │
│  ┌───────────┐ ┌───────────┐ ┌────────────┐ ┌────────────┐ │
│  │   Auth    │ │Dashboard  │ │ Transfers  │ │  Reports   │ │
│  └─────┬─────┘ └─────┬─────┘ └─────┬──────┘ └─────┬──────┘ │
│        └─────────────┴─────────────┴──────────────┘        │
│                    flutter_bloc (State)                     │
└───────────────────────────┬─────────────────────────────────┘
                            │ HTTPS (REST API)
┌───────────────────────────▼─────────────────────────────────┐
│              NGINX REVERSE PROXY (Port 443)                 │
│         SSL Termination / Rate Limiting / Caching           │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│              ASP.NET CORE 8 (Docker Container)              │
│  ┌───────────┐ ┌───────────┐ ┌────────────┐ ┌────────────┐ │
│  │ Auth JWT  │ │Inventory  │ │ Products   │ │ Transfers  │ │
│  └─────┬─────┘ └─────┬─────┘ └─────┬──────┘ └─────┬──────┘ │
│        └─────────────┴─────────────┴──────────────┘        │
│                    EF Core 8 (ORM)                          │
└───────────────────────────┬─────────────────────────────────┘
                            │ TCP (Port 5432 - Internal Only)
┌───────────────────────────▼─────────────────────────────────┐
│                PostgreSQL 16 (Docker Container)             │
│  ┌───────────┐ ┌───────────┐ ┌────────────┐ ┌────────────┐ │
│  │ Tables    │ │ Triggers  │ │ Functions  │ │Event Store │ │
│  └───────────┘ └───────────┘ └────────────┘ └────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Decoupled Subsystem Breakdowns

#### 1. Mobile Client (Flutter & Dart)

* **State Isolation:** Powered by `flutter_bloc` to convert raw stream events into distinct immutable state models, strictly separating visual components from reactive layer updates.
* **Network Pipeline:** Configured a resilient `Dio` instance injected with automated interceptors to process real-time HTTP header modifications, bearer JWT refresh tokens, and global `401 Unauthorized` session closures.

#### 2. Backend Engine (.NET 8 Clean Architecture)

* **Domain Layer:** Pure, framework-agnostic models containing business objects, enum states, and foundational aggregate boundaries.
* **Application Layer:** Structured application flows using query abstractions, handling input request verification, data transformations, and tracking user state dependencies.
* **Infrastructure & API Layers:** Leveraged high-speed compilation patterns in ASP.NET Core to handle asynchronous traffic pipelines, utilizing EF Core 8 as an ORM abstraction mapping into native database types.

#### 3. Relational Storage Layer (PostgreSQL 16)

* Highly optimized engine deploying procedural `PL/pgSQL` triggers for sub-millisecond invariant calculations, structural tracking views, and optimized JSONB query operations.

---

## 💡 Key Engineering Achievements

### 1. Hybrid Event Sourcing Implementation

To eliminate any risk of audit forging, mutations are committed through an append-only transaction pipeline.

* **State Rehydration:** When calculating active inventory, the application pulls state entries from an `events` table.
* **Automated Snapshot Engine:** To prevent execution slowdowns as event volume scales, a background `PL/pgSQL` loop watches current event logs. Upon every 100 entries per structural aggregate, a snapshot is compiled and saved into a `snapshots` JSONB field, cutting down subsequent state computation times.
* **Cron-Driven Data Lakes:** A localized utility queries the event engine daily, exporting finished transfers into compressed JSON text stores, which are then passed directly to external cold-storage directories.

### 2. Low-Level Database Invariants & Guardrails

Relying entirely on the web server for business validation creates a single point of failure if an automated script or a raw SQL console bypasses the application layer. To prevent this, the core transfer hierarchy logic was built directly into PostgreSQL triggers:

* **Topology Validation:** A `BEFORE INSERT` constraint engine verifies structural lineage. It blocks data entry if a `local` type assignment is missing a verified `regional` target, or if a `regional` node points to an invalid `central` system entity.
* **Flow Direction Enforcement:** A custom trigger validates transfer vectors, aborting any operation attempting to push material upstream (e.g., Local → Central) by dropping the transaction and returning a structured SQL exception back to the app layer.
* **State Machine Invariants:** Data modifications are bound to an un-skippable status lifecycle (`pending` → `approved` → `in_transit` → `completed`). Any out-of-order state mutations are instantly blocked.

### 3. High-Concurrency Race Mitigation

```text
Simultaneous Operations Requesting Stock Movement from Same Node:

User X: Attempting Transfer ──┐  (Acquires pg_advisory_xact_lock) ──> [Executes Safely]
                              ├─> [Race Condition Blocked]
User Y: Attempting Transfer ──┘  (Queued via Transaction Lock) ────> [Evaluates Real Quantity]
```

During peak operational hours, multiple warehouse operators frequently process inventory adjustments concurrently. If two processes execute simultaneously, a classic race condition could cause double-spending or drop total inventory below zero.

* **The Solution:** The logic is encapsulated inside an isolated stored procedure (`execute_stock_transfer`) built around `pg_advisory_xact_lock`.
* By establishing an explicit lock hash bound directly to the unique `WarehouseId + ProductId` string key, parallel requests on identical resources are queued in order. This guarantees true serial execution and guarantees that secondary queries read real, updated quantity metrics.

### 4. Context-Aware Resource-Level RBAC

The system enforces strict Resource-Based Access Control at the API routing boundary:

* **Administrative Scope:** Owners and global operations managers possess permissions to read, update, or manipulate inventory fields across any node within the global system hierarchy.
* **Keeper Scope:** Standard facility operators are explicitly restricted. The authentication engine cross-references the calling user's JWT ID against the assigned `owner_id` entry listed inside the target `warehouses` table. Any unauthorized mutation attempt immediately triggers a `403 Forbidden` response block, stopping the execution path before it reaches the core services.

---

## 🔒 Security Posture & Hardened DevOps

```text
Public Web Traffic ──[Port 443]──> [ NGINX Reverse Proxy ] ──[Internal Only]──> [ .NET Web API ]
                                         │                                            │
                                   [ Fail2Ban ] <── (Monitors Auth Logs)        [ PostgreSQL ]
```

* **Edge Network Hardening:** The server drops all public connections directed toward internal databases. All traffic is multiplexed through a reverse Nginx proxy over TLS 1.3. The Uncomplicated Firewall (UFW) drops any non-standard packet delivery attempts.
* **OS-Level Shielding:** Direct password authorization protocols over SSH are completely disabled. Access requires signature verification via strict `ED25519` key exchanges. An active `Fail2Ban` daemon parses connection failures, dropping problematic IP ranges trying to execute brute-force attacks.
* **Automated Redundancy Pipelines:** System shell automation tasks run a compressed `pg_dump` sequence every midnight. Backups are stored as encrypted `.sql.gz` targets. A sliding 7-day retention engine maintains snapshot availability while keeping local disk usage consistent.
* **Decoupled Health Inspections:** A native monitoring shell script executes every 5 minutes to ping the primary API wrapper. If a container drops out or responds with an invalid status code, the inspector leverages an isolated `msmtp` service to alert system engineers via email immediately.

---

## 🛠️ Tech Stack

| Layer | Component | Rationale |
|-------|-----------|-----------|
| **Frontend** | Flutter 3.x / Dart 3.x | Compiles into native ARM binaries, delivering smooth performance on tablets and rugged devices. |
| **State Mgmt** | `flutter_bloc` 8.x | Predictable state transitions, making complex state debugging trivial. |
| **Backend** | ASP.NET Core 8 (C#) | High-performance asynchronous processing with excellent type safety. |
| **ORM** | Entity Framework Core 8 | Safe SQL generation with raw SQL injection support for optimized operations. |
| **Database** | PostgreSQL 16 | Strong concurrency primitives, transactional stability, reliable stored procedures. |
| **Orchestration** | Docker / Nginx / Certbot | Isolated deployment, automated SSL, secure network gateway. |
| **OS** | Ubuntu 22.04 LTS | Long-term support with regular security patches. |

---

## 📁 Project Structure

```
WMS/
├── WMS.API/                # API Routing — REST Gateways, Middleware, JWT Handlers, Swagger
├── WMS.Application/        # Business Logic — Command/Query Handlers, DTOs, Interfaces
├── WMS.Domain/             # Enterprise Core — Entities, Value Objects, Domain Exceptions
├── WMS.Infrastructure/     # External Systems — EF Core DBContext, Repository Implementations
└── wms_app/                # Client — Flutter Mobile Application
    └── lib/
        ├── core/           # Configs — API Interceptors, App Themes, Constants
        └── features/       # Modules — Auth, Warehousing, Transfers (BLoC + Views)
```

