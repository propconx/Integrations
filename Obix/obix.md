# 🧩 OBIX Integration – Node-RED Template

## Overview

This repository provides a **fully implemented Node-RED integration** for **Niagara / OBIX–based control systems** (e.g. Tridium Niagara).
It follows Propvalue’s standard integration pattern and implements the full pipeline from **point discovery** to **normalized telemetry streaming**.

The template enables you to:

- Discover and normalize all available OBIX points into a **Gross List**
- Maintain a user-managed streaming subset as a **Net List**
- Fetch live values using **OBIX WatchService**
- Apply robust data guards:
  - **Time-Gate** (drop out-of-order data)
  - **Change / Keepalive**
- Emit standardized telemetry to any downstream system

All flows are modular, documented, and reusable.

---

## ⚙️ Architecture Summary

| Flow (tab) | Description |
|-----------|-------------|
| **OBIX – gross list** | Logs in to Niagara, discovers systems and points, builds and persists the **Gross List** (`global.obixGrossList`). |
| **OBIX – Send gross metadata** | Flattens the Gross List and emits metadata for all known points. |
| **OBIX – Send net metadata** | Flattens the Net List and emits metadata for the user-selected subset. |
| **OBIX – Telemetry data** | Creates watches, polls values, normalizes telemetry, applies guards, and outputs time-series data. |

> **Note:** This integration expects a `global.config` object to exist in **persistent context** before any flows are run.

---

## 🔐 Configuration – `global.config`

### Required Structure

```json
{
  "scheme": "https",
  "host": "10.104.0.43",
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

---

## 🧱 Gross List

The Gross List flow discovers all OBIX points under `/obix/config/System/`, normalizes them, fetches initial values, and stores them in persistent context.

Points are uniquely identified by their `/out/` URI and assigned a deterministic hash.

---

## 🧩 Net List

The Net List is a **user-managed subset** of the Gross List.

Only points with:

```json
{ "toStream": true }
```

are included in telemetry polling and streaming.

---

## 🧾 Metadata

Separate flows exist for exporting:

- **Gross metadata** – all discovered points
- **Net metadata** – only points in the Net List

Both emit flattened metadata arrays suitable for HTTP, MQTT, or database sinks.

---

## ⏱️ Telemetry

Telemetry is collected using **OBIX WatchService**:

1. Create watch
2. Add points
3. Initial `pollRefresh`
4. Repeated `pollChanges`
5. Normalize and filter values
6. Emit telemetry per point

Example payload:

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

## 🛡️ Guards

- **Time-Gate**: drops out-of-order or replayed values
- **Change / Keepalive**: emits only changes, with periodic keepalive updates

---

## 🚀 Deployment

1. Enable persistent context in Node-RED
2. Import all OBIX flows
3. Create `global.config`
4. Run Gross List builder
5. Mark Net List points with `toStream: true`
6. Attach output nodes
7. Deploy

---

## 📚 License

Provided as a reference implementation for Propvalue integrations.

© Propvalue AB
