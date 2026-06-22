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
        │   ┌───────────────── API routes ─────────────────┐     ┌──────────── Dashboard UI ───┐ │
        │   │  POST /api/v1/enroll                          │     │ timeline / summaries /       │ │
        │   │  POST /api/v1/ingest                          │     │ company overview /           │ │
        │   │  POST /api/v1/heartbeat                       │     │ heartbeat panel /            │ │
        │   │  GET  /api/v1/update/manifest                 │     │ device↔employee mapping /    │ │
        │   └───────────────────┬───────────────────────────┘     │ category rules               │ │
        │                       │                                  └──────────────────────────────┘ │
        │                       ▼                                                                    │
        │                 ┌───────────┐                                                              │
        │                 │ MongoDB   │   devices · intervals · categories · employees · settings    │
        │                 └───────────┘                                                              │
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

---

## 3. Data flow (one interval's life)

1. Helper samples foreground state every **3–5 s** in memory.
2. When the **app changes**, the **browser domain changes**, or the **activity
   state flips** (active ↔ passive_media ↔ idle), the helper closes the current
   interval and sends it to the service.
3. Service appends the closed interval to the **SQLite buffer** (`synced = 0`).
4. Every **5 min**, the sync loop gathers unsynced rows, gzips them as one JSON
   batch, and `POST`s to `/api/v1/ingest` with the device token.
5. On HTTP 200 + ACK, the service marks those rows `synced = 1` (then purges).
   On failure, it keeps them and retries with exponential backoff.
6. Console validates the token, dedupes by `batchId`, resolves a **category** for
   each interval (productive/unproductive/neutral) using the admin rules, and
   stores them in MongoDB.

---

## 4. The three-state activity model

Each interval carries exactly one **state**:

| State | Condition |
|-------|-----------|
| `active` | Input (keyboard/mouse) within the **5-min** idle threshold. |
| `passive_media` | No recent input, **but** media is actively *Playing* (GSMTC reports a Playing session; WASAPI audio-output is the fallback signal). |
| `idle` | No recent input **and** no media playing. |

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
  machine. Shared-Windows-login attribution is out of scope for v1.

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
- **Auto-update packages are code-signed**; the agent verifies the signature
  (and a SHA-256 hash from the manifest) before swapping its binary. Self-signed +
  internally-trusted cert in v1.
- **Threat model**: we defend against a **standard (non-admin) employee**. We do
  **not** claim to beat a local administrator. The heartbeat (online/offline) is
  the backstop — an agent that goes silent is itself a flag.

---

## 7. Server data model (MongoDB collections)

> Indicative shapes; the implementing repo owns exact schemas/validation. All
> timestamps are stored in **UTC**.

- **devices**: `{ _id: deviceId, pcName, hostname, osUsername, employeeId?,
  agentVersion, status, lastSeenAt, enrolledAt, archived: bool, machineHints,
  config }`
- **intervals**: `{ _id, deviceId, employeeId?, startUtc, endUtc, durationSec,
  state, appName, processName, windowTitle, browser?, domain?, pageTitle?,
  osUsername, category, batchId, receivedAt }`
  - Indexes: `{ deviceId, startUtc }`, `{ employeeId, startUtc }`, `{ batchId }`
    (unique-ish for idempotent ingest), `{ domain }`, `{ category }`.
- **categories**: `{ _id, pattern, matchType: 'domain'|'process'|'exact',
  category: 'productive'|'unproductive'|'neutral', updatedBy, updatedAt }`
- **employees**: `{ _id, name, email?, department?, deviceIds: [] }`
- **deviceTokens**: `{ _id, deviceId, tokenHash, createdAt, revokedAt? }`
- **settings**: org key(s), default agent config, retention policy, etc.

Categorization is resolved at ingest (store the resolved category for fast
queries) **and** re-resolvable in bulk when rules change.

---

## 8. The console app

- One **Next.js** app serves both the **API routes** the agents call and the
  **dashboard UI** admins use. At ~100 agents the load is ~0.3 req/s — trivial.
- **Cloudflare** provides TLS, WAF, and DDoS protection.
- **MongoDB** is the store; the dashboard queries aggregations (time per
  app/domain/category/state per employee per day/week).
- Stores **UTC**, renders in the **viewer's local time** (multi-timezone team).

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
  release: `dotnet publish` (self-contained single-file) → WiX MSI → **sign** →
  upload package to **R2** → update **manifest.json**.
- Agent polls the **manifest** (piggybacked on heartbeat), and if a newer version
  exists, downloads from R2, verifies signature + hash, swaps, and restarts.
- First install is a manual MSI walk to ~100 PCs; every update after is remote.
