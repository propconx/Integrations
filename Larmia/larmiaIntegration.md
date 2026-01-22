# 🧩 Larmia Integration – Node-RED Template

## Overview

This repository provides a **fully implemented Node-RED integration** for **Larmia**-based control systems.  
It follows Propvalue’s integration template conventions and includes the full pipeline from **point discovery** to **normalized telemetry streaming**.

The package enables you to:

- Discover and normalize all available datapoints into a **Gross List**
- Maintain a user-managed streaming subset as a **Net List**
- Poll live values from Larmia’s `/h-api/openapi/points` and `/s-api/openapi/currentvalue/list` endpoints
- Apply robust data guards:
  - **Time-Gate (drop out-of-order data)**
  - **Change / Keepalive (per-point value tracking)**
- Emit standardized, normalized telemetry messages to any downstream system

All building blocks are modular, documented, and reusable.

---

## ⚙️ Architecture Summary

| Flow (tab) | Description |
|-----------|-------------|
| **Skapa bruttolista – Larmia** | Creates and maintains the **Gross List** for Larmia: config bootstrap, token management, and point discovery via `/h-api/openapi/points`. Normalizes points, parses labels, builds deterministic hashes, and mirrors into `global.netList` as an editable Net List. |
| **Send gross metadata – Larmia** | Flattens all points in `global.grossList` and sends metadata for every known point (no live values, no `hasValue`). |
| **Send net metadata – Larmia** | Flattens all points in `global.netList` and sends metadata for the current Net List (the user-managed subset). `hasValue` is stripped from the payload. |
| **Tidsseriedata – Larmia** | Time-series pipeline: polls live values using `/s-api/openapi/currentvalue/list` **only for points where `toStream === true`**, merges results into `global.netList`, shapes telemetry, applies time-gate and change/keepalive filters, and outputs normalized telemetry. |

> **Note:** Like other templates, this integration **expects** a `global.config` object to exist in persistent context. See **Configuration** below.

---

## 🔐 Configuration – `global.config`

### Purpose

All flows read connection and runtime settings from `global.config` in **persistent** context.  
You must create this object before running the integration.

### Required Structure

```json
{
  "scheme": "http",
  "host": "10.0.0.1",
  "port": 80,
  "user": "username",
  "pass": "password",
  "JOB_UUID": "job2",
  "property": "prop1",
  "deviceId": "edge-01",
  "pollingInterval": 60000,
  "baseUrl": "http://10.0.0.1:80/api/v1"
}
```

- `scheme`, `host`, `port` – used for **all** Larmia HTTP calls (`/w-api`, `/h-api`, `/s-api`).
- `user`, `pass` – Larmia credentials for `/w-api/v1/accounts/gettoken`.
- `JOB_UUID` – integration/job identifier copied into metadata and telemetry.
- `property` – site / property identifier, also used in hashing.
- `deviceId` – edge device identifier (read from `global.deviceInfo` if available).  
- `pollingInterval` – preferred poll interval for time-series (ms or “seconds that will be normalized” – see your scheduler).
- `baseUrl` – convenience base for other integrations; the Larmia flows in this template build URLs directly from `scheme`, `host`, `port`.

### How to set `global.config` (Example Function)

The `Skapa bruttolista – Larmia` tab includes a **“Set global.config”** Function you can trigger on deploy. It uses a pattern like:  

```javascript
const prev = global.get("config", "persistent") || {};
const deviceInfo = global.get("deviceInfo", "persistent") || {};
const deviceId = deviceInfo?.iotEdge?.deviceId || "edge-01";

const cfg = {
  scheme: "http",
  host: "10.0.0.1",     // CHANGE
  port: 80,             // CHANGE if needed
  user: "apiuser",      // CHANGE
  pass: "secret",       // CHANGE (or use env vars)
  JOB_UUID: "job2",
  property: "prop1",
  deviceId,
  pollingInterval: 60000,
  baseUrl: "http://10.0.0.1:80/api/v1"
};

global.set("config", cfg, "persistent");
msg.payload = { ok: true, saved: cfg };
return msg;
```

> **Security:** For production, prefer environment variables or a secret store and assemble `global.config` from `env` in this Function.

---

## 🧱 Gross List Flow – *Skapa bruttolista – Larmia*

This flow discovers all points from Larmia, parses and normalizes their metadata, and stores them in the **Gross List**.

### Flow Components

1. **Initialize config (on deploy)**  
   - Function: **Set global.config**  
   - Initializes or updates `global.config` as described above and logs the saved config to debug.  

2. **Refresh token (every 50 min)**  
   - Function: **Build POST /w-api/.../gettoken (from global.config)**  
   - Builds `POST {scheme}://{host}:{port}/w-api/v1/accounts/gettoken` with body `{ Username, Password }`.  
   - Adds `msg.ip = host` so the token can be associated with that controller instance.  
   - HTTP Request: **HTTP gettoken** sends the request and returns a JSON response.  
   - Function: **Store token in global**  
     - Extracts `payload.Key` from the response and stores it under:  
       `global.larmiaAuth.tokens[ip] = token` (persistent context).  
     - Outputs `{ ok: true, ip, tokenSaved: true }`.

3. **Refresh points (every 15 min)**  
   - Function: **Build GET /h-api/openapi/points (from global.config)**  
     - Reads `host, port, scheme` from `global.config`.  
     - Reads token from `global.larmiaAuth.tokens[host]`.  
     - Builds `GET {scheme}://{host}{port}/h-api/openapi/points` with `Authorization: Bearer <token>`.  
   - HTTP Request: **HTTP points**  
     - Calls the points endpoint, returns a JSON object with `payload.List` of points.  
     - Debug node: **Raw points response** for inspection.

4. **Save points to grossList (parsed-only, merge)**  
   - Parses `msg.payload.List` into a **parsed-only** structure per point:  
     - `PID` – point ID (string)
     - `Property` – extracted from `GroupName`
     - `system` – system portion from `GroupName`
     - `label` – full `PointName` (after stripping prefixes like `REG:`)
     - `Description` – human-readable description from `PointName`
     - `unit` – unit from the API
     - `component` – parsed component token (e.g., `GP41`, `ST21:1` → respecting `PREFER_BASE_COMPONENT`)
     - `"system id"` – sanitized `property_system` (alphanumeric only)
     - `"component id"` – `system id` + `_` + component (when both exist)
     - `"HASH"` – **pidPropertyHash** FNV-1a 32-bit hash of `"PID|Property"` (hex string)
     - `floor`, `zone`, `room` – placeholders (empty strings) for future enrichment
   - Stores these under:  
     `global.grossList.larmia_controller[ip].points[PID]`.  
   - **Mirrors** the same structure into:  
     `global.netList = global.grossList` (initial Net List identical to Gross List).  
   - Outputs a summary:  
     ```json
     {
       "ok": true,
       "ip": "10.0.0.1",
       "deviceId": "edge-01",
       "savedPoints": 123,
       "totalPoints": 123
     }
     ```

---

## 🧩 Net List – User-Managed Subset

The **Net List** represents the **streaming subset** of points derived from `global.grossList`.  

- After the first Gross List build, the template copies the parsed points to:  
  `global.netList.larmia_controller[ip].points[PID]`.  
- From there, **you** are expected to manage per-point flags such as:
  ```json
  {
    "toStream": true
  }
  ```
- Only points with `toStream === true` are:
  - Included in the `/s-api/openapi/currentvalue/list` polling requests.
  - Emitted as **telemetry** in the time-series flow.

You can maintain the Net List via:

- Node-RED Context Explorer (manually editing `global.netList`)
- Import/export via JSON or CSV + Function nodes
- External systems using Node-RED HTTP/API integrations

> The template does **not** auto-toggle `toStream`. The Net List is intentionally user-managed.

---

## 🧾 Metadata Flows

### 1. Send gross metadata – Larmia

Tab: **Send gross metadata – Larmia**  

- Inject → Function **Send metadata**
- Reads from `global.grossList[controller][ip].points` (default controller: `"larmia_controller"`).
- Walks all IPs and PIDs, flattens into a single array:
  ```json
  {
    "PID": "1234",
    "Property": "prop1",
    "system": "SYS1",
    "label": "SYS1-GP41 RoomTemp",
    "Description": "Room temperature",
    "unit": "°C",
    "component": "GP41",
    "system id": "prop1_SYS1",
    "component id": "prop1_SYS1_GP41",
    "HASH": "d2aea336",
    "job_uuid": "job2"
  }
  ```
- **Important:** During flattening, it **removes `hasValue`** from each point and adds `job_uuid` from `global.config.JOB_UUID`.  
- Optionally sorts the array by:
  1. `Property`
  2. `system`
  3. `PID` (numeric-aware)
- Outputs:
  - `msg.payload` – flat array of metadata objects
  - `msg.count` – number of rows

You can connect any output node (HTTP, MQTT, DB, etc.) after the debug node.

---

### 2. Send net metadata – Larmia

Tab: **Send net metadata – Larmia**  

- Inject → Function **Send metadata**
- Reads from `global.netList[controller][ip].points`.
- Flattening logic is identical to the Gross List metadata flow:
  - Removes `hasValue`
  - Adds `job_uuid`
  - Optional sort by `Property`, `system`, `PID`
- Output is the **current Net List** (typically the points you intend to stream), independent of whether `toStream` is `true` or not – you can filter downstream if desired.

---

## ⏱️ Time-Series Polling Flow – *Tidsseriedata – Larmia*

This tab handles **live polling** from Larmia and emits normalized telemetry.  

### 1. Build POST /s-api/openapi/currentvalue/list from netList

Function: **Build POST /s-api/openapi/currentvalue/list from netList**

Responsibilities:

- Reads `scheme`, `host`, `port`, `JOB_UUID` from `global.config`.
- Reads points from:  
  `global.netList.larmia_controller[ip].points`.
- Builds an `items` array **filtered to**:
  ```js
  p && p.toStream === true
  ```
- From these items, constructs a unique list of string PIDs: `PIDList: [ "1234", "5678", ... ]`.
- Reads token from `global.larmiaAuth.tokens[ip]`; errors if missing.
- Configures `msg` for POST:
  ```json
  {
    "method": "POST",
    "url": "http://<ip>/s-api/openapi/currentvalue/list",
    "headers": {
      "Accept": "application/json",
      "Content-Type": "application/json",
      "Authorization": "Bearer <token>"
    },
    "payload": { "PIDList": ["1234", "5678", "..."] },
    "items": [ /* selected points with toStream === true */ ],
    "job_uuid": "job2"
  }
  ```

> If no points with `toStream === true` are found, the node logs a warning and returns `null`.

---

### 2. HTTP POST currentvalue/list

Node: **HTTP POST currentvalue/list**

- Sends the POST request built above.
- Expects a JSON response with `payload.ValueList`, where each entry typically contains:
  - `PID`
  - `Value`
  - `Time` (timestamp from Larmia)

---

### 3. Merge values + build telemetry message

Function: **Merge values + build telemetry message**  

Responsibilities:

1. **Merge response values into Net List**
   - Takes `msg.items` (points from Net List used in the request).
   - Takes `msg.payload.ValueList` (rows from `/currentvalue/list`).
   - Builds a lookup `PID → { latestRead, value }`.
   - Reads `global.netList.larmia_controller[ip].points`.
   - For each item:
     - Updates `points[PID].latestRead` from `Time`.
     - Updates `points[PID].hasValue` from `Value`.
   - Writes updates back to `global.netList` in persistent context.

2. **Build telemetry items**
   - Iterates over points in `global.netList.larmia_controller[ip].points`.
   - **Only includes points that:**
     - Have `toStream === true` (as per your latest edits), and
     - Have a `hasValue` property.
   - For each such point, constructs:
     ```json
     {
       "job_uuid": "job2",
       "ip": "10.0.0.1",
       "pid": "1234",
       "label": "SYS1-GP41 RoomTemp",
       "dataType": "Number",
       "value": 21.7,
       "unit": "°C",
       "latestRead": "2025-11-11T08:30:00.000Z"
     }
     ```
     - `dataType` is inferred from JS type of `hasValue`:
       - `Number`, `Boolean`, or default `String`.
     - `latestRead` is ISO-8601; uses now as fallback if missing.

3. **Wrap in a telemetry envelope**
   - Final `msg.payload`:
     ```json
     {
       "messageType": "telemetry",
       "source": "larmia",
       "ip": "10.0.0.1",
       "job_uuid": "job2",
       "items": [ /* telemetryItems */ ]
     }
     ```

---

### 4. SET msg.input = "larmia"

Change node: **SET msg.input = "larmia"**

- Sets `msg.input = "larmia"`.
- Ensures `msg.payload.messageType = "telemetry"` for downstream consistency.

This gives a stable **input name** used by the shared filter utilities.

---

### 5. Normalize & annotate

Function: **Normalize & annotate**  

Responsibilities:

- **Numeric normalization**
  - If `msg.payload.value` is numeric (or numeric string), round to 2 decimals.
- **Input & measuredValueType**
  - Ensures `msg.payload.input` is set (from `msg.input` or `"unknown"`).
  - If `msg.payload.measuredValueType` is missing, sets it to `inputName`.
- **Time handling**
  - Normalizes `msg.payload.time`:
    - If a valid time string → converts to ISO.
    - Else → uses current time as ISO.
  - Mirrors to `msg.payload.timestamp`.
- **Incoming counters**
  - Increments `incoming_count:<inputName>` in persistent flow context for basic throughput stats.

All subsequent filters operate on this normalized envelope.

---

### 6. Drop: older than youngest (per HASH, 2 outputs)

Function: **Drop: older than youngest (per HASH, 2 outputs)**

Purpose:

- Implements a **time-gate** to prevent replay or out-of-order data from being delivered.
- Maintains the latest accepted timestamp for each key (typically the **HASH** for a point), per input.

Key aspects:

- Config:
  - `SCOPE = "per_hash"` – state is tracked per `(inputName, hash)`.
  - `MAX_LEN = 4000`, `COMPACT_TO = 1500` – rolling array with periodic compaction to limit memory usage.
- Resolves a key via `keyFromMsg(msg)`:
  - Reads `cfg.host` and `payload.PID/pid/topic`.
  - Looks up `global.netList.larmia_controller[ip].points[PID].HASH`; falls back to `payload.hash` / `uuid` / `id` / `pointId` / `pid` / `"<input>:unknown"` if `HASH` is missing.  
- Converts incoming `payload.time` or `payload.timestamp` to milliseconds.
- Compares `incomingMs` with the last accepted timestamp for that key:
  - If **older** → drops message to the **second output** with a small audit payload:
    ```json
    {
      "outcome": "drop",
      "reason": "OLDER_THAN_YOUNGEST",
      "incomingIso": "...",
      "lastAcceptedIso": "...",
      "input": "larmia",
      "scope": "per_hash",
      "key": "<hash or synthetic>",
      "hash": "<hash>",
      "pid": "<pid>"
    }
    ```
  - If valid and newer/same → updates internal state and passes the message on the **first output**.

Counters:

- `dropped_invalid_time` – global invalid time counter.
- `dropped_older_than_youngest:<inputName>` – per-input drop counter.
- `accepted_count:<inputName>` – per-input accepted counter.

---

### 7. Filter: change / keepalive

Function: **Filter: change / keepalive**  

Purpose:

- Avoids spamming unchanged values while still sending **keepalive** updates periodically.

Logic:

- Identifies each point using `keyFromMsg(msg)` (same semantics as the time-gate).
- Writes resolved `hash` and `pid` back into `msg.payload` so downstream nodes see them.
- Maintains a per-input map in `flow.get("valueStore", "persistent")`:
  ```js
  {
    [inputName]: {
      [hash]: {
        lastValue: <value>,
        lastSent: <timestamp_ms>
      }
    }
  }
  ```
- On each message:
  - If `payload.value` is **different** from `lastValue`:
    - Update `lastValue` and `lastSent`.
    - **Forward** message.
  - Else if `now - lastSent >= TIMEOUT` (default: 1 hour):
    - Update `lastSent`.
    - **Forward** message as keepalive.
  - Else:
    - **Drop** message (no output).

This filter runs after the time-gate, so it only sees time-consistent data.

---

### 8. Count outgoing + Telemetry payload debug

- Function: **Count outgoing**
  - Increments `outgoing_count:<inputName>` in persistent flow context.
  - Lets you compare incoming vs outgoing counts for each input.  

- Debug node: **Telemetry payload**
  - Outputs the final `msg.payload` telemetry envelope after all guards.

Attach your chosen transport node (HTTP, MQTT, DB, etc.) after **Count outgoing**.

---

## 🧮 Data Normalization Rules

### Point Metadata (Gross / Net List)

From the Gross List builder:  

| Field           | Description                                        | Source / Logic                       |
|----------------|----------------------------------------------------|--------------------------------------|
| `PID`          | Point ID                                           | Larmia `PID`                         |
| `Property`     | Property/site name                                 | Derived from `GroupName`             |
| `system`       | System identifier                                  | Derived from `GroupName`             |
| `label`        | Original (cleaned) `PointName`                     | API                                  |
| `Description`  | Parsed textual description                         | Parsed from `PointName`              |
| `unit`         | Unit                                               | API `Unit`                           |
| `component`    | Parsed component token (e.g. `GP41`)               | Parsed from `PointName`              |
| `"system id"`  | Sanitized `property_system`                        | Alphanumeric only                    |
| `"component id"` | `system id` + `_` + component                    | Derived                              |
| `"HASH"`       | FNV-1a hash of `PID|Property` (32-bit hex)         | Deterministic                        |
| `floor/zone/room` | Reserved for spatial metadata                   | Currently empty strings              |
| `toStream`     | Flag indicating the point should be streamed       | User-maintained in `global.netList`  |
| `job_uuid`     | Integration/job identifier                         | Added in metadata flows              |

### Telemetry

After merging live values and filters, each telemetry item follows roughly:

```json
{
  "job_uuid": "job2",
  "ip": "10.0.0.1",
  "pid": "1234",
  "label": "SYS1-GP41 RoomTemp",
  "dataType": "Number",
  "value": 21.73,
  "unit": "°C",
  "latestRead": "2025-11-11T08:30:00.000Z",
  "hash": "d2aea336",     // exposed via filters
  "timestamp": "2025-11-11T08:30:00.000Z",  // from Normalize & annotate
  "input": "larmia",
  "measuredValueType": "larmia"
}
```

The top-level message sent from the time-series flow:

```json
{
  "messageType": "telemetry",
  "source": "larmia",
  "ip": "10.0.0.1",
  "job_uuid": "job2",
  "items": [ /* telemetry item array */ ]
}
```

---

## 🧱 Responsibilities

| Stage                      | Provided by Template | User Responsibility |
|---------------------------|----------------------|---------------------|
| Config structure          | ✅ Example Function  | ✅ Fill real values / secure secrets |
| Token management          | ✅ (gettoken + store)| — |
| Gross List creation       | ✅                   | — |
| Parsing & normalization   | ✅                   | — |
| Net List creation/update  | Initial mirror only  | ✅ Maintain `global.netList` & `toStream` |
| Metadata output           | ✅ (gross + net flows) | ✅ Attach output node |
| Time-series polling       | ✅                   | ✅ Ensure `toStream` flags are correct |
| Filters & guards          | ✅ (time-gate + change/keepalive) | — |
| Final transport           | —                   | ✅ HTTP/MQTT/DB/etc. |

---

## 🚀 Deployment Steps

1. **Enable persistent context** in Node-RED (`settings.js → contextStorage`).
2. **Import the flows** (all four tabs) into your workspace.
3. **Configure `global.config`** using the “Set global.config” Function or equivalent.
4. **Trigger “Refresh token”** to obtain and store a Larmia token in `global.larmiaAuth`.
5. **Trigger “Refresh points”** once to build the Gross List and initial Net List.
6. **Adjust your Net List**:
   - Edit `global.netList.larmia_controller[ip].points[PID].toStream = true` for points you want to stream.
7. **Attach outputs**:
   - Add HTTP/MQTT/DB nodes after:
     - Metadata debug nodes (for metadata export)
     - `Count outgoing` / `Telemetry payload` (for telemetry streaming).
8. **Deploy** and watch the Debug pane:
   - Confirm metadata lists.
   - Confirm telemetry payloads with expected shape and volume.

---

## 🧩 Troubleshooting

| Symptom | Likely Cause | Suggested Fix |
|---------|--------------|---------------|
| **“Missing global.config”** | Config not initialized | Trigger the **Set global.config** node once, then redeploy. |
| **No token / “No token for host”** | `global.larmiaAuth.tokens[host]` empty | Run the **Refresh token** flow and verify credentials. |
| **Gross List empty** | Points endpoint unreachable or token invalid | Check `/h-api/openapi/points` debug output + HTTP status code. |
| **No telemetry emitted** | No points with `toStream === true` or empty ValueList | Set `toStream` to `true` for at least one point and ensure `/s-api/openapi/currentvalue/list` returns values. |
| **Messages dropped by time-gate** | Out-of-order timestamps | Inspect audit output of the Drop node; verify time source in Larmia and gateway. |
| **Too many unchanged values** | TIMEOUT too small for your use case | Adjust `TIMEOUT` in **Filter: change / keepalive** or rely more on change detection. |

---

## 📊 Flow Overview

```text
+-----------------------------+
|    Skapa bruttolista        |
|  (config + token + points)  |
+-------------+---------------+
              |
              v
      +-----------------+
      |  Gross List     |
      |  (global.gross) |
      +--------+--------+
               |
               v
      +-----------------+
      |  Net List       |
      | (global.net)    |
      +---+--------+----+
          |        |
          |        |
          |        +--------------------+
          |                             |
          v                             v
+--------------------+         +----------------------+
| Send gross metadata|         | Send net metadata    |
+--------------------+         +----------------------+

          |
          v
+------------------------------+
|   Tidsseriedata – Larmia     |
|  (poll → merge → filters →   |
|   telemetry)                 |
+------------------------------+
```

---

## 📚 License & Usage

This Node-RED integration template is provided as a **reference implementation** for Propvalue partners and customers integrating with **Larmia**.  
You’re free to adapt, extend, and deploy it in your own environments, subject to your internal policies and agreements.
