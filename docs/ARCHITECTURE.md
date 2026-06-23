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

- The service runs as **LocalSystem** (it needs `SE_TCB` to use a user token). **It
  must stay LocalSystem** — the device token and buffer key are DPAPI machine-scope
  blobs that only LocalSystem can decrypt; switching to a least-privilege account
  would brick the token and lose the buffer (no migration path in v1).
- It calls **`WTSGetActiveConsoleSessionId`** → **`WTSQueryUserToken`** →
  **`CreateProcessAsUser`** (with `lpDesktop = "Winsta0\\Default"`,
  `CREATE_NO_WINDOW | CREATE_UNICODE_ENVIRONMENT`, and the user's environment block)
  so the helper runs **windowless, as the logged-on user** (not SYSTEM — UIPI would
  otherwise block UIA reads). It launches the helper **only when a real user token
  exists** — not at the logon/lock screen (Session-0 / logon-UI token).
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
has a security descriptor restricting it to **LocalSystem + the interactive user**.
Because the user is an allowed pipe client, the service **verifies the peer** on
connect — `GetNamedPipeClientProcessId` → `QueryFullProcessImageName` → confirm it's
the installed helper exe and **code-signed by the agent cert** — and rejects anything
else (a standard user can otherwise connect and inject fake samples). **In dev** (the
agent running as a console, helper unsigned — the signing cert isn't created until
Phase 5/6) the signature check is **skipped behind a dev flag**; only the image-path
check runs. In prod (Windows Service) the **full signature check is mandatory**. On connect the
helper sends a **per-instance UUID** handshake; the service dedupes samples by
`(instanceId, seq)`, so a seq reset after a helper restart can't falsely drop the
first post-restart samples. The helper auto-reconnects with ~1 s backoff.

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
   are wall-clock UTC. On **sleep/suspend** (the service's `SERVICE_CONTROL_POWEREVENT`
   handler, debounced for sub-30 s blips) the open interval is closed at the suspend
   time; on **resume the new interval's state is re-derived from the next sample**
   (it may be `passive_media` if media auto-resumed — not presumed `idle`). A
   monotonic sanity check force-closes any open interval whose elapsed time exceeds
   the idle threshold ×3, in case a power event is missed (VM / Modern Standby). A
   force-close **splits**: the span up to the last real sample keeps its sampled
   state, the gap after it is recorded **`idle`** (+ a `missed_power_event` log), so a
   silent multi-hour hang never becomes a phantom `active` block.
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
therefore **gated**: it only yields `passive_media` when a **recent (≤60 s)** active
GSMTC source is a **non-communications** app (Decision #7). An **active comms call
overrides** — even if Spotify is also playing, a user on a Teams call is never
`passive_media` (they're `active` if inputting, else `idle`). A user on a silent
call is `idle` until they provide input. Misclassifying a 3-hour client call as
"passive media" would be both wrong and damaging.

**Hysteresis** (Decision #34): the machine enters `idle` after **300 s with no
meaningful input** — and only a **meaningful input quantum** (≥~2 s of sustained input
**or** ≥3 input events within a 10 s window; a mouse-move < 4 px is ignored) **resets**
the timer / **leaves `idle`** (→ `active`). A *single isolated event* — one mouse-poll,
one jiggle — does not. This sits between v1.1's "30 s sustained" rule (which buried a
bursty-typing call as `idle`) and v1.2's "any input" rule (which let a $2 mouse-jiggler
or a wireless-mouse keepalive keep an empty desk perpetually `active`); a deliberate
keep-awake tool is additionally surfaced as a **`tamper` flag** (Decision #51), since
the quantum can't always tell patterned fakery from real bursts. Hysteresis governs
only the input-driven `idle ↔ active` boundary; **media transitions are immediate**
(`idle → passive_media` the moment media is detected, no wait). Sub-~5 s intervals are
merged, resolving a "presumed-then-sampled" conflict in favor of the sampled state.

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
  per device/IP (see API-CONTRACT §5) so a buggy or rogue agent can't flood it
  (`/enroll` exempts a corp-CIDR allowlist so the manual rollout isn't blocked).
- **Revoked devices stay visible, not dark**: a revoked token is rejected on
  `/ingest`/`/enroll` but **accepted on `/heartbeat`** (→ `flags:[revoked]`), so the
  machine shows as `revoked` rather than silently "offline"; an admin un-revoke sets a
  **sticky** `reauthorized` (until the agent acts) and the agent self-heals by
  resuming `/ingest` on its existing token. A revoked heartbeat updates `lastSeenAt`
  only — **capture-health fields freeze** at the revoke instant and the dashboard shows
  `revoked` as the primary badge — so a leaked revoked token can't fake a healthy,
  online device (keeping the silent-agent backstop honest).
- **Three distinct admin actions** (Decisions #45/#49/#48): **revoke** = security kill
  (token dies; `/ingest`+`/enroll` `401`/`403`); **archive** = retire (token stays
  valid but ingest-blocked, agent pauses capture, reversible — for a decommissioned PC
  or departed employee whose history is kept); **block & wipe** = irreversible erase
  (delete intervals + receipts + dataGaps, doc retained blocked so it can't re-enroll).
- **Tamper signal**: the agent reports known keep-awake/jiggler processes; the server
  raises a `tamper` flag (Decision #51). Defends against standard users only (below).
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

- **devices**: `{ _id: deviceId, pcName, hostname, osUsername, timezone, employeeId?,
  agentVersion, os, configSchemaVersion, liveness, flags, lastSeenAt, lastSampleUtc,
  captureEnabled, helperRunning, helperStartedAtUtc, machineGuidHistory: [],
  clockOffsetMs?, lastDropped?, enrolledAt, blocked: bool, archived: bool,
  machineHints: { machineGuid }, config }`
  - **State = liveness + flags** (Decision #39, server-owned, **derived from the
    stored inputs**). The **source of truth** is the booleans/fields `blocked`,
    `archived`, `lastSeenAt`, `deviceTokens.revokedAt`, `agentVersion`,
    `configSchemaVersion`, `clockOffsetMs`, `machineGuidHistory`, health samples. The
    server computes `liveness` + `flags` from them and stores the result denormalized
    — written **only** through one `updateDeviceState(deviceId, …)` helper, in a single
    atomic document update, so the two can never disagree (a revoke sets `blocked` +
    pushes `revoked` + sets `revokedAt` together). The dashboard reads `liveness`/`flags`
    and **never re-derives**.
  - `liveness`: `pending` (enrolled, not yet seen) → `online`/`offline`
    (`lastSeenAt` vs `heartbeatIntervalSec × 3`).
  - **Flag triggers & clear/co-existence** (orthogonal — any combination; `revoked`
    then `archived` take dashboard-badge precedence over `liveness`):

    | flag | set when | cleared when | terminal? |
    |------|----------|--------------|-----------|
    | `outdated` | `agentVersion < minSupportedVersion` | agent updates (next heartbeat) | no |
    | `clock_skewed` | `|clockOffsetMs| > 1 h` for 2 heartbeats | `< 30 min` for 2 heartbeats | no |
    | `health_warning` | `cpuPct > 5` for 3 consecutive heartbeats **or** `memMb > 200` once | condition normal for 2 heartbeats | no |
    | `config_mismatch` | agent `configSchemaVersion` < server's | agent catches up | no |
    | `tamper` | heartbeat reports `tamperHints` (keep-awake/jiggler) | no hint for 2 heartbeats | no |
    | `possible_clone` | ≥2 distinct **non-null** `machineGuid`s in `machineGuidHistory` | admin "dismiss clone" (audit-logged) | sticky until dismissed |
    | `revoked` | admin revoke (`revokedAt`) | admin un-revoke (→ sticky `reauthorized`) | until un-revoked |
    | `archived` | admin archive (`archived:true`) | admin un-archive (→ `reauthorized`) | until un-archived |

    `liveness` is independent of every flag. `revoked` + `archived` may co-exist
    (`revoked` wins the badge). Non-terminal badges (`outdated`/`clock_skewed`/
    `health_warning`/`config_mismatch`/`tamper`/`possible_clone`) may stack freely.
- **intervals**: `{ _id, deviceId, employeeId, sessionId, startUtc, endUtc,
  durationSec, state, appName, processName, windowTitle, browser?, domain?,
  pageTitle?, urlCaptureMethod, osUsername, category, batchId, receivedAt }`
  - `osUsername` is a **capture-time snapshot, never back-filled**. `employeeId` is
    resolved at ingest but **re-resolvable** (admin remap → future intervals + an
    optional audit-logged bulk reassign of a historical range). `durationSec` is
    canonical (not `endUtc − startUtc`). `receivedAt` is **server-set**, equal to the
    batch's `receivedAt` (one commit-time stamp).
  - Indexes: `{ deviceId, startUtc }`, **`{ employeeId, startUtc }`** (hot query),
    **`{ deviceId, batchId }` NON-unique** (find/delete-by-batch — uniqueness lives on
    `batches`, never here, or every multi-interval batch would be rejected),
    `{ domain }`, `{ category }`.
- **batches** (receipt; Decision #28): `{ _id: batchId, deviceId, accepted,
  receivedAt, expireAt }`. **`_id = batchId`** is the dedup key (a UUIDv4 is globally
  unique — no compound unique index needed); a non-unique `{ deviceId, receivedAt }`
  index serves purge queries. **TTL on `expireAt`**, where `expireAt = receivedAt +
  batchReceiptDays` and `batchReceiptDays = max(180, maxBufferCapDays + 45)` — the
  receipt window **must outlive the fleet's max buffer hold**, or a long-offline /
  crash-corrupted agent could replay a batch whose receipt expired and duplicate data
  (the `intervals` index is non-unique and wouldn't stop it). Ingest checks this first;
  ordinary interval delete **never** touches receipts (no stale-replay resurrection),
  but the **block-&-wipe** admin action deletes intervals, receipts **and** dataGaps.
- **categories**: `{ _id, pattern, matchType: 'exact'|'domain'|'process',
  priority, category: 'productive'|'unproductive'|'neutral', updatedBy, updatedAt }`
  - **Precedence** (deterministic): `exact` > `domain` > `process`, then higher
    `priority`; unmatched defaults to `neutral`.
- **employees**: `{ _id, name, email?, department?, tzOverride?, deviceIds: [] }`
  (`tzOverride` = optional IANA tz for a non-Dhaka hire's day-bucketing).
- **deviceTokens**: `{ _id, deviceId, tokenHash, createdAt, revokedAt?,
  supersededAt? }` — unique index on `tokenHash`. Token-loss re-enroll supersedes the
  old row and mints a new one (Decision #43).
- **dataGaps**: `{ _id, deviceId, count, byState, oldestUtc, newestUtc, detectedAt }`
  — persisted from a heartbeat `dropped` block so a buffer-overflow gap stays visible
  on the timeline after the heartbeat passes.
- **settings** (document-per-key so updates are atomic):
  - `orgKeys: [{ keyId, valueHash, createdAt, expiresAt, revokedAt? }]` — expired +
    revoked keys pruned ~90 d after revocation.
  - `agentDefaults: { sampleIntervalSec, idleThresholdSec, … }`
  - `rateLimits: { ingestPerDevice, heartbeatPerDevice, enrollPerIp, enrollAllowlistCidrs: [] }`
  - `retention: { intervalDays: null, batchReceiptDays, screenshotDays: 7 }` —
    `batchReceiptDays = max(180, maxBufferCapDays + 45)` (must exceed the fleet's max
    `bufferCapDays`; see batches TTL).
  - `contractDeprecation: { [contractMajor]: "warn" | "reject" }` — drives the
    heartbeat `contractDeprecated` flag; never set for the current major.
  - `orgTimeZone: "Asia/Dhaka"` — **validated against the IANA db** before save;
    changes are audit-logged + stamped `orgTimeZoneChangedAt` and affect future
    bucketing only.
- **auditLog**: `{ _id, adminId, action, target, before?, after?, atUtc }` — every
  destructive/rules-changing admin action (revoke/un-revoke, archive/un-archive,
  **block & wipe**, dismiss-clone, delete, category edit, device→employee remap,
  `orgTimeZone` change). Written **before** any destructive step (so a wipe is
  auditable after the data is gone). Infra + write helper land in **Phase 0** (the
  mapping action ships then); the viewer is Phase 4. `adminId` = the Cloudflare-Access
  email until Phase 4 NextAuth assigns a stable UUID.

Categorization and `employeeId` are resolved at ingest (stored for fast queries)
**and** bulk-re-resolvable when rules/mappings change (no re-ingest needed).

---

## 8. The console app

- One **Next.js** app serves both the **API routes** the agents call and the
  **dashboard UI** admins use. At ~100 agents the load is ~0.3 req/s — trivial.
- **Cloudflare** provides TLS, WAF, DDoS protection, **and Cloudflare Access** (the
  admin auth layer); a `middleware.ts` backstops it (see §6).
- **MongoDB** is a **single-node replica set** (so transactions work) bootstrapped
  with `rs.initiate()` bound to **`127.0.0.1`** (binding to a hostname the driver
  can't resolve from `localhost` is the classic "not primary" failure); connection
  string `…?replicaSet=rs0`. A deterministic init script + dev `docker-compose` ships
  in Phase 0. Aggregations are **date-bounded** on `{employeeId|deviceId, startUtc}`;
  add a pre-aggregated `dailySummary` later if they slow (deferred).
- **Time**: stored **UTC**. **Day/week bucketing** uses `settings.orgTimeZone`
  (`Asia/Dhaka`, or `employees.tzOverride` for a non-Dhaka hire), with the device's
  `clockOffsetMs` applied **before** the day boundary is computed. **Timeline
  rendering** uses the **viewer's** local zone, converted **client-side**
  (`Intl…resolvedOptions().timeZone`). Summary bucket ids are returned as org-tz date
  strings (`bucketDate` + `bucketTimeZone`) and are **not** client-converted, so the
  summary and the timeline don't disagree.
- **`GET /api/v1/health`** checks MongoDB connectivity (and replica-set status); an
  external uptime monitor (e.g. UptimeRobot) watches it.
- **DB backups**: a daily `mongodump` to R2 with a documented, **rehearsed** restore
  procedure (restore data → `rs.initiate()` on the fresh node → transactions resume);
  a VPS loss costs ≤ 24 h while agents buffer locally.

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
  package and the manifest** → upload to **R2** (which **retains the last 5 versions**
  so a downgrade target is never 404) → update **manifest.json**.
- Agent polls the **manifest** (piggybacked on heartbeat), and if a newer version
  exists: **verify manifest signature → download from R2 with HTTP Range/resume →
  verify package signature + SHA-256 → swap → restart**. The embedded public key is
  compiled in, never fetched.
- **Binary swap mechanics** (a running `.exe` can't overwrite itself): download to
  `…\AtlasGreen\update\`, write a `pending-update` marker, then on service
  `OnStop` (or via a tiny bootstrap) rename old→`.bak`, new→in place, and let SCM
  auto-restart launch the new binary. **On every startup** the service checks the
  marker and finishes a stranded swap (if the staged package still verifies) before
  the main loop — so a crash mid-update self-heals. No reboot required; abort safely
  on any failure (keep the current version, report `updateStatus: failed`).
- The **MSI upgrade/repair preserves `C:\ProgramData\AtlasGreen`** (DeviceId, DPAPI
  blobs, buffer) — only a full uninstall removes it — so an update never forks
  identity or loses the buffer.
- A new agent version may add config fields; the agent honors `configSchemaVersion`
  (ignore-unknown for forward-compat) so a newer-config agent doesn't break.
- First install is a manual MSI walk to ~100 PCs (IT pre-trusts the signing cert via
  GPO on AD machines, or PowerShell/MSI custom action on workgroup PCs); every update
  after is remote.
- **One signing cert signs both the service and the helper** for the whole v1 fleet
  (the service's pipe peer-check verifies the helper against that cert — §2.2). If the
  cert is ever rotated (leak/expiry/OV upgrade), the rotation must ship **as an agent
  auto-update that bundles the new public key and accepts both the old and new
  signatures during a transition window**, or a staggered rollout would have new
  services rejecting old helpers (and vice-versa).
