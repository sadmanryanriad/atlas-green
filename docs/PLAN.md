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
- [ ] **C** Next.js app scaffolded; connects to **MongoDB**.
- [ ] **C** `/api/v1/enroll` and `/api/v1/heartbeat` implemented per the contract.
- [ ] **C** Minimal dashboard page lists devices with **online/offline + last-seen**.
- [ ] **C** Device → employee mapping stub (assign a name to a device).
- [ ] **A+C** Unit-test scaffolding wired (`dotnet test` / `vitest`) and green.
- [ ] **M** READMEs updated with "how to run in dev" for both repos.

**Exit gate:** `dotnet run` (agent) enrolls and heartbeats against `npm run dev`
(console); the device appears **online** in the dashboard and flips **offline**
when the agent stops.

---

## Phase 1 — Core capture (local only)  ← *Opencode*

**Goal:** the agent accurately produces aggregated activity **intervals** locally
(logged/inspected), with the correct three-state model. No server changes.

- [ ] **A** User-session **helper** process; service launches + supervises it
      (restart on death / on user login).
- [ ] **A** Foreground **window + process** sampling every 3–5 s.
- [ ] **A** **Idle** detection via `GetLastInputInfo` (5-min threshold).
- [ ] **A** **Media** detection via **GSMTC** (Playing?) → `passive_media`;
      WASAPI audio-output fallback.
- [ ] **A** **Three-state machine** (`active` / `passive_media` / `idle`).
- [ ] **A** **Interval aggregator** (Model B): flush on app/domain/state change;
      idle **splits** the interval.
- [ ] **A** Helper → service transport (e.g. named pipe).
- [ ] **A** **Unit tests** for the state machine + aggregator (pure logic).
- [ ] **M** README: what's captured and how to watch it live in dev.

**Exit gate:** running in dev and switching apps / playing a video / going idle
produces correct intervals with correct states, verified live in the logs.

---

## Phase 2 — Buffer + sync  ← *Opencode*

**Goal:** intervals persist to SQLite and reliably reach MongoDB via `/ingest`.

- [ ] **A** **SQLite** buffer (encrypted, ACL-locked under `C:\ProgramData\...`).
- [ ] **A** Append closed intervals; `synced` flag; buffer cap (~2–3 months).
- [ ] **A** **Sync loop** (5 min): gzipped batch → `/ingest`; **ACK before purge**.
- [ ] **A** Exponential backoff; offline accumulation; `batchId` idempotency.
- [ ] **C** `/api/v1/ingest` per contract: validate token, dedupe `batchId`,
      store intervals, resolve category at ingest.
- [ ] **C** `intervals` collection + indexes.
- [ ] **A** Unit tests for batching / retry / purge-after-ack.
- [ ] **C** Integration test: batch in → stored once (and deduped on retry).

**Exit gate:** intervals captured offline flush to MongoDB when connectivity
returns; retried batches never double-store.

---

## Phase 3 — URL capture  ← *Opencode*

**Goal:** browser intervals carry **domain + page title** across all browsers /
profiles / incognito.

- [ ] **A** **UI Automation** read of the browser address bar (Chrome, Edge,
      Brave, Opera, Firefox).
- [ ] **A** Derive **domain**; capture **page title** from window title/UIA.
- [ ] **A** Filter noise (mid-typing address bar, `new tab`, blank).
- [ ] **A** Domain change triggers a new interval (aggregator integration).
- [ ] **A** Unit tests for domain parsing + noise filtering (pure logic).
- [ ] **M** README: browser coverage + known limits.

**Exit gate:** browsing across normal, incognito, and a second profile all yield
correct `domain` + `pageTitle` intervals, verified live.

---

## Phase 4 — Dashboard  ← *Opencode*

**Goal:** the console turns stored intervals into the management views.

- [ ] **C** Per-employee **daily timeline** (state-colored).
- [ ] **C** Per-employee **daily/weekly summary** (state split, top apps/domains,
      productive/unproductive/neutral split).
- [ ] **C** **Company overview** (sortable).
- [ ] **C** **Heartbeat panel** (online/offline/last-seen).
- [ ] **C** **Device → employee** management UI.
- [ ] **C** **Category rules** UI (`domain/process → category`), unknown = neutral;
      bulk re-resolve when rules change.
- [ ] **C** All times rendered in the **viewer's local zone** (stored UTC).
- [ ] **C** Component/integration tests for aggregation queries.

**Exit gate:** a real day of captured data renders correctly across all views,
times shown in local zone.

---

## Phase 5 — Packaging & hardening  ← *Opencode*

**Goal:** a real, silent, installable agent.

- [ ] **A** `dotnet publish` **self-contained single-file**.
- [ ] **A** **WiX MSI**; silent install with `PCNAME` param; installs the service.
- [ ] **A** Confirm **zero UI footprint** (no window/tray/console/notification).
- [ ] **A** Service **ACL** hardening (standard user can't stop/uninstall/delete).
- [ ] **A** SCM **auto-restart** recovery; helper **watchdog**.
- [ ] **A** DPAPI token + buffer locked down; verify on a non-admin account.
- [ ] **M** README/runbook: install + uninstall + PC-name steps for IT.

**Exit gate:** clean install on a test machine runs invisibly, captures + syncs,
survives reboot/logout, and a standard user cannot stop it.

---

## Phase 6 — Auto-update + CI/CD  ← *Opencode*

**Goal:** push code once → fleet updates itself.

- [ ] **A** Self-signed code-signing cert + internal trust process documented.
- [ ] **A** Update client: poll manifest → download from **R2** → verify
      **signature + SHA-256** → swap → restart; abort safely on failure.
- [ ] **C/M** Publish **manifest.json** + package to R2 (release pipeline).
- [ ] **A** GitHub Actions (Windows): test → publish → MSI → sign → upload R2 →
      update manifest.
- [ ] **C** Console deploy pipeline (VPS/dokploy) confirmed.
- [ ] **A** End-to-end update test: vX → vY on a running agent.

**Exit gate:** a running agent auto-updates to a newly released version without
manual touch; bad signatures are rejected.

---

## Phase 7 — Hardening & edge cases  ← *Opencode*

**Goal:** production resilience. (Edge-case list lives in DECISIONS/notes; expand
as found.)

- [ ] Multi-monitor; fast user switching; RDP/remote sessions.
- [ ] Laptop sleep/hibernate/resume; clock changes; timezone travel.
- [ ] Long offline periods near buffer cap.
- [ ] Browser variants / hardened UIA configs; PWAs / Electron apps.
- [ ] Token revocation + re-enroll flow; org-key rotation.
- [ ] AV false-positive review of the signed binary.
- [ ] Load sanity at 100 devices.

**Exit gate:** edge-case checklist verified; no data loss or noticeable footprint.

---

## v2.0.0 — Screenshots  *(future)*

Randomized **10–15 min** capture, skip-on-idle, ~1280px **WebP** @ ~55%, all
monitors stitched, uploaded to **Cloudflare R2** via presigned URLs, **7-day**
retention. Dashboard gallery + timeline thumbnails. (Mind fullscreen-exclusive /
DRM black-frame capture.)
