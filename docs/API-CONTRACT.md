# Atlas Green — API Contract (v1.3)

> **This document is authoritative.** `atlas-agent` and `atlas-console` must both
> implement it exactly. To change the contract: edit this file first, bump the
> version, then update both sides. Never let the two implementations drift.
> **When this file is bumped, verify both child `AGENTS.md` files still match, and
> search the file for the old `X-Atlas-Contract` value and update every occurrence.**
>
> **v1.1 (2026-06)** folded in the first multi-model review: UUID `batchId` +
> `batches` receipt + atomic ingest; capture-health, `sessionId`, `urlCaptureMethod`,
> `employeeId`, title caps; config ranges, clock-skew, rate limits.
>
> **v1.2 (2026-06, iteration-2 review)** corrects/clarifies: dedup uniqueness lives
> only on `batches` (not `intervals`); enrollment **rotates** on token loss;
> `/heartbeat` accepts a **revoked** token (→ `revoked`) and **never returns 426**;
> device `status` becomes **liveness + flags**; `/enroll` rate limit relaxed for the
> office NAT; `durationSec` is canonical; device timezone + `configSchemaVersion`
> added to heartbeat; `dropped.byState`. The URL path stays `/api/v1`.
>
> **v1.3 (2026-06, iteration-3 review)** — precision pass (no architectural change):
> wire header bumped to **`1.3`** (`426`/deprecation compares the **major** only);
> `/enroll` gains an optional `Authorization` so no-op and rotation are decidable;
> `archive` defined as a reversible ingest-blocking "retire" (distinct from revoke
> and block-&-wipe); revoked-token heartbeat **freezes capture-health** and the
> dashboard shows `revoked` as the primary badge; `reauthorized` is **sticky until
> the agent acts** and resumes `/ingest` before re-enrolling; `401` recovery is
> **scoped by endpoint**; `batches` TTL is **tied to `bufferCapDays`**; new `capture.
> enabled`, and `tamper`/`config_mismatch` flags; `contractDeprecated` trigger,
> `health_warning` thresholds, and the block-&-wipe admin endpoint are specified.

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
- **Contract version** is the **wire** version `major.minor` (currently **`1.3`**),
  independent of any doc revision number. It is echoed by the server in every response
  (header `X-Atlas-Contract: 1.3`) and sent by the agent as `X-Atlas-Contract: 1.3`.
  The mismatch check compares the **major component only** — a minor bump
  (1.1→1.2→1.3) is additive and never triggers it. On a **major** mismatch the server
  returns `426 Upgrade Required` (for `/enroll` and `/ingest` only — see §5).

---

## 1. `POST /enroll`

Called **once** on first run (and again only if the device token is lost/revoked).

**Request headers**
```
X-Atlas-Org-Key: <shared org key>
Authorization: Bearer <existing device token>   // OPTIONAL — sent only if the agent still has one
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
  "timezone": "Asia/Dhaka",                           // device IANA tz (mutable attribute)
  "machineHints": {                                   // informational only in v1 (no dedup)
    "machineGuid": "…"                                // read by the SERVICE (LocalSystem); null if unavailable
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
- Enrollment is **idempotent on `deviceId`**, with **token-loss recovery by
  rotation** (v1.2/1.3). The agent **sends `Authorization: Bearer <token>` iff it
  still has a stored token**, which makes no-op and rotation decidable:
  - **`Authorization` present and the token hash matches a valid, non-revoked row
    bound to this `deviceId`** → **no-op**: return the device record, same token
    stays valid (an agent that re-enrolled defensively keeps its token).
  - **`Authorization` absent or invalid/unknown** — the normal case after a
    lost/corrupted DPAPI blob — → **rotate**: mark the prior `deviceTokens` row
    `supersededAt` and issue a **new** token. (Tokens are stored hashed, so the
    server cannot "return the existing" one; rotation is safe because batches are
    keyed by `deviceId`+`batchId`, not token era, so buffered rows still sync.)
  - Re-enroll attempts are **strictly sequential** (never two `/enroll` in flight).
    A no-`Authorization` enroll arriving within a short window of a *still-unused*
    freshly-minted token returns **that same token** rather than rotating again, so a
    network-blip retry can't burn two rotation cycles (the double-rotation race).
- **Auth-check order**: validate the **org key first** (`401` if invalid), **then**
  the device block (`403` if blocked/revoked/archived), **then** the optional
  `Authorization` — so a caller without a valid org key can't probe block status.
  `/enroll` must not silently un-revoke a blocked or archived device.
- The server may hold **multiple active org keys** (rotation with a transition
  window); any currently-valid key is accepted.
- **Config validation**: the agent **clamps/ignores** out-of-range config values
  (keeps the previous value, logs a warning). The ranges above are authoritative.

**Errors**: `401` (bad org key / token-device mismatch), `403` (blocked, revoked, or
archived device), `400` (malformed), `426` (major contract mismatch on enroll/ingest
only), `429` (rate limit), `5xx`.

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
- **Atomic per batch.** The server writes the `batchId` receipt into the **`batches`
  collection** and inserts all intervals in **one MongoDB transaction** (≤ 1000
  intervals/txn) — all or nothing. There is no partial accept.
- **Idempotency uniqueness lives on `batches` only** — `_id = batchId` (a UUIDv4 is
  already globally unique, so no compound unique index is needed; `deviceId` is stored
  for scoping with a non-unique `{deviceId, receivedAt}` index for purge queries). On
  `intervals` the `{deviceId, batchId}` index is **non-unique** — every interval in a
  batch shares the same `batchId`, so a unique index there would reject all but the
  first. A retry of an already-received batch returns the original `accepted` with
  `duplicate: true`, re-inserting nothing. The `batches` collection is **retained
  independently** of `intervals`, so a stale replay of a batch whose intervals an
  admin already deleted is **rejected as a duplicate**, not resurrected — *within the
  receipt TTL*. The TTL must **exceed the fleet's max `bufferCapDays`**:
  `batchReceiptDays = max(180, maxBufferCapDays + 45)`, stored as a per-receipt
  `expireAt` (TTL on `expireAt`) so raising `bufferCapDays` at runtime extends it. The
  agent correspondingly **drops unsynced rows older than `min(bufferCapDays,
  serverBatchesTtl)`** (with a `dropped` report) so a long-offline or crash-corrupted
  buffer can't replay a batch past its evicted receipt and duplicate data.
- `durationSec` (from the agent's monotonic clock) is the **canonical** duration;
  consumers must **not** derive duration from `endUtc − startUtc` (they can differ
  after a clock correction). Server logs a warning if they diverge by > 5 s.
- The agent verifies the response `batchId` matches, then marks rows `synced` and
  purges — **only** after a 200 with a count.
- The server resolves each interval's **`category`** (admin rules) **and**
  **`employeeId`** (current device→employee mapping) at ingest and stores both.
- **Size cap ≤ 5000 intervals / ≤ 5 MB decompressed.** The agent **self-splits
  before sending** when over the cap (fresh UUID per sub-batch); `413` is the
  fallback if the server still rejects, and a sub-batch that still `413`s is split
  again (recursive halving). Sub-batches obey the same rate limit (don't fire them
  all at once).

**Errors**: `401` (bad/revoked/**archived** token, or token↔deviceId mismatch — a
revoked *or* archived device is ingest-blocked), `400` (malformed/bad gzip /
`durationSec` out of bounds), `413` (batch too large — split), `426` (major contract
mismatch), `429` (rate-limited — honor `Retry-After`), `5xx` (agent keeps rows, backs
off with jitter), `404` (device doc deleted — re-enroll once).

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
    "enabled": true,                // false when capture is paused (revoked/archived) — dashboard mutes the panel
    "helperRunning": true,          // service's view: pipe connected + recent sample
    "lastSampleUtc": "2026-06-22T09:14:58.000Z",
    "helperStartedAtUtc": "2026-06-22T09:00:11.000Z",
    "helperRestartCount": 0
  },
  "tamperHints": ["powertoys_awake"],  // known keep-awake/jiggler processes seen (→ server tamper flag); omit/[] if none
  "dropped": {                      // present only when the buffer cap forced a drop
    "count": 0, "oldestUtc": null, "newestUtc": null,
    "byState": { "active": 0, "passive_media": 0, "idle": 0 }
  },
  "updateStatus": "idle",           // "idle"|"downloading"|"verifying"|"success"|"failed"
  "attributes": {                   // sent on first heartbeat + when a value changed
    "pcName": "PC-07", "hostname": "DESK-RAVI", "osUsername": "ravi",
    "timezone": "Asia/Dhaka", "machineGuid": "…"   // tz + machineGuid for clone detection
  }
}
```
> First heartbeat after enrollment or restart sends `attributes`/`os` unconditionally;
> thereafter only when a value differs from the last sent (persisted in an
> `agent_state` row so a restart isn't treated as a baseline reset).

**Response 200**
```jsonc
{
  "serverTimeUtc": "2026-06-22T09:15:00.000Z",
  "liveness": "online",             // "pending" (enrolled, not yet seen) | "online" | "offline"; informational for the agent
  "flags": [],                      // ALWAYS an array; e.g. ["outdated","clock_skewed","revoked","archived","health_warning","possible_clone","tamper","config_mismatch"]
  "reauthorized": false,            // sticky true after admin un-revoke/un-archive, until the agent acts (a successful /ingest)
  "contractDeprecated": false,      // server-set when settings.contractDeprecation marks the agent's MAJOR "warn"/"reject"; never true for the current major
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
- `/heartbeat` **always returns `200`** (it is the recovery lifeline) — including for
  a **revoked or archived** token (response carries `flags:["revoked"]` / `["archived"]`;
  the agent pauses capture, keeps its buffer) and including across a major contract
  bump (`426` is for `/enroll` and `/ingest` only; here set `contractDeprecated:true`).
  A token↔deviceId mismatch is still `401`; a valid token whose **device doc was
  deleted** → `404`.
- **Revoked/archived heartbeat is restricted**: it updates `lastSeenAt` (so the device
  shows online, not falsely "offline") **but must not refresh capture-health**
  (`capture.*`, `helperRunning`, `lastSampleUtc`) or clear `health_warning`/
  `possible_clone` — those **freeze at the revoke/archive instant**, so a leaked
  revoked token can't spoof a healthy/online device (the silent-agent backstop, #17,
  stays intact). The **dashboard renders `revoked` (then `archived`) as the primary
  badge, overriding `liveness`** — "online + revoked" displays as "revoked".
- Server sets `liveness` (`pending` until first interval/seen, then `online`;
  `offline` after `heartbeatIntervalSec × 3`, default 15 min) and `flags`
  (`outdated`/`clock_skewed`/`revoked`/`archived`/`health_warning`/`possible_clone`/
  `tamper`/`config_mismatch`) — triggers + clear conditions in `ARCHITECTURE.md` §7.
  `health_warning` = `cpuPct > 5` for **3 consecutive heartbeats** OR `memMb > 200` on
  any one (cleared after 2 normal); `config_mismatch` = the agent's
  `configSchemaVersion` is **below** the server's; `tamper` = the heartbeat reported
  `tamperHints`. The helper is "down" (`helperRunning:false`) when no sample has
  arrived for ~60 s.
- The server **persists** a non-zero `dropped` as a `dataGap` record (so the gap
  stays visible historically) and surfaces `helperRunning:false`/stale
  `lastSampleUtc` and `updateStatus:"failed"` in the heartbeat panel.
- Server applies mutable `attributes` to the device doc (identity unchanged). The
  agent's reported **`pcName` is informational — logged, not stored** (admins rename
  from the dashboard); `hostname`/`osUsername`/`timezone`/`machineGuid` are
  **agent-owned** and written to the doc. **Two or more distinct non-`null`**
  `machineGuid`s on one `deviceId` → `possible_clone` (a `null` from an intermittent
  read is ignored).
- On `reauthorized:true` the agent exits its paused state and **resumes `/ingest`
  with its existing (now-valid) token first** — it re-enrolls **only** if that still
  returns `401`. Because `reauthorized` is sticky until a successful `/ingest`, a
  missed heartbeat can't strand recovery. **`401` recovery is scoped by endpoint**: a
  `401` from **`/ingest`** → a single re-enroll with the org key (the same-identity
  recovery); a `401` from **`/heartbeat`** (token↔deviceId mismatch / superseded) or a
  `404` (deleted doc) → **re-enroll as a new device**. The agent applies returned
  `config`; **out-of-range values are ignored** (see §1); a `configSchemaVersion`
  newer than it supports → ignore-unknown + log. It computes `clockOffset =
  serverTimeUtc − localUtc` and flags a skew (see §5). On a heartbeat failure it
  **retries once after ~30 s**, else waits for the next cycle.

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
- **Swap is crash-safe**: download to a staging dir, write a `pending-update` marker,
  rename on service `OnStop`. **On every startup** the service checks the marker and,
  if the staged package still verifies, completes the rename before the main loop;
  otherwise it clears the marker and reports `updateStatus:"failed"`.
- `mandatory:true` means update **as soon as possible** (next heartbeat cycle, not
  the regular poll). On repeated failure the agent reports `updateStatus:"failed"`
  and **keeps running the current version** — it never shuts off ingest (never lose
  data). `agentVersion` comparison uses **semver** on `AssemblyInformationalVersion`.
- A failed verification aborts the update (keeps the current version), reported via
  `updateStatus:"failed"`.
- Below `minSupportedVersion`: the server still **accepts ingest** but flags the
  device `outdated`. `latestVersion < current` is a valid **downgrade** (emergency
  rollback). **R2 retains the last 5 versions** (+ each signed manifest) so a
  downgrade target is never 404; CI enforces this.

---

## 5. Conventions

- **Versioning**: the path carries the major version (`/api/v1`); `X-Atlas-Contract`
  carries the wire `major.minor` (`1.3`). Breaking changes → `/api/v2`; the server
  returns `426` only on a **major** mismatch and **only for `/enroll` and `/ingest`**
  — minor bumps (1.x) are additive and never 426 — while **`/heartbeat` always 200**
  (with `contractDeprecated:true` + the `update` block) so any agent can self-update.
  `contractDeprecated` is driven by `settings.contractDeprecation: { [major]: "warn" |
  "reject" }` and is never set for the current major.
- **Clock skew**: the agent stamps timestamps from its own UTC clock and **never
  rewrites already-buffered timestamps**, but `durationSec` comes from a **monotonic
  clock** and is **canonical** (don't derive duration from `endUtc − startUtc`). It
  computes `clockOffset = serverTimeUtc − localUtc` each heartbeat; if `|offset| >
  5 min` it logs a warning, and if large+stable across two heartbeats the device is
  flagged `clock_skewed`. The stored offset is applied at **both display and
  day-bucketing**; buffered raw data stays untouched.
- **Rate limits** (server-enforced, per device unless noted): `/ingest` ≤ 1 / 30 s,
  `/heartbeat` ≤ 1 / 60 s, `/enroll` ≤ **10 / min / IP** (a corp-CIDR allowlist in
  `settings` is exempt, so the manual 100-PC rollout behind one office NAT isn't
  throttled); global ≤ ~200 req/s / IP. Over-limit → `429` with `Retry-After`.
- **Payload size**: keep ingest batches under the hard cap (≤ 5000 intervals or
  ≤ 5 MB decompressed). If `413`, split into fresh-UUID sub-batches and retry.
- **Backoff & jitter**: failed sync retries use exponential backoff **with ±jitter**
  (so 100 agents recovering after an outage don't thunder-herd in lockstep); the
  5-min sync timer also carries ±30 s jitter.
- **Auth failures**, scoped by endpoint: a **`401` from `/ingest`** (revoked/invalid
  token) → the agent attempts a **single re-`/enroll`** with the org key (sending its
  current token as `Authorization` if it still has one). If `/enroll` returns `403`
  (blocked/revoked/archived) the agent **pauses capture, keeps its buffer, and only
  heartbeats** (`flags` reflect `revoked`/`archived`) until a `reauthorized:true`. If
  `/enroll` returns **`401` (bad org key — the MSI's key may have been rotated, *not*
  a block)**, the agent logs a critical warning and **keeps heartbeating with its
  existing token** (which is valid again after an un-revoke) rather than entering the
  paused state. A **`401` from `/heartbeat`** (token↔deviceId mismatch / superseded)
  or a **`404`** (deleted doc) → the agent **re-enrolls as a new device**. Re-enrolls
  are sequential and capped (~5 / hour) to avoid a loop draining the org-key budget.
- **Secrets**: the org key lives only in the MSI/agent and is used only at enroll;
  device tokens are stored hashed server-side and DPAPI-encrypted client-side.

---

## 6. Admin actions (dashboard → server)

These are **admin** endpoints (behind Cloudflare Access + `middleware.ts`, not agent
Bearer auth); shapes are owned by the console but the security-relevant semantics are
fixed here. Every action writes an `auditLog` entry **before** any destructive step.

- **Revoke / un-revoke** a device — sets/clears `deviceTokens.revokedAt` + the
  `revoked` flag (token dies / self-heals via `reauthorized`).
- **Archive / un-archive** a device — sets/clears `archived` + the `archived` flag
  (ingest-blocked but token stays valid; reversible — Decision #49).
- **`POST /api/v1/admin/devices/:deviceId/wipe`** (block & wipe, Decision #48) —
  atomically: revoke the token, set `blocked` **permanently**, and delete the device's
  `intervals`, `batches` receipts, and `dataGaps`. The minimal device doc is
  **retained** (so the `deviceId` can't re-enroll) as a tombstone; response
  `{ deviceId, deletedIntervals, deletedBatches, revokedAt }`. Irreversible.
- All four go through the single `updateDeviceState` path so `liveness`/`flags` stay
  consistent with the source booleans (Decision #39).
