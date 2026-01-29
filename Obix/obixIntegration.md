# 🧩 OBIX Integration – Node-RED Template

## Overview

This repository provides a **fully implemented Node-RED integration** for **Niagara / OBIX–based control systems** (e.g. Tridium Niagara).
It follows Propvalue’s standard integration pattern and mirrors the level of detail and responsibility split used in other integrations such as **Larmia** and **Lindinspect**.

The integration covers the **entire lifecycle** of OBIX data handling:

- Authentication and session handling (Niagara cookies)
- Point discovery and inventory building (**Gross List**)
- User-managed streaming selection (**Net List**)
- Metadata export (gross + net)
- Live value polling using **OBIX WatchService**
- Telemetry normalization and enrichment
- Robust data guards (time-gate, change/keepalive)
- Fan-out to downstream transports

All flows are production-grade, stateful, and restart-safe through **persistent context**.

---

## ⚙️ Architecture Summary

| Flow (tab) | Description |
|-----------|-------------|
| **OBIX – gross list** | Authenticates against Niagara, discovers systems and points, fetches initial values, and builds the **Gross List** (`global.obixGrossList`). |
| **OBIX – Send gross metadata** | Flattens the Gross List and emits metadata for **all discovered points**. |
| **OBIX – Send net metadata** | Flattens the Net List and emits metadata for the **user-selected subset**. |
| **OBIX – Telemetry data** | Creates OBIX watches, polls values, normalizes telemetry, applies guards, and emits time-series data. |

> **Important:** This integration expects a valid `global.config` object to exist in **persistent context** before any flows are triggered.

---

## 🔐 Configuration – `global.config`

### Purpose

All flows read connection details, identifiers, and runtime parameters from `global.config` stored in **persistent** context.

### Required Structure

```json
{
  "scheme": "https",
  "host": "10.100.0.10",
  "port": 443,
  "user": "apiuser",
  "pass": "secret",
  "JOB_UUID": "job-obix-01",
  "property": "prop1",
  "deviceId": "edge-01",
  "pollingInterval": 60000,
  "baseUrl": "obix"
}
```

### Field semantics

| Field | Description |
|------|-------------|
| `scheme` | `http` or `https` |
| `host` | Niagara controller hostname or IP |
| `port` | Optional port (defaults by scheme) |
| `user` / `pass` | Niagara credentials |
| `JOB_UUID` | Integration/job identifier |
| `property` | Property / site identifier |
| `deviceId` | Edge device identifier |
| `pollingInterval` | Poll interval (ms or seconds, auto-normalized) |
| `baseUrl` | OBIX base path (usually `obix`) |

### Example bootstrap Function

```javascript
const cfg = {
  scheme: "https",
  host: "10.100.0.10",
  port: 443,
  user: "apiuser",
  pass: "secret",
  JOB_UUID: "job-obix-01",
  property: "prop1",
  deviceId: "edge-01",
  pollingInterval: 60000,
  baseUrl: "obix"
};

global.set("config", cfg, "persistent");
msg.payload = { ok: true, saved: cfg };
return msg;
```

---

## 🧱 Gross List – *OBIX – gross list*

The **Gross List** represents the full discovered OBIX inventory.

### Discovery process

1. Authenticate using `POST /obix/about`
2. Persist the returned Niagara session cookie in flow context
3. Fetch `/obix/config/System/`
4. Iterate each system folder
5. Fetch `/obix/config/System/{system}/`
6. Discover all `obix:Point` references
7. Build a canonical `/out/` URI per point
8. Fetch initial live values and units
9. Persist results in `global.obixGrossList`

### Storage structure

```js
global.obixGrossList = {
  "<controllerIp>": {
    systems: {
      "<systemName>": [ point, point, ... ]
    }
  }
}
```

### Point normalization

Each discovered point is normalized into a stable structure:

```json
{
  "name": "Room temperature",
  "description": "Room temperature",
  "label": "/obix/config/System/VS3/VS3$2dGT11_mv/out/",
  "href": "/obix/config/System/VS3/VS3$2dGT11_mv/",
  "property": "prop1",
  "source": "10.104.0.43",
  "system": "VS3",
  "component": "GT11",
  "type": "number",
  "integrationType": "Obix",
  "unit": "°C",
  "value": 22.4,
  "systemId": "prop1_VS3",
  "componentId": "prop1_VS3_GT11",
  "HASH": "8f3c2a91",
  "toStream": "Unknown",
  "deprecated": false,
  "lastSeen": "2025-01-12T09:21:00Z"
}
```

### Identity guarantees

- **Stable identity** is based on the `/out/` URI
- Hash collisions are practically avoided by combining:
  - integration type
  - property
  - controller source
  - label (URI)

---

## 🧩 Net List – User-Managed Streaming Selection

The **Net List** defines which points are streamed as telemetry.

- Stored as:
  ```js
  global.obixNetList
  ```
- Initially mirrored from the Gross List
- **Never auto-modified** by the integration

To enable streaming:

```json
{
  "toStream": true
}
```

Only points with `toStream === true` are:

- Added to OBIX watches
- Polled for live values
- Emitted as telemetry

This mirrors the responsibility model used in other Propvalue integrations.

---

## 🧾 Metadata Export

### Send gross metadata – OBIX

- Reads from `global.obixGrossList`
- Flattens all systems and points
- Adds `jobUuid`, `property`, and `deviceId`
- Emits a single payload:

```json
{
  "jobUuid": "job-obix-01",
  "property": "prop1",
  "deviceId": "edge-01",
  "points": [ ... ]
}
```

### Send net metadata – OBIX

- Reads from `global.obixNetList`
- Identical schema to gross metadata
- Represents the **current streaming configuration**

Downstream transport (HTTP, MQTT, DB, etc.) is user responsibility.

---

## ⏱️ Telemetry – *OBIX – Telemetry data*

Live values are handled using **OBIX WatchService**.

### Watch lifecycle

1. Create watch: `POST /obix/watchService/make`
2. Store returned `watchUri`
3. Add points to watch
4. Initial snapshot: `pollRefresh`
5. Incremental updates: `pollChanges`

### Poll scheduler

- Driven by `global.config.pollingInterval`
- Accepts milliseconds or seconds
- Restart-safe via persistent context

### Value merging

- Incoming values are matched by `/out/` URI
- Units are normalized when provided by OBIX
- Values are type-cast (number, boolean, string)
- Results are written back to `global.obixNetList`

### Telemetry envelope (per point)

```json
{
  "job_uuid": "job-obix-01",
  "time": "2025-01-12T09:30:00Z",
  "value": 22.4,
  "label": "/obix/config/System/VS3/VS3$2dGT11_mv/out/",
  "hash": "8f3c2a91",
  "input": "tridium"
}
```

---

## 🛡️ Filters and Guards

### Time-Gate

- Drops replayed or out-of-order data
- Tracks latest accepted timestamp per HASH
- Uses compact rolling arrays with automatic compaction

### Change / Keepalive

- Emits on value change
- Emits periodic keepalive updates
- Prevents noisy unchanged streams

Both filters are **stateful and restart-safe**.

---

## 🧱 Responsibility Split

| Stage | Template | User |
|------|---------|------|
| Configuration | example | real values |
| Authentication | ✅ | — |
| Gross List | ✅ | — |
| Net List | initial mirror | ✅ |
| Metadata export | ✅ | attach output |
| Telemetry polling | ✅ | ensure `toStream` |
| Guards | ✅ | — |
| Transport | — | ✅ |

---

## 🚀 Deployment Steps

1. Enable **persistent context** in Node-RED
2. Import all OBIX flows
3. Create and persist `global.config`
4. Run **Build obixGrossList**
5. Edit `global.obixNetList` (`toStream: true`)
6. Attach metadata and telemetry outputs
7. Deploy and monitor

---


