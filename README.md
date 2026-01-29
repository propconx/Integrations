# 🧩 Propvalue – Node-RED Integration Templates

## Overview

This repository contains **public, reference-grade Node-RED integration templates** maintained by **Propvalue**.
Each integration follows a shared architectural pattern and is designed to be:

- **Production-ready**
- **Explicitly documented**
- **Restart-safe** (persistent context)
- **User-extensible**

The purpose of this repository is to provide **clear, inspectable examples** of how Propvalue integrates with
common building management and control system platforms.

These templates are intended for:
- Partners implementing Propvalue-compatible integrations
- Customers running Propvalue on edge devices
- Internal and external developers who need a canonical reference

---

## 🎯 Common Integration Pattern

All integrations in this repository follow the same conceptual model:

### 1. Configuration (Externalized)
Each integration expects a `global.config` object to exist in **persistent Node-RED context**.
Credentials, identifiers, and runtime parameters are **never hard-coded** into flows.

### 2. Gross List (Full Inventory)
A discovery flow builds a **Gross List** containing *all* available datapoints exposed by the source system.

- One entry per physical datapoint
- Stable, deterministic identity (`HASH`)
- No filtering or business logic

### 3. Net List (User-Managed Subset)
The **Net List** is a user-maintained subset of the Gross List.

- Controlled via flags such as:
  ```json
  { "toStream": true }
  ```
- Defines what is:
  - Polled
  - Streamed
  - Exported as telemetry

> The Net List is **never auto-modified** by the integration logic.

### 4. Metadata Export
Separate flows emit metadata for:
- The full Gross List
- The current Net List

These flows are transport-agnostic and can be connected to HTTP, MQTT, databases, etc.

### 5. Telemetry Pipeline
Live data is collected, normalized, and guarded before emission.

Common guarantees:
- Deterministic point identity
- ISO-8601 UTC timestamps
- Typed values (Number / Boolean / String)
- Restart-safe state handling

### 6. Guards & Filters
All integrations implement:
- **Time-Gate** – drops out-of-order or replayed data
- **Change / Keepalive** – suppresses unchanged values while ensuring liveness

---

## 📦 Included Integrations

### 🔌 Larmia

Integration for **Larmia-based control systems**.

**Capabilities**
- Token-based authentication
- Point discovery via `/h-api/openapi/points`
- Time-series polling via `/s-api/openapi/currentvalue/list`
- Full Gross List / Net List lifecycle
- High-volume telemetry support

📄 Documentation: `larmiaIntegration.md`

---

### 🔌 Lindinspect

Integration for **Lindinspect and similar systems**.

**Capabilities**
- Hierarchical node and leaf discovery
- Unit normalization from API-provided metadata
- Deterministic hash identity per datapoint
- Configurable polling with rate control
- Robust metadata and telemetry flows

📄 Documentation: `lindinspectIntegration.md`

---

### 🔌 OBIX (Niagara / Tridium)

Integration for **Niagara / OBIX–based systems**.

**Capabilities**
- Session-based authentication (Niagara cookies)
- System and point discovery via `/obix/config`
- Live values via **OBIX WatchService**
- Stable identity using `/out/` URIs
- Watch-based polling with refresh and change semantics

📄 Documentation: `obix.md`

> Example system names used in documentation are illustrative.
> Actual system and component names depend on the station configuration.

---

## 🧱 Responsibility Model

All integrations deliberately separate responsibilities:

| Responsibility | Integration | User |
|---------------|-------------|------|
| Configuration structure | Example | Real values / secrets |
| Authentication | ✅ | — |
| Inventory discovery | ✅ | — |
| Net List selection | — | ✅ |
| Metadata transport | — | ✅ |
| Telemetry transport | — | ✅ |
| Filtering & guards | ✅ | — |

This keeps integrations predictable, auditable, and easy to extend.

---

## 🚀 Getting Started

1. Install Node-RED with **persistent context** enabled
2. Clone this repository
3. Choose the integration you need
4. Import the Node-RED flows
5. Create `global.config` for your environment
6. Run the Gross List builder once
7. Select points in the Net List
8. Attach output nodes and deploy

Each integration’s documentation contains **step-by-step deployment instructions**.

---

## 🔒 Security Notes

- Credentials are never stored in flows
- All sensitive values are expected to be injected via `global.config`
- Example values in documentation are anonymized
- No customer-specific data is included in this repository

---

## 📚 License & Usage

This repository is provided as a **public reference implementation** by **Propvalue AB**.

You are free to:
- Study the flows
- Adapt them for your environment
- Build derivative integrations

Subject to your commercial agreement with Propvalue.

---

© Propvalue AB
