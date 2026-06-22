# Atlas Green — Build Plan

Work proceeds **phase by phase**. Each phase has a checklist and an explicit
**exit gate**. A phase is done only when every box is ticked, tests are green, and
the human has manually verified the goal. After each phase the human returns to
Claude for a **codebase review** before the next phase starts.

**Coding ownership:** Phase 0 is implemented by Claude (first coder). Phases 1+ are
implemented by Opencode, with Claude + the human as reviewers. READMEs are kept
current as features land.

Legend: `[ ]` todo · `[x]` done. Repos: **A** = atlas-agent, **C** = atlas-console,
**M** = atlas-green (docs).

---

## Phase 0 — Foundations + thin vertical slice  ← *Claude implements*

**Goal:** both apps build and run in dev, and a dummy heartbeat travels
agent → console → MongoDB → dashboard shows the device "online." Prove the whole
pipe end-to-end before building real features.

- [ ] **A** .NET 8 solution scaffolded; runs as **console in dev** and **Windows
      Service in prod** via the generic host (`UseWindowsService`).
- [ ] **A** Structured logging to a rotating local file (dev: also console).
- [ ] **A** Reads config (sample/idle/sync/heartbeat intervals) with defaults.
- [ ] **A** Generates + persists a **DeviceId GUID** (file + registry, ACL-locked).
- [ ] **A** Calls `/enroll` (org key) → stores device token via **DPAPI**.
- [ ] **A** Sends a **dummy heartbeat** on a timer.
- [ ] **A** **Self-contained single-file publish smoke test**: `dotnet publish -r
      win-x64` and run the `.exe` on a clean Windows VM (no .NET SDK) — it enrolls +
      heartbeats. Validates the publish pipeline before any real logic. *(was Ph5)*
- [ ] **A** Minimal **SQLite** skeleton (create DB, one table, insert/read a row) to
      validate the library + migration approach early.
- [ ] **C** Next.js app scaffolded; connects to **MongoDB** initialized as a
      **single-node replica set** (`rs.initiate()`, needed for transactions later).
- [ ] **C** `/api/v1/enroll` and `/api/v1/heartbeat` implemented per the contract.
- [ ] **C** **`settings` collection schema** seeded (orgKeys array, agentDefaults,
      rateLimits, retention, orgTimeZone).
- [ ] **C** **Admin auth from day one**: Cloudflare Access in front + a
      `middleware.ts` that rejects unauthenticated `/dashboard/*`. *(was unplanned)*
- [ ] **C** Minimal dashboard page lists devices with **online/offline + last-seen**.
- [ ] **C** Device → employee mapping stub (assign a name to a device).
- [ ] **C** **Dev seed script** (3 sample devices, one with 24 h of synthetic
      intervals across all 3 states) so the dashboard is built against real-shaped
      data, not an empty DB.
- [ ] **A+C** Unit-test scaffolding wired (`dotnet test` / `vitest`) and green.
- [ ] **M** READMEs updated with "how to run in dev" for both repos.

**Exit gate:** `dotnet run` (agent) enrolls and heartbeats against `npm run dev`
(console); the device appears **online** in the dashboard and flips **offline**
when the agent stops; **an unauthenticated browser cannot view `/dashboard`**; the
published single-file `.exe` runs on a clean VM.

---

## Phase 1 — Core capture (local only)  ← *Opencode*

**Goal:** the agent accurately produces aggregated activity **intervals** locally
(logged/inspected), with the correct three-state model. No server changes.

- [ ] **A** User-session **helper** launched from the service via **`WTSQueryUser
      Token` + `CreateProcessAsUser`** (service as **LocalSystem**, helper as the
      user); reacts to **`WTS_SESSION_CHANGE`** (logon/logoff, fast user switch);
      **console session only**, RDP ignored. Watchdog restarts a dead helper. *(core
      correctness — was deferred to Ph7)*
- [ ] **A** Helper runs **windowless** in prod config (no console window); verified
      on a non-admin account. *(continuous silent invariant from here on)*
- [ ] **A** Foreground **window + process** sampling every 3–5 s; **ignore foreground
      hijackers** (`Consent.exe`/UAC, `LogonUI`, elevated `TaskMgr`, `WerFault`).
- [ ] **A** **`appName` from the exe `FileDescription`** (fallback = exe name); each
      interval tagged with the Windows **`sessionId`**.
- [ ] **A** **Idle** detection via `GetLastInputInfo` (5-min threshold) **with
      hysteresis** (leave idle only after ~30 s sustained input).
- [ ] **A** **Media** detection via **GSMTC** (Playing?) → `passive_media`; **gated
      WASAPI fallback** (audio counts only when the active GSMTC source is a
      **non-communications** app — calls are never `passive_media`).
- [ ] **A** **Three-state machine** (`active` / `passive_media` / `idle`).
- [ ] **A** **Interval aggregator** (Model B): flush on **HWND**/app/domain/state
      change; idle **splits** the interval; **durations from a monotonic clock**;
      **close the open interval on sleep/suspend** (`PowerModeChanged`), reopen idle
      on resume. *(sleep/clock were Ph7)*
- [ ] **A** **Graceful shutdown**: on `ApplicationStopping`, flush the open interval
      to the buffer (and stop the helper) so restarts don't lose activity.
- [ ] **A** Helper → service transport: **length-prefixed JSON named pipe** with a
      LocalSystem+user security descriptor (no other process can inject samples).
- [ ] **A** Heartbeat carries **capture health** (`helperRunning`, `lastSampleUtc`,
      `helperRestartCount`). *(so "online" ≠ "service alive but helper dead")*
- [ ] **A** **Unit tests** for the state machine + aggregator (pure logic), incl.
      the **oscillating-input** (hysteresis) and **sleep-resume** scenarios.
- [ ] **M** README: what's captured and how to watch it live in dev.

**Exit gate:** running in dev and switching apps / playing a video / **being on a
Teams call** / going idle / sleeping+resuming produces correct intervals with
correct states (the call is `active`/`idle`, **not** `passive_media`); the helper
relaunches correctly on a fast user switch; verified live in the logs.

---

## Phase 2 — Buffer + sync  ← *Opencode*

**Goal:** intervals persist to SQLite and reliably reach MongoDB via `/ingest`.

- [ ] **A** **SQLite** buffer with **app-layer AES-GCM** encryption (key via **DPAPI
      machine scope**; no native dep), ACL-locked under `C:\ProgramData\...`.
- [ ] **A** Append closed intervals; `synced` flag; buffer cap. When the cap forces a
      drop, **never silently**: log it and report `dropped {count,oldest,newest}` on
      the next heartbeat (prefer dropping `idle` over active/passive). *(was Ph7)*
- [ ] **A** **Sync loop** (5 min **±30 s jitter**): gzip batch with a **UUID
      `batchId`** → `/ingest`; **ACK before purge** (verify response `batchId`).
- [ ] **A** Exponential backoff **+ jitter**; offline accumulation; honor
      `Retry-After`; **persist nothing about `batchId` across restarts is needed**
      (UUID removes reuse risk).
- [ ] **A** **Config validation**: clamp/ignore out-of-range server config values.
- [ ] **C** `/api/v1/ingest` per contract: validate token **(bound to deviceId)**,
      **atomic per batch in one transaction**, write the **`batches` receipt**,
      dedupe on `{deviceId,batchId}`, resolve **category + employeeId** at ingest.
- [ ] **C** `intervals` + `batches` collections + indexes (`{deviceId,batchId}`
      unique, `{employeeId,startUtc}`); `deviceTokens.tokenHash` unique index.
- [ ] **C** **Device `status` state machine** (server side: pending/online/offline/
      outdated/revoked) driven by heartbeat + admin actions.
- [ ] **C** **Rate limiting** on `/ingest`,`/heartbeat`,`/enroll` (per device/IP).
- [ ] **C** **Org-key array + rotation** in `settings`; **blocked device → `403`** at
      `/enroll`; re-enroll returns the existing token (no rotation). *(was Ph7)*
- [ ] **A** Unit tests for batching / retry / purge-after-ack.
- [ ] **C** Integration tests: batch in → stored once; **partial-insert-then-retry**
      never loses or duplicates; stale replay of an admin-deleted batch is rejected.

**Exit gate:** intervals captured offline flush to MongoDB when connectivity
returns; retried/partial/stale batches never double-store or resurrect data; a
buffer-cap drop is visible on the heartbeat panel.

---

## Phase 3 — URL capture  ← *Opencode*

**Goal:** browser intervals carry **domain + page title** across all browsers /
profiles / incognito.

- [ ] **A** **UIA feasibility spike first**: read the address bar from Chrome, Edge,
      Brave, Opera, Firefox (normal + incognito) and document the working
      `AutomationId`/`ControlType` path per browser. Firefox is the known-weak one.
- [ ] **A** **Fallback chain**: UIA → window-title parse → `domain: null`, recorded
      via the **`urlCaptureMethod`** field. Never silently drop a browser interval.
- [ ] **A** **Normalize domains to eTLD+1** (strip `www.`/`m.`, public-suffix list)
      *before* buffering, so `youtu.be`/`m.youtube.com` collapse to `youtube.com`.
- [ ] **A** Capture **page title** (≤256 chars); window title ≤512. Filter noise
      (mid-typing address bar, `new tab`, blank).
- [ ] **A** Domain change triggers a new interval (aggregator integration).
- [ ] **A** **DPI-aware** helper manifest; verify UIA name/value reads on a **4K @
      200%** display. *(was Ph7)*
- [ ] **A** Unit tests for **eTLD+1 normalization** (20+ real-world variants) +
      noise filtering (pure logic).
- [ ] **M** README: browser coverage **matrix** (each browser × normal/incognito/2nd
      profile, + UIA-disabled fallback) + known limits (e.g. F11 fullscreen).

**Exit gate:** browsing across normal, incognito, and a second profile on the
supported browsers all yield correct normalized `domain` + `pageTitle` intervals
(or an honest `urlCaptureMethod: title-fallback`/`unavailable`), verified live.

---

## Phase 4 — Dashboard  ← *Opencode*

**Goal:** the console turns stored intervals into the management views.

- [ ] **C** Per-employee **daily timeline** (state-colored).
- [ ] **C** Per-employee **daily/weekly summary** (state split, top apps/domains,
      productive/unproductive/neutral split).
- [ ] **C** **Company overview** (sortable).
- [ ] **C** **Heartbeat panel** (online/offline/last-seen).
- [ ] **C** **Device → employee** management UI.
- [ ] **C** **Category rules** UI (`exact/domain/process → category` with
      precedence + priority), unknown = neutral; bulk re-resolve when rules change.
- [ ] **C** **Day/week bucketing uses the org timezone** (`Asia/Dhaka`); timeline
      **rendering** in the **viewer's** local zone, converted client-side (no
      SSR/CSR flicker).
- [ ] **C** **Audit log** of destructive/rules-changing admin actions; **device→
      employee remap** writes an `auditLog` entry and re-resolves affected intervals.
- [ ] **C** Upgrade admin auth to **NextAuth** (per-admin accounts/roles) on top of
      Cloudflare Access; dashboard reads device **`status`** (not derived fields).
- [ ] **C** Component/integration tests for aggregation queries.

**Exit gate:** a real day of captured data renders correctly across all views;
day-buckets honor the org timezone, timeline times show in the viewer's zone; admin
actions are audit-logged.

---

## Phase 5 — Packaging & hardening  ← *Opencode*

**Goal:** a real, silent, installable agent.

- [ ] **A** `dotnet publish` **self-contained single-file** (publish pipeline already
      smoke-tested in Phase 0).
- [ ] **A** **WiX MSI**; silent install with `PCNAME` param; installs the service.
- [ ] **A** Confirm **zero UI footprint** (no window/tray/console/notification).
- [ ] **A** Service **ACL** hardening (standard user can't stop/uninstall/delete).
- [ ] **A** SCM **auto-restart** recovery; helper **watchdog**.
- [ ] **A** DPAPI token + buffer locked down; verify on a non-admin account.
- [ ] **A** **AV / SmartScreen / Defender review** of the signed MSI+exe; document
      expected behavior per installer variant. *(moved earlier from Ph7)*
- [ ] **A** Verify on an **AppLocker/WDAC-hardened** reference machine: binary
      allow-listed, helper launches, ProgramData R/W + DPAPI work under the
      restricted token; document the GPO allow-list snippet.
- [ ] **M** README/runbook: **GPO cert pre-trust** to `LocalMachine\Root`, install +
      uninstall + PC-name steps for IT.

**Exit gate:** clean install on a test machine (incl. one hardened machine) runs
invisibly, captures + syncs, survives reboot/logout, and a standard user cannot
stop it; no SmartScreen/AV prompt with the cert pre-trusted.

---

## Phase 6 — Auto-update + CI/CD  ← *Opencode*

**Goal:** push code once → fleet updates itself.

- [ ] **A** Self-signed code-signing cert + **GPO trust** process documented.
- [ ] **A** Update client: heartbeat says "newer exists" → **verify manifest
      signature** → download from **R2 with HTTP Range/resume** → verify package
      **signature + SHA-256** → **swap (staging dir + rename on OnStop, no reboot)** →
      restart; abort safely on failure (report `updateStatus: failed`).
- [ ] **A** Honor **`configSchemaVersion`** (ignore-unknown) so a new agent against an
      old config doesn't break; support **downgrade** (`latestVersion < current`).
- [ ] **C/M** Publish a **signed `manifest.json`** + package to R2 (release pipeline).
- [ ] **A** GitHub Actions (Windows): test → publish → MSI → **sign package +
      manifest** → upload R2 → update manifest.
- [ ] **C** Console deploy pipeline (VPS/dokploy) confirmed; **daily mongodump → R2**
      backup with a restore test.
- [ ] **A** End-to-end update test: vX → vY on a running agent (and a forced
      downgrade vY → vX).

**Exit gate:** a running agent auto-updates to a newly released version without
manual touch; **a tampered manifest or bad signature is rejected**; a resumed
download completes over a flaky link.

---

## Phase 7 — Residual hardening & scale  ← *Opencode*

**Goal:** production resilience for what genuinely couldn't be folded into earlier
phases. (Most former Phase-7 items were redistributed to the phase that owns them —
helper/session/sleep/clock → Ph1, buffer-cap/revocation/org-key → Ph2, browser
variants/DPI → Ph3, AV/AppLocker → Ph5 — to avoid a tar-pit final phase.)

- [ ] Multi-monitor; PWAs / Electron apps (appName edge cases).
- [ ] **100-device soak test**: simulate 100 agents for 24 h with realistic
      intervals — zero dropped batches, dashboard queries < 2 s, stable VPS
      CPU/memory.
- [ ] End-to-end **token revocation → re-enroll-blocked (`403`) → revoked state** on a
      real agent.
- [ ] Sweep any edge cases found during earlier phases (expand as discovered).

**Exit gate:** soak test passes at 100 devices; revocation flow verified end-to-end;
no data loss or noticeable footprint.

---

## v2.0.0 — Screenshots  *(future)*

Randomized **10–15 min** capture, skip-on-idle, ~1280px **WebP** @ ~55%, all
monitors stitched, uploaded to **Cloudflare R2** via presigned URLs, **7-day**
retention. Dashboard gallery + timeline thumbnails. (Mind fullscreen-exclusive /
DRM black-frame capture.)
