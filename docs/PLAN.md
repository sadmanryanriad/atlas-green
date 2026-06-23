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
- [ ] **A** Prod logs go to an **ACL-locked** `C:\ProgramData\AtlasGreen\logs\` (not
      user-readable — titles/process names aren't secrets but are sensitive).
- [ ] **C** Next.js app scaffolded; **MongoDB single-node replica set** via a
      deterministic init script + dev `docker-compose` (`--replSet rs0`,
      `rs.initiate()` bound to **`127.0.0.1`**, conn string `…?replicaSet=rs0`).
- [ ] **C** `/api/v1/enroll`, `/api/v1/heartbeat`, **`/api/v1/health`** per contract.
- [ ] **C** **`settings` collection schema** seeded (orgKeys array, agentDefaults,
      rateLimits incl. `enrollAllowlistCidrs`, retention, **IANA-validated** orgTimeZone).
- [ ] **C** **Admin auth from day one**: Cloudflare Access + a `middleware.ts` that
      rejects unauthenticated `/dashboard/*`, with a **dev bypass** behind an env flag
      (`DEV_AUTH_BYPASS=…`, never in prod). *(was unplanned)*
- [ ] **C** **`auditLog` collection + write helper** (infra only); the device→employee
      mapping action writes an audit entry from day one. Viewer is Phase 4. *(r2: minimax H3)*
- [ ] **C** Minimal dashboard page lists devices with **online/offline + last-seen**.
- [ ] **C** Device → employee mapping stub (assign a name to a device).
- [ ] **C** **Dev org key + secrets**: seed script creates one dev org key; the agent
      reads it from an env var / local dev config (never committed).
- [ ] **C** **Dev seed script** (fixed-seed). Intervals: one device with 24 h —
      all 3 states, 5+ domains incl. `null`, an interval crossing the Dhaka 18:00-UTC
      midnight, one >4 h, one <10 s. **Device state matrix** so the dashboard is built
      against the full liveness+flags space: `pending`; `online+[]`; `offline`; and one
      each of `online+[outdated]`, `[clock_skewed]`, `[revoked]`, `[archived]`,
      `[health_warning]`, `[possible_clone]`, `[tamper]` (the revoked & archived ones
      also carry intervals, to exercise the paused-but-has-history UX).
- [ ] **A+C** Test scaffolding wired **with one real smoke test each** (`[Fact]` /
      `it()` that runs and passes — "0 tests" is not a green toolchain).
- [ ] **M** READMEs updated with "how to run in dev" for both repos.

**Exit gate:** `dotnet run` (agent) enrolls and heartbeats against `npm run dev`
(console); the device appears **online** in the dashboard and flips **offline**
when the agent stops; **an unauthenticated browser cannot view `/dashboard`**; the
published single-file `.exe` runs on a clean VM.

---

## Phase 1 — Core capture (local only)  ← *Opencode*

**Goal:** the agent accurately produces aggregated activity **intervals** locally,
with the correct three-state model. No server changes. **Split into 1a/1b** (the
heaviest, riskiest phase) so Phase 2's SQLite/schema work can start against 1a's
basic intervals while 1b is in progress.

### Phase 1a — Helper + transport + foreground sampling

- [ ] **A** User-session **helper** launched from the service via **`WTSQueryUser
      Token` + `CreateProcessAsUser`** (service **LocalSystem**, helper as the user,
      `lpDesktop="Winsta0\\Default"`, `CREATE_NO_WINDOW`); launched **only for a real
      user token** (not the logon/lock screen); reacts to **`WTS_SESSION_CHANGE`**;
      **console session only**, RDP ignored. Watchdog restarts a dead helper.
- [ ] **A** Helper runs **windowless** in prod config; verified on a non-admin account.
- [ ] **A** Foreground **window + process** sampling every 3–5 s; **ignore foreground
      hijackers** (`Consent.exe`/UAC, `LogonUI`, elevated `TaskMgr`, `WerFault`).
- [ ] **A** **`appName` from the exe `FileDescription`** (fallback = exe name); each
      interval tagged with the Windows **`sessionId`**.
- [ ] **A** Helper → service **length-prefixed JSON named pipe**; service **verifies
      the peer** (PID → image path → signature) and dedupes by **`(instanceId, seq)`**
      from a connect handshake (so a helper restart can't drop the first samples).
- [ ] **A** Heartbeat carries **capture health** (`helperRunning`, `lastSampleUtc`,
      `helperStartedAtUtc`, `helperRestartCount`).

**1a exit gate:** the helper launches into a real user session (verified non-admin),
relaunches on a fast user switch, and foreground samples reach the service over the
verified pipe — visible in logs. *(Phase 2's **server-side** work — `/ingest`, the
`batches` collection + atomic txn, rate limits, org-key rotation, the liveness+flags
state machine — may now begin in parallel against a test harness. Phase 2's
**agent-side** work — SQLite buffer, sync loop, dropped-byState, token-loss recovery —
starts after **1b**, since it's only testable against the aggregator's real
intervals.)* The prod pipe **signature** check needs the Phase 5/6 cert; in dev it's
skipped behind a flag (image-path check only) — see ARCHITECTURE §2.2.

### Phase 1b — State machine + aggregator

- [ ] **A** **Idle** via `GetLastInputInfo`: enter `idle` after **300 s with no
      meaningful input**; only a **meaningful input quantum** (≥~2 s sustained or ≥3
      events / 10 s; mouse-move < 4 px ignored) resets the timer / **leaves idle** — a
      single isolated event (jiggle, poll) does not.
- [ ] **A** **Tamper detection**: flag known keep-awake/jiggler processes (Caffeine,
      PowerToys Awake, mouse-jiggler utils, obvious `SendKeys` loops) → `tamperHints`
      on the heartbeat (server raises the `tamper` flag).
- [ ] **A** **Media** via **GSMTC** (Playing?) → `passive_media`; **gated WASAPI
      fallback** (recent ≤60 s non-comms GSMTC source); **an active comms call always
      overrides** (calls are never `passive_media`); media transitions bypass hysteresis.
- [ ] **A** **Three-state machine** (`active` / `passive_media` / `idle`).
- [ ] **A** **Interval aggregator** (Model B): flush on **HWND**/app/domain/state
      change; idle **splits**; **durations from a monotonic clock** (`durationSec`
      canonical); **close on sleep via `SERVICE_CONTROL_POWEREVENT`** (sub-30 s blip
      debounce; monotonic sanity force-close for missed/Modern-Standby events);
      **re-derive state from the first post-resume sample** (not hardcoded `idle`).
- [ ] **A** **Graceful shutdown**: on `ApplicationStopping`, flush the open interval.
- [ ] **A** **Unit tests** (pure logic): the **call-vs-media gating**, **idle
      reset-on-meaningful-input** (no oscillation), **bursty-input call stays active**,
      **isolated jiggle every 60 s → stays `idle`** (the keep-awake case),
      **sleep-resume with media auto-resume**, **force-close splits to `idle` after the
      last sample** (no phantom multi-hour `active`), eTLD+1 normalization stub.
- [ ] **M** README: what's captured and how to watch it live in dev.

**1b exit gate:** switching apps / playing a video / **being on a Teams call (with
Spotify also playing)** / going idle / sleeping+resuming produces correct intervals
and states (the call is `active`/`idle`, **never** `passive_media`); verified live.

---

## Phase 2 — Buffer + sync  ← *Opencode*

**Goal:** intervals persist to SQLite and reliably reach MongoDB via `/ingest`.

- [ ] **A** **SQLite** buffer with **app-layer AES-GCM** encryption (key via **DPAPI
      machine scope**; no native dep), ACL-locked under `C:\ProgramData\...`.
- [ ] **A** Append closed intervals; `synced` flag; buffer cap. A cap drop is **never
      silent**: log it + report `dropped {count, oldest, newest, byState}` (prefer
      dropping `idle`); the server persists it as a `dataGap`. Also enforce a
      **retention floor**: drop unsynced rows older than `min(bufferCapDays,
      serverBatchesTtl)` (so a corrupted/long-offline buffer can't replay past an
      evicted receipt). *(was Ph7)*
- [ ] **A** **Sync loop** (5 min **±30 s jitter**): gzip batch with a **UUID
      `batchId`** → `/ingest`; **ACK before purge** (verify response `batchId`). The
      agent **self-splits before send** when over the size cap (fresh UUID per
      sub-batch, recursive halving), spacing sub-batches per the rate limit.
- [ ] **A** Exponential backoff **+ jitter**; offline accumulation; honor `Retry-After`.
- [ ] **A** **Config validation**: clamp/ignore out-of-range server config values.
- [ ] **A** **Token-loss recovery**: re-enroll sends `Authorization` iff a token still
      exists (→ no-op); a lost token → re-enroll **rotates** (new token), buffered rows
      then sync. Re-enrolls are **sequential** + capped (~5/h). On a sticky
      `reauthorized`, **resume `/ingest` on the existing token first**, re-enroll only
      if that 401s; a `401` on `/enroll` (bad org key) ≠ a `403` (blocked) — don't pause.
- [ ] **C** `/api/v1/ingest` per contract: validate token **(bound to deviceId)**,
      **atomic per batch** (one txn, ≤1000 intervals, `maxCommitTimeMS`, retry on
      `TransientTransactionError`), write the **`batches` receipt**, resolve
      **category + employeeId** at ingest.
- [ ] **C** Collections + indexes: `intervals` `{deviceId,batchId}` **NON-unique**,
      `{employeeId,startUtc}`; **`batches` unique `{deviceId,batchId}` + TTL on
      `receivedAt`**; `deviceTokens.tokenHash` unique. *(unique-index bug fixed — r2)*
- [ ] **C** **Device state = liveness + flags** (server side), derived from stored
      inputs through one **`updateDeviceState`** path (no second source of truth);
      `/heartbeat` **accepts a revoked/archived token** (→ `flags:[revoked]`/`[archived]`)
      but **freezes capture-health** from a revoked/archived heartbeat; `reauthorized`
      is **sticky** until the agent acts.
- [ ] **C** **Rate limiting** per device/IP; `/enroll` = 10/min/IP with a
      **corp-CIDR allowlist** so the rollout isn't throttled.
- [ ] **C** **Org-key array + rotation** in `settings`; **blocked/revoked/archived
      device → `403`** at `/enroll` (org-key checked first). Three admin actions:
      **revoke**, **archive** (reversible, ingest-blocked, keeps history),
      **`POST /admin/devices/:id/wipe`** (block & wipe — deletes intervals + receipts
      **+ dataGaps**, doc retained blocked, irreversible). *(was Ph7)*
- [ ] **A** Unit tests for batching / retry / purge-after-ack.
- [ ] **C** Integration tests: batch of N stored once; **partial-insert-then-retry**
      never loses/duplicates; stale replay of an admin-deleted batch is rejected;
      **double-rotation race** (two rapid token-loss re-enrolls → same token, one row
      superseded); **end-to-end revocation** (revoke → `/ingest` 401, `/enroll` 403,
      `/heartbeat` 200 `revoked` with frozen capture, dashboard shows revoked) **and
      archive** (un-archive → resume). *(revocation E2E moved here from Ph7)*

**Exit gate:** intervals captured offline flush to MongoDB when connectivity
returns; retried/partial/stale batches never double-store or resurrect data; a
buffer-cap drop is visible on the heartbeat panel; revocation works end-to-end.

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
- [ ] **C** **Day/week bucketing** uses `orgTimeZone` (or `employees.tzOverride`),
      clock-skew offset applied **before** the boundary; summary returns org-tz
      `bucketDate`+`bucketTimeZone` (not client-converted); timeline **rendering** in
      the **viewer's** local zone (no SSR/CSR flicker).
- [ ] **C** **Audit-log viewer** (infra shipped in Phase 0). A **device→employee
      remap** updates **future** intervals; a **separate bulk-reassign** action
      re-resolves a historical range (audit-logged). `osUsername` is never back-filled.
- [ ] **C** Dashboard reads device **liveness + flags** (not re-derived); `revoked`/
      `archived` override the badge; **capture panel muted when `capture.enabled:false`**;
      a **flag filter** on the company overview (chip per flag); a **"dismiss clone"**
      action (audit-logged). Flags a device showing **≥2 distinct `osUsername`s** in
      24 h (possible mis-map).
- [ ] **C** **`dataGaps` surfaced** on the timeline (a greyed band "N intervals
      dropped" on the detection day) + a fleet count of devices with recent gaps —
      so a buffer-overflow gap isn't write-only storage.
- [ ] **C** Upgrade admin auth to **NextAuth** (per-admin accounts/roles) on top of
      Cloudflare Access.
- [ ] **C** Component/integration tests, incl. a **clock-skewed device buckets into
      the correct local day** and a **workday crossing Dhaka midnight** test.

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
- [ ] **A** **MSI upgrade/repair preserves** `C:\ProgramData\AtlasGreen` (DeviceId +
      DPAPI blobs + buffer) — only full uninstall removes it.
- [ ] **M** README/runbook: cert pre-trust to `LocalMachine\Root` — **GPO** on AD
      machines **and** the **PowerShell `Import-Certificate` / MSI custom-action**
      path for **workgroup** PCs; install + uninstall + PC-name steps for IT.

**Exit gate:** clean install on a test machine (incl. one hardened machine) runs
invisibly, captures + syncs, survives reboot/logout, and a standard user cannot
stop it; no SmartScreen/AV prompt with the cert pre-trusted.

---

## Phase 6 — Auto-update + CI/CD  ← *Opencode*

**Goal:** push code once → fleet updates itself.

- [ ] **A** Self-signed code-signing cert + cert-trust process documented (GPO + the
      workgroup PowerShell/MSI path).
- [ ] **A** Update client: heartbeat says "newer exists" → **verify manifest
      signature** → download from **R2 with HTTP Range/resume** → verify package
      **signature + SHA-256** → **swap (staging dir + rename on OnStop, no reboot)** →
      restart; **on startup, finish a stranded swap** (`pending-update` marker) before
      the main loop; abort safely on failure (report `updateStatus: failed`).
- [ ] **A** Honor **`configSchemaVersion`** (ignore-unknown); support **downgrade**
      (`latestVersion < current`); `mandatory:true` updates ASAP but never shuts off
      ingest.
- [ ] **C/M** Publish a **signed `manifest.json`** + package to R2 (release pipeline);
      **R2 keeps the last 5 versions** for downgrade.
- [ ] **A** GitHub Actions (Windows): test → publish → MSI → **sign package +
      manifest** → upload R2 → update manifest.
- [ ] **C** Console deploy pipeline (VPS/dokploy) confirmed; **daily mongodump → R2**
      with a **rehearsed restore** (restore data → `rs.initiate()` → transactions resume).
- [ ] **A** End-to-end update test: vX → vY on a running agent (and a forced
      downgrade vY → vX), incl. a **crash-mid-swap** recovery.

**Exit gate:** a running agent auto-updates to a newly released version without
manual touch; **a tampered manifest or bad signature is rejected**; a resumed
download completes over a flaky link.

---

## Phase 7 — Residual hardening & scale  ← *Opencode*

**Goal:** production resilience for what genuinely couldn't be folded into earlier
phases. (Most former Phase-7 items were redistributed to the phase that owns them —
helper/session/sleep/clock → Ph1, buffer-cap/revocation/org-key → Ph2, browser
variants/DPI → Ph3, AV/AppLocker → Ph5 — to avoid a tar-pit final phase.)

- [ ] Multi-monitor (incl. the `passive_media`-on-non-foreground-monitor attribution
      limit); PWAs / Electron apps (appName edge cases).
- [ ] **100-device soak + chaos test**: 24 h of realistic intervals → zero dropped
      batches, queries < 2 s, stable VPS; **plus chaos** — kill helper on 5%, partition
      the network 10 min (backlog flushes, no dupes), re-enroll 1% mid-stream, inject a
      bogus `batchId` (rejected without affecting others).
- [ ] Resolve any `phase-7`-labelled edge cases triaged during earlier phases.

**Exit gate:** soak + chaos pass at 100 devices; no data loss or noticeable footprint.

---

## v2.0.0 — Screenshots  *(future)*

Randomized **10–15 min** capture, skip-on-idle, ~1280px **WebP** @ ~55%, all
monitors stitched, uploaded to **Cloudflare R2** via presigned URLs, **7-day**
retention. Dashboard gallery + timeline thumbnails. (Mind fullscreen-exclusive /
DRM black-frame capture.)
