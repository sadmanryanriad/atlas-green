# Atlas Green — API Contract (v1)

> **This document is authoritative.** `atlas-agent` and `atlas-console` must both
> implement it exactly. To change the contract: edit this file first, bump the
> version, then update both sides. Never let the two implementations drift.

- **Base URL**: `https://<console-host>/api/v1`
- **Transport**: HTTPS only.
- **Encoding**: request bodies are JSON; large bodies (ingest) are
  `Content-Encoding: gzip`.
- **Time**: every timestamp is **UTC**, ISO-8601 with `Z` (e.g.
  `2026-06-22T09:14:05.123Z`).
- **Auth**:
  - `/enroll` → header `X-Atlas-Org-Key: <org key>`.
  - everything else → header `Authorization: Bearer <device token>`.
- **Contract version** is echoed by the server and may be sent by the agent as
  `X-Atlas-Contract: 1`.

---

## 1. `POST /enroll`

Called **once** on first run (and again only if the device token is lost/revoked).

**Request headers**
```
X-Atlas-Org-Key: <shared org key>
Content-Type: application/json
```

**Request body**
```jsonc
{
  "deviceId": "7f3a9c2e-1b4d-4e8a-9c2e-1b4d4e8a9c2e", // client-generated GUID
  "pcName": "PC-07",                                  // from MSI install param, optional
  "hostname": "DESK-RAVI",
  "osUsername": "ravi",
  "os": "Windows 11 Pro 23H2",
  "agentVersion": "0.1.0",
  "machineHints": {                                   // informational only in v1 (no dedup)
    "machineGuid": "…",
    "mac": "AA-BB-CC-DD-EE-FF"
  }
}
```

**Response 200**
```jsonc
{
  "deviceId": "7f3a9c2e-…",
  "deviceToken": "atg_live_…",      // store via DPAPI machine scope; sent as Bearer hereafter
  "serverTimeUtc": "2026-06-22T09:14:05.123Z",
  "config": {                        // agent reads remote-tunable settings from here
    "sampleIntervalSec": 4,
    "idleThresholdSec": 300,
    "syncIntervalSec": 300,
    "heartbeatIntervalSec": 300,
    "bufferCapDays": 75
  }
}
```

**Behavior**
- If `deviceId` is already enrolled, the server may return the existing record and
  (re)issue a token — enrollment is idempotent on `deviceId`.
- Invalid/missing org key → `401`.

**Errors**: `401` (bad org key), `400` (malformed), `429` (rate limit), `5xx`.

---

## 2. `POST /ingest`

The agent's main upload. Sends a batch of closed intervals.

**Request headers**
```
Authorization: Bearer <device token>
Content-Type: application/json
Content-Encoding: gzip
```

**Request body** (decompressed)
```jsonc
{
  "deviceId": "7f3a9c2e-…",
  "batchId": "b-2026-06-22T09:10:00Z-001",  // unique per batch; used for idempotency
  "agentVersion": "0.1.0",
  "intervals": [
    {
      "startUtc": "2026-06-22T09:02:15.000Z",
      "endUtc":   "2026-06-22T09:14:40.000Z",
      "durationSec": 745,
      "state": "active",                 // "active" | "passive_media" | "idle"
      "appName": "Google Chrome",
      "processName": "chrome.exe",
      "windowTitle": "10 hour rain sounds - YouTube",
      "browser": "chrome",               // null if not a browser
      "domain": "youtube.com",           // null if not a browser / not resolvable
      "pageTitle": "10 hour rain sounds", // null if not available
      "osUsername": "ravi"
    }
    // … more intervals
  ]
}
```

**Response 200**
```jsonc
{
  "batchId": "b-2026-06-22T09:10:00Z-001",
  "accepted": 37,                 // agent purges only after this ACK
  "duplicate": false,             // true if this batchId was already stored
  "serverTimeUtc": "2026-06-22T09:14:06.000Z"
}
```

**Behavior**
- **Idempotent on `batchId`**: if the agent retries a batch it already delivered,
  the server returns `accepted` for the same set and `duplicate: true`, without
  double-storing. This makes "ACK before purge" safe.
- The agent marks rows `synced` and purges them **only** after a 200 with a count.
- Server resolves each interval's `category` at ingest using the admin rules.

**Errors**: `401` (bad/revoked token), `400` (malformed/bad gzip), `413` (batch too
large — agent should split), `429`, `5xx` (agent keeps rows, backs off).

---

## 3. `POST /heartbeat`

Lightweight liveness ping; also the agent's update + config check.

**Request headers**
```
Authorization: Bearer <device token>
```

**Request body**
```jsonc
{
  "deviceId": "7f3a9c2e-…",
  "agentVersion": "0.1.0",
  "queuedRows": 12,                 // unsynced rows still in the buffer
  "lastSyncUtc": "2026-06-22T09:10:06.000Z",
  "health": { "cpuPct": 0.4, "memMb": 38 }
}
```

**Response 200**
```jsonc
{
  "serverTimeUtc": "2026-06-22T09:15:00.000Z",
  "config": { "sampleIntervalSec": 4, "idleThresholdSec": 300, "syncIntervalSec": 300 },
  "update": {                       // null when up to date
    "latestVersion": "0.2.0",
    "manifestUrl": "https://<r2-host>/atlas-agent/manifest.json",
    "mandatory": false
  }
}
```

**Behavior**
- Server updates the device's `lastSeenAt` and `status = online`. Devices not seen
  within ~3× the heartbeat interval are shown **offline**.
- The agent applies any returned `config` changes (remote tuning).

---

## 4. `GET /update/manifest` (and the package)

The agent fetches the manifest (URL given by heartbeat, or a fixed R2 URL).

**`manifest.json`** (served from R2)
```jsonc
{
  "latestVersion": "0.2.0",
  "package": {
    "url": "https://<r2-host>/atlas-agent/atlas-agent-0.2.0.pkg",
    "sha256": "…",
    "signature": "…",            // verified against the embedded/internal public key
    "sizeBytes": 78123456
  },
  "minSupportedVersion": "0.1.0",  // agents older than this must update
  "releasedUtc": "2026-07-01T00:00:00Z"
}
```

**Behavior**
- Agent downloads the package, verifies **signature** and **SHA-256**, then swaps
  its binary and restarts the service. A failed verification aborts the update
  (the agent keeps running the current version) and is reported on next heartbeat.

---

## 5. Conventions

- **Versioning**: the path carries the major version (`/api/v1`). Breaking changes
  → `/api/v2`; the agent sends `X-Atlas-Contract` so the server can detect mismatch.
- **Clock skew**: the agent trusts its own UTC clock for timestamps but reads
  `serverTimeUtc` to detect/sync drift; it never rewrites already-buffered
  timestamps.
- **Payload size**: keep ingest batches under a sane cap (e.g. a few MB
  decompressed). If `413`, split and retry.
- **Auth failures** (`401`): on a revoked/invalid device token the agent attempts
  a single re-`/enroll` with the org key; if that also fails it backs off and
  keeps buffering.
- **Secrets**: the org key lives only in the MSI/agent and is used only at enroll;
  device tokens are stored hashed server-side and DPAPI-encrypted client-side.
