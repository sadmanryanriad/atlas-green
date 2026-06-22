# Atlas Green — API Contract (v1.1)

> **This document is authoritative.** `atlas-agent` and `atlas-console` must both
> implement it exactly. To change the contract: edit this file first, bump the
> version, then update both sides. Never let the two implementations drift.
>
> **v1.1 (2026-06)** folds in the multi-model review: `batchId` is now a UUID with
> a `batches` receipt collection and atomic ingest; capture-health, `sessionId`,
> `urlCaptureMethod`, `employeeId`, and title caps are added; enrollment, config
> ranges, clock-skew, and rate limits are specified. The URL path stays `/api/v1`
> (no breaking change to routes).

- **Base URL**: `https://<console-host>/api/v1`
- **Transport**: HTTPS only.
- **Encoding**: request bodies are JSON; large bodies (ingest) are
  `Content-Encoding: gzip`.
- **Time**: every timestamp is **UTC**, ISO-8601 with `Z` (e.g.
  `2026-06-22T09:14:05.123Z`).
- **Auth**:
  - `/enroll` → header `X-Atlas-Org-Key: <org key>`.
  - everything else → header `Authorization: Bearer <device token>`.
  - The server **binds each token to its `deviceId`** at enrollment and **rejects**
    any `/ingest` or `/heartbeat` whose body `deviceId` ≠ the token's bound
    `deviceId` (`401`). A token can only ever submit its own device's data.
- **Contract version** is echoed by the server in every response (header
  `X-Atlas-Contract: 1.1`) and sent by the agent as `X-Atlas-Contract: 1.1`. On a
  major mismatch the server returns `426 Upgrade Required`.

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
    "machineGuid": "…"                                // mac dropped in v1.1 (multi-NIC, unused)
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
    "configSchemaVersion": 1,        // agent ignores unknown fields if this is newer
    "sampleIntervalSec": 4,          // valid [2,60]
    "idleThresholdSec": 300,         // valid [60,3600]
    "syncIntervalSec": 300,          // valid [60,3600]
    "heartbeatIntervalSec": 300,     // valid [60,3600]
    "bufferCapDays": 75              // valid [7,365]
  }
}
```

**Behavior**
- Enrollment is **idempotent on `deviceId`**: a re-enroll of an already-enrolled,
  non-revoked device returns the **existing** token (no rotation). Token rotation
  happens only via explicit admin revocation. (This avoids orphaning buffered rows
  tagged under an old token era.)
- If the device is **blocked/revoked** server-side, `/enroll` returns **`403`**
  (it must not silently un-revoke by issuing a fresh token).
- The server may hold **multiple active org keys** (rotation with a transition
  window); any currently-valid key is accepted. Invalid/missing org key → `401`.
- **Config validation**: the agent **clamps/ignores** out-of-range config values
  (keeps the previous value, logs a warning). The ranges above are authoritative.

**Errors**: `401` (bad org key / token-device mismatch), `403` (blocked device),
`400` (malformed), `426` (contract mismatch), `429` (rate limit), `5xx`.

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
  "batchId": "9b1c7e54-3a2f-4d18-9e6b-2c5a0f7d4a31",  // UUID v4, generated once per batch
  "agentVersion": "0.1.0",
  "intervals": [
    {
      "startUtc": "2026-06-22T09:02:15.000Z",
      "endUtc":   "2026-06-22T09:14:40.000Z",
      "durationSec": 745,                // 1 ≤ durationSec ≤ 86400 (24h); else rejected
      "state": "active",                 // "active" | "passive_media" | "idle"
      "appName": "Google Chrome",        // from the exe FileDescription; fallback = exe name
      "processName": "chrome.exe",
      "windowTitle": "10 hour rain sounds - YouTube",  // ≤ 512 chars (agent truncates)
      "sessionId": 1,                    // Windows session id the helper observed it in
      "browser": "chrome",               // null if not a browser
      "domain": "youtube.com",           // eTLD+1, normalized; null if not resolvable
      "pageTitle": "10 hour rain sounds", // ≤ 256 chars; null if not available
      "urlCaptureMethod": "uia",         // "uia" | "title-fallback" | "unavailable"
      "osUsername": "ravi"
    }
    // … more intervals
  ]
}
```

**Response 200**
```jsonc
{
  "batchId": "9b1c7e54-3a2f-4d18-9e6b-2c5a0f7d4a31",  // MUST equal the request batchId
  "accepted": 37,                 // agent purges only after this ACK
  "duplicate": false,             // true if this batchId was already received
  "serverTimeUtc": "2026-06-22T09:14:06.000Z"
}
```

**Behavior**
- **Atomic per batch.** The server records the `batchId` in a **`batches` receipt
  collection** and inserts all intervals in **one MongoDB transaction** — all or
  nothing. There is no partial accept.
- **Idempotent on `{deviceId, batchId}`** (strict unique index). A retry of an
  already-received batch returns the original `accepted` count with
  `duplicate: true`, without re-inserting. The `batches` collection is **retained
  independently** of `intervals`, so a stale offline replay of a batch whose
  intervals an admin already deleted is **rejected as a duplicate**, not resurrected.
- The agent verifies the response `batchId` matches, then marks rows `synced` and
  purges — **only** after a 200 with a count.
- The server resolves each interval's **`category`** (admin rules) **and**
  **`employeeId`** (current device→employee mapping) at ingest and stores both.
- **`413`**: the agent splits into sub-batches, **each with its own fresh UUID**
  `batchId`, and retries them independently. Hard cap: ≤ 5000 intervals or ≤ 5 MB
  decompressed per batch.

**Errors**: `401` (bad/revoked token, or token↔deviceId mismatch), `400`
(malformed/bad gzip / `durationSec` out of bounds), `413` (batch too large —
split), `426` (contract mismatch), `429` (rate-limited — honor `Retry-After`),
`5xx` (agent keeps rows, backs off with jitter).

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
  "configSchemaVersion": 1,         // so the server can spot an agent too old to update itself
  "os": "Windows 11 Pro 23H2",      // sent when changed; lets the server track OS across the fleet
  "queuedRows": 12,                 // unsynced rows still in the buffer
  "lastSyncUtc": "2026-06-22T09:10:06.000Z",
  "health": { "cpuPct": 0.4, "memMb": 38 },
  "capture": {                      // proves the HELPER (not just the service) is alive
    "helperRunning": true,
    "lastSampleUtc": "2026-06-22T09:14:58.000Z",
    "helperRestartCount": 0
  },
  "dropped": {                      // present only when the buffer cap forced a drop
    "count": 0,
    "oldestUtc": null,
    "newestUtc": null
  },
  "updateStatus": "idle",           // "idle"|"downloading"|"verifying"|"success"|"failed"
  "attributes": {                   // sent only when a mutable attribute changed
    "pcName": "PC-07", "hostname": "DESK-RAVI", "osUsername": "ravi"
  }
}
```

**Response 200**
```jsonc
{
  "serverTimeUtc": "2026-06-22T09:15:00.000Z",
  "config": {                       // same shape as /enroll (all remotely-tunable fields)
    "configSchemaVersion": 1,
    "sampleIntervalSec": 4, "idleThresholdSec": 300, "syncIntervalSec": 300,
    "heartbeatIntervalSec": 300, "bufferCapDays": 75
  },
  "update": {                       // null when up to date
    "latestVersion": "0.2.0",
    "manifestUrl": "https://<r2-host>/atlas-agent/manifest.json",
    "mandatory": false
  }
}
```

**Behavior**
- Server updates the device's `lastSeenAt` and `status` (see the device status
  state machine in `ARCHITECTURE.md` §7). Devices not seen within
  `heartbeatIntervalSec × 3` (default 15 min) are shown **offline**.
- The server surfaces `capture.helperRunning == false` / stale `lastSampleUtc`
  (helper dead though service online), a non-zero `dropped` (data gap), and
  `updateStatus == failed` in the dashboard heartbeat panel.
- Server applies mutable `attributes` to the device doc (identity is unchanged).
- The agent applies returned `config` changes; **out-of-range values are ignored**
  (see §1 ranges). It computes `clockOffset = serverTimeUtc − localUtc` and flags a
  skew (see §5). On a heartbeat failure it **retries once after ~30 s**, else waits
  for the next cycle.

---

## 4. Update manifest (and the package)

There is **one** update channel: the **heartbeat `update` block** tells the agent
*whether* a newer version exists and gives the `manifestUrl`; the **manifest on R2**
is the full spec. (There is no separate console `/update/manifest` route — it was
removed in v1.1 to avoid two sources of truth.)

**`manifest.json`** (served from R2)
```jsonc
{
  "latestVersion": "0.2.0",
  "package": {
    "url": "https://<r2-host>/atlas-agent/atlas-agent-0.2.0.pkg",
    "sha256": "…",
    "signature": "…",            // package signature, verified against the EMBEDDED public key
    "sizeBytes": 78123456
  },
  "manifestSignature": "…",        // detached signature OVER manifest.json itself
  "minSupportedVersion": "0.1.0",  // agents below this are flagged agent_outdated (ingest still accepted)
  "changelogUrl": "https://…",     // optional; lets IT see what changed
  "releasedUtc": "2026-07-01T00:00:00Z"
}
```

**Behavior**
- The agent **verifies `manifestSignature` first** (so a compromised R2 can't
  redirect it to a malicious `package.url`), then downloads the package with
  **HTTP Range/resume** support (Bangladesh links are flaky and the package is
  ~60–80 MB), verifies the package **signature + SHA-256**, then swaps and restarts.
  The public key is **compiled into the agent binary**, never fetched.
- A failed verification aborts the update (the agent keeps running the current
  version) and is reported via `updateStatus: "failed"` on the next heartbeat.
- Below `minSupportedVersion`: the server still **accepts ingest** (never lose data)
  but marks the device `outdated` so IT can intervene. `latestVersion < current` is
  a valid **downgrade** signal (emergency rollback), handled like any other update.

---

## 5. Conventions

- **Versioning**: the path carries the major version (`/api/v1`). Breaking changes
  → `/api/v2`; the agent sends `X-Atlas-Contract` and the server returns `426` on a
  major mismatch (the agent then surfaces it via heartbeat and relies on
  auto-update to recover).
- **Clock skew**: the agent stamps timestamps from its own UTC clock and **never
  rewrites already-buffered timestamps**, but it measures interval *durations* from
  a **monotonic clock** (so a wall-clock jump can't produce negative/inflated
  durations). It computes `clockOffset = serverTimeUtc − localUtc` each heartbeat;
  if `|offset| > 5 min` it logs a warning, and if it's large and stable across two
  heartbeats the device is flagged `clock_skewed` (the dashboard can apply the
  stored offset at display time). Buffered raw data stays untouched.
- **Rate limits** (server-enforced, per device unless noted): `/ingest` ≤ 1 / 30 s,
  `/heartbeat` ≤ 1 / 60 s, `/enroll` ≤ 1 / hour / IP; global ≤ ~200 req/s / IP.
  Over-limit → `429` with `Retry-After`, which the agent honors.
- **Payload size**: keep ingest batches under the hard cap (≤ 5000 intervals or
  ≤ 5 MB decompressed). If `413`, split into fresh-UUID sub-batches and retry.
- **Backoff & jitter**: failed sync retries use exponential backoff **with ±jitter**
  (so 100 agents recovering after an outage don't thunder-herd in lockstep); the
  5-min sync timer also carries ±30 s jitter.
- **Auth failures** (`401`): on a revoked/invalid device token the agent attempts a
  single re-`/enroll` with the org key. If `/enroll` returns `403` (blocked device)
  or re-enroll fails repeatedly, the agent stops generating new intervals, keeps its
  buffer, and only heartbeats (`status` reflects `revoked`).
- **Secrets**: the org key lives only in the MSI/agent and is used only at enroll;
  device tokens are stored hashed server-side and DPAPI-encrypted client-side.
