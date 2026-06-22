# Atlas Green — Architecture

This document describes how the whole system fits together. For the exact wire
format see [API-CONTRACT.md](API-CONTRACT.md); for *why* each choice was made see
[DECISIONS.md](DECISIONS.md).

---

## 1. High-level picture

```
        ┌─────────────────────────────── Each Windows Desktop (×~100) ───────────────────────────────┐
        │                                                                                             │
        │    atlas-agent                                                                              │
        │   ┌───────────────────────┐        launches/supervises        ┌──────────────────────────┐ │
        │   │  Windows Service       │ ───────────────────────────────► │  User-Session Helper      │ │
        │   │  (Session 0, no UI)    │ ◄─────────────────────────────── │  (active desktop)         │ │
        │   │                        │      foreground samples           │                          │ │
        │   │  • supervises helper   │                                   │  • foreground window/proc │ │
        │   │  • SQLite buffer       │                                   │  • idle time (input)      │ │
        │   │  • sync loop (5 min)   │                                   │  • media state (GSMTC)    │ │
        │   │  • heartbeat           │                                   │  • browser URL via UIA    │ │
        │   │  • auto-update         │                                   │  • (v2) screenshots       │ │
        │   └───────────┬────────────┘                                   └──────────────────────────┘ │
        │               │ HTTPS (gzip JSON, Bearer device token)                                      │
        └───────────────┼─────────────────────────────────────────────────────────────────────────────┘
                        │
                        ▼
        ┌─────────────────────────────── VPS (behind Cloudflare) ───────────────────────────────┐
        │   atlas-console (Next.js)                                                              │
        │   ┌──────── API routes (Bearer token) ───────────┐     ┌─ Dashboard UI (Cloudflare ─┐ │
        │   │  POST /api/v1/enroll                          │     │  Access + middleware auth) │ │
        │   │  POST /api/v1/ingest                          │     │ timeline / summaries /     │ │
        │   │  POST /api/v1/heartbeat                       │     │ company overview /         │ │
        │   │  GET  /api/v1/health                          │     │ heartbeat panel /          │ │
        │   └───────────────────┬───────────────────────────┘     │ device↔employee / rules    │ │
        │                       │            (manifest lives on R2, not here)                       │
        │                       ▼                                                                    │
        │                 ┌───────────┐   devices · intervals · batches · categories · employees ·   │
        │                 │ MongoDB   │   deviceTokens · settings · auditLog                         │
        │                 └───────────┘   (single-node replica set → transactions)                  │
        └─────────────────────────────────────────────────────────────────────────────────────────┘
                        ▲
                        │ signed update packages + manifest
                ┌───────────────┐
                │ Cloudflare R2 │   agent-vX.Y.Z.pkg  ·  manifest.json  ·  (v2) screenshots
                └───────────────┘
```

---

## 2. The agent: why two parts

A Windows **Service** runs in **Session 0**, isolated from the interactive
desktop. It is perfect for persistence (starts at boot, survives logout, hard for
a standard user to kill) but it **cannot see the user's screen, foreground window,
or input** directly. So:

- **Service** (the persistent brain): owns the SQLite buffer, the 5-minute sync
  loop, the heartbeat, auto-update, and supervises the helper. Protected by ACLs
  and SCM auto-restart.
- **User-Session Helper** (the eyes): a lightweight process the service launches
  into the active user session. It samples the foreground window/process, idle
  time, media state, and the browser URL, and streams those samples back to the
  service (e.g., via a local named pipe). It is restarted by the service if it
  dies or when a new user logs in.

The helper holds **no secrets** and does **no networking** — it only observes and
reports to the service. All buffering, auth, and uploading live in the service.

### 2.1 How the service launches the helper (Session 0 → user desktop)

A Session-0 service cannot `CreateProcess` straight onto the interactive desktop.
The mechanism (locked, Decision #32):

- The service runs as **LocalSystem** (it needs `SE_TCB` to use a user token).
- It calls **`WTSGetActiveConsoleSessionId`** → **`WTSQueryUserToken`** →
  **`CreateProcessAsUser`** with that token, so the helper runs **as the logged-on
  user** (not SYSTEM — UIPI/integrity would otherwise block UIA reads of the
  browser).
- It registers for **`WTS_SESSION_CHANGE`** (`WTSRegisterSessionNotification`) and
  relaunches/tears down the helper on logon/logoff and **fast user switching**.
- **v1 monitors the console (physical) session only.** RDP sessions are ignored
  (one employee, one desktop). When no user is logged in, the service still
  heartbeats from Session 0 but produces no intervals.
- A **watchdog** restarts a dead helper; repeated rapid restarts raise
  `helperRestartCount` (surfaced via heartbeat — see §6).

### 2.2 Helper → service transport (named pipe)

One-way **helper → service** over a local named pipe, **length-prefixed JSON**
(4-byte little-endian length header + UTF-8 JSON body), max 64 KB/message. The pipe
has a security descriptor restricting it to **LocalSystem + the interactive user**
(so no other process can inject fake samples). The helper auto-reconnects with ~1 s
backoff if the pipe drops; each sample carries a monotonic seq so the service can
drop a duplicate on reconnect.

---

## 3. Data flow (one interval's life)

1. Helper samples foreground state every **3–5 s** in memory.
2. When the **foreground window (HWND) changes**, the **app changes**, the
   **browser domain changes**, or the **activity state flips**, the helper closes
   the current interval and sends it to the service. (HWND-change matters: two
   Chrome windows on different domains are the same process — without it, time on
   one domain is mis-attributed to the other.) The helper **ignores foreground
   hijackers** (`Consent.exe`/UAC, `LogonUI`, elevated `TaskMgr`, `WerFault`, and
   any window in a different session), keeping the last real app for their duration.
3. Service appends the closed interval to the **SQLite buffer** (`synced = 0`).
   Open-interval **duration is measured from a monotonic clock**; `startUtc`/`endUtc`
   are wall-clock UTC. On **sleep/suspend** the open interval is closed at the
   suspend time; a new `idle` interval starts on resume.
4. Every **5 min** (±30 s jitter), the sync loop gathers unsynced rows, gzips them
   as one JSON batch with a **UUID `batchId`**, and `POST`s to `/api/v1/ingest`.
5. On HTTP 200 + ACK (with a matching `batchId`), the service marks those rows
   `synced = 1` and purges. On failure it keeps them and retries with exponential
   backoff + jitter.
6. The console validates the token (and that it's bound to this `deviceId`), writes
   the `batchId` receipt and all intervals in **one transaction**, and resolves each
   interval's **`category`** (admin rules) and **`employeeId`** (current mapping) at
   ingest before storing. A replayed `batchId` is a no-op (`duplicate: true`).

---

## 4. The three-state activity model

Each interval carries exactly one **state**:

| State | Condition |
|-------|-----------|
| `active` | Input (keyboard/mouse) within the **5-min** idle threshold. |
| `passive_media` | No recent input, **but** real media is *Playing*: GSMTC reports a Playing session, **or** WASAPI audio is active **and** the most-recent GSMTC source is a **non-communications** app. |
| `idle` | No recent input **and** no media playing. |

**Calls are not media.** Raw audio output also fires on Teams/Zoom/Meet/Discord/
Slack calls, notification dings, and screen-reader TTS. The WASAPI fallback is
therefore **gated**: it only yields `passive_media` when the active/most-recent
GSMTC media source is a non-communications app (Decision #7). A user on a silent
call is `idle` until they provide input; a user actively on a call (input within
threshold) is `active`. Misclassifying a 3-hour client call as "passive media"
would be both wrong and damaging.

**Hysteresis** (Decision #34): the state machine enters `idle` at 300 s of no input
but **leaves `idle` only after ~30 s of sustained input**, and sub-~5 s intervals
are merged. Without this, a single input near the threshold oscillates the state
every sample and flushes constantly — defeating Model B's "rare flush" benefit.

Idle **splits** an interval: if the user is active on an app and then goes idle on
the same app, that produces an `active` interval followed by an `idle` interval, so
"active time" is never inflated.

Gaming is handled implicitly: games produce continuous input → `active`, tagged
with the game's process name. AFK-in-game (no input, no media) → `idle`. Honest.

---

## 5. Identity model

- **DeviceId**: an opaque random GUID generated by the agent at first run, stored
  redundantly (ACL-locked file under `C:\ProgramData\...` **and** a registry key).
  It is the MongoDB `_id` for the device and the join key for all intervals.
- **Mutable attributes** (never identity): `pcName` (label), `hostname`,
  `osUsername`, `mac`, `ip`, `agentVersion`. Renaming a PC is just a field update.
- A **reformatted / reimaged** machine loses the stored GUID and re-enrolls as a
  **new device** (v1 has no hardware dedup — an admin re-maps it to the employee).
- Office reality: **one dedicated desktop per employee, one Windows user** per
  machine. Shared-Windows-login attribution is out of scope for v1, but every
  interval carries the Windows **`sessionId`** and **`osUsername`** snapshot, so a
  mis-mapped machine (intervals under two distinct usernames in a short window) is
  *visible* in the dashboard rather than silently merged.

---

## 6. Security model

- **Transport**: HTTPS only, Cloudflare in front.
- **Enrollment** uses the **org key** (shared secret embedded in the MSI, used
  only during `/enroll`, rotatable). It grants nothing except the ability to
  enroll and receive a device token.
- **All other calls** use a **per-device token** (long random secret), stored
  encrypted at rest via **DPAPI machine scope** in an ACL-locked folder. Tokens
  are stored **hashed** on the server. A compromised machine is revoked
  individually without re-keying the fleet.
- **Local buffer** is encrypted and ACL-locked so a standard user can't read or
  tamper with it.
- **Dashboard auth**: the admin dashboard sits behind **Cloudflare Access**
  (email-OTP / SSO) **and** a Next.js `middleware.ts` that rejects unauthenticated
  `/dashboard/*` and admin-API requests — so a request that bypasses Cloudflare
  (direct IP, misconfigured DNS) is still refused. Wired in **Phase 0**; the
  dashboard is never deployed open. (The agent API routes keep their Bearer-token
  auth; this is the *human* access layer that was missing.)
- **Rate limiting**: the server throttles `/ingest`, `/heartbeat`, and `/enroll`
  per device/IP (see API-CONTRACT §5) so a buggy or rogue agent can't flood it.
- **Both org-key and the manifest can be compromised at the edge**, so: the org key
  is **rotatable** (an array of active keys with expiry); a **blocked device is
  refused at `/enroll`**; and the **manifest is signed** (verified before the agent
  trusts any package URL). New enrollments are surfaced in the dashboard so a rogue
  enrollment is visible and revocable.
- **Auto-update packages are code-signed**; the agent verifies the manifest
  signature, then the package signature + SHA-256, before swapping its binary.
  **Self-signed + internally-trusted cert in v1, pre-pushed to `LocalMachine\Root`
  by IT via GPO before rollout** (Decision #18) so SmartScreen/Defender stay quiet.
- **Threat model**: we defend against a **standard (non-admin) employee**. We do
  **not** claim to beat a local administrator. The heartbeat (online/offline **and
  capture health**) is the backstop — an agent that goes silent, or whose helper is
  being killed in a loop, is itself a flag.

---

## 7. Server data model (MongoDB collections)

> Indicative shapes; the implementing repo owns exact schemas/validation. All
> timestamps are stored in **UTC**.

- **devices**: `{ _id: deviceId, pcName, hostname, osUsername, employeeId?,
  agentVersion, os, status, lastSeenAt, lastSampleUtc, helperRunning,
  clockOffsetMs?, enrolledAt, blocked: bool, archived: bool, machineHints:
  { machineGuid }, config }`
  - **`status` is an explicit server-owned state machine** (Decision #39):
    `pending` (enrolled, no intervals yet) → `online`/`offline` (driven by
    `lastSeenAt` vs `heartbeatIntervalSec × 3`), plus `outdated` (< `minSupportedVersion`),
    `clock_skewed`, `revoked`, `archived`. The dashboard reads **only** `status`,
    never re-derives it from `archived`/`revokedAt`/`lastSeenAt` separately.
- **intervals**: `{ _id, deviceId, employeeId, sessionId, startUtc, endUtc,
  durationSec, state, appName, processName, windowTitle, browser?, domain?,
  pageTitle?, urlCaptureMethod, osUsername, category, batchId, receivedAt }`
  - `employeeId` and `osUsername` are **snapshots at ingest/capture** — never
    back-filled; the current values live on `devices`. `receivedAt` is **server-set**.
  - Indexes: `{ deviceId, startUtc }`, **`{ employeeId, startUtc }`** (the hot
    dashboard query), **`{ deviceId, batchId }` unique**, `{ domain }`,
    `{ category }`.
- **batches** (receipt; retained independently of `intervals` — Decision #28):
  `{ _id: batchId, deviceId, accepted, receivedAt }`, unique on `{ deviceId,
  batchId }`. Ingest checks this first; an admin deleting a device's intervals
  **never** deletes its batch receipts, so a stale replay can't resurrect data.
- **categories**: `{ _id, pattern, matchType: 'exact'|'domain'|'process',
  priority, category: 'productive'|'unproductive'|'neutral', updatedBy, updatedAt }`
  - **Precedence** (deterministic): `exact` > `domain` > `process`, then higher
    `priority`; unmatched defaults to `neutral`.
- **employees**: `{ _id, name, email?, department?, deviceIds: [] }`
- **deviceTokens**: `{ _id, deviceId, tokenHash, createdAt, revokedAt? }` — unique
  index on `tokenHash` (every ingest/heartbeat looks up by it).
- **settings** (document-per-key so updates are atomic):
  - `orgKeys: [{ keyId, valueHash, createdAt, expiresAt, revokedAt? }]` (rotation)
  - `agentDefaults: { sampleIntervalSec, idleThresholdSec, … }`
  - `rateLimits: { ingestPerDevice, heartbeatPerDevice, enrollPerIp }`
  - `retention: { intervalDays: null, screenshotDays: 7 }`
  - `orgTimeZone: "Asia/Dhaka"` (used for day/week bucketing — Decision #40)
- **auditLog**: `{ _id, adminId, action, target, before?, after?, atUtc }` — every
  destructive/rules-changing admin action (revoke, archive, delete, category edit,
  device→employee remap). Planned for Phase 4.

Categorization and `employeeId` are resolved at ingest (stored for fast queries)
**and** bulk-re-resolvable when rules/mappings change (no re-ingest needed).

---

## 8. The console app

- One **Next.js** app serves both the **API routes** the agents call and the
  **dashboard UI** admins use. At ~100 agents the load is ~0.3 req/s — trivial.
- **Cloudflare** provides TLS, WAF, DDoS protection, **and Cloudflare Access** (the
  admin auth layer); a `middleware.ts` backstops it (see §6).
- **MongoDB** (single-node **replica set**, so transactions work) is the store; the
  dashboard queries aggregations (time per app/domain/category/state per employee
  per day/week). Every aggregation query is **date-bounded** and uses the
  `{employeeId|deviceId, startUtc}` indexes. If these slow as data grows, add a
  pre-aggregated `dailySummary` collection (deferred — see DECISIONS).
- **Time**: stored **UTC**. **Day/week bucketing** uses the fixed **org timezone**
  (`Asia/Dhaka`); **timeline rendering** uses the **viewer's** local zone, converted
  **client-side** (`Intl…resolvedOptions().timeZone`) to avoid SSR/CSR mismatch.
- **`GET /api/v1/health`** checks MongoDB connectivity; an external uptime monitor
  (e.g. UptimeRobot) watches it — nothing else watches the console itself.
- **DB backups**: a daily `mongodump` to R2 with a documented restore test (the data
  is irreplaceable and retention is indefinite).

### Dashboard views (v1)
- Per-employee **daily timeline** (state-colored strip).
- Per-employee **daily/weekly summary** (active vs passive_media vs idle; top
  apps & domains; productive/unproductive/neutral split).
- **Company overview** (all employees, sortable).
- **Heartbeat panel** (online/offline/last-seen).
- **Device → employee mapping** and **category rule** management.

Deferred: alerts, departments/org chart, PDF/CSV export.

---

## 9. Update distribution

- CI (GitHub Actions, Windows runner) builds + tests + publishes the agent on
  release: `dotnet publish` (self-contained single-file) → WiX MSI → **sign the
  package and the manifest** → upload to **R2** → update **manifest.json**.
- Agent polls the **manifest** (piggybacked on heartbeat), and if a newer version
  exists: **verify manifest signature → download from R2 with HTTP Range/resume →
  verify package signature + SHA-256 → swap → restart**. The embedded public key is
  compiled in, never fetched.
- **Binary swap mechanics** (a running `.exe` can't overwrite itself): download to
  `…\AtlasGreen\update\`, write a `pending-update` marker, then on service
  `OnStop` (or via a tiny bootstrap) rename old→`.bak`, new→in place, and let SCM
  auto-restart launch the new binary. No reboot required; abort safely on any
  failure (keep running the current version, report `updateStatus: failed`).
- A new agent version may add config fields; the agent honors `configSchemaVersion`
  (ignore-unknown for forward-compat) so a v1.1 agent against a v1.0 config doesn't
  break.
- First install is a manual MSI walk to ~100 PCs (IT pre-trusts the signing cert via
  GPO first); every update after is remote.
