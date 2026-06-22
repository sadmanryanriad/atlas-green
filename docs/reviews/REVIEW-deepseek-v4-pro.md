# Atlas Green — Pre-Implementation Design Review

**Model:** deepseek-v4-pro
**Date:** 2026-06-22

## Executive Summary

The architecture is well-grounded: the Service+Helper split is the correct pattern for silent Windows monitoring, the 3-state activity model is a genuine improvement over binary active/idle, and the ACK-before-purge sync with `batchId` idempotency is the right reliability primitive. The API contract is clean and covers the critical paths. That said, four gaps demand attention before coding begins: (1) the `batchId` idempotency check must include content-hash verification to prevent silent data loss from a content-mutated retry; (2) `batchId` uniqueness must be scoped to `deviceId` — two devices can collide on the timestamp+seq format; (3) the silent-running contract is threatened by UIA accessibility restrictions and the absence of a session-detection design to handle fast user switching; (4) the MSI packaging is deferred to Phase 5, which risks late discovery of self-contained publish issues. Several medium-gap items (buffer encryption method unspecified, clock-skew correction strategy undefined, auto-update swap mechanics vague) should be resolved before their respective phases.

---

## Critical

### C1 — `batchId` idempotency check lacks content-verification; retry with mutated content = silent data loss
**File:** `docs/API-CONTRACT.md` §2, `docs/ARCHITECTURE.md` §3 step 6
**Problem:** The ingest endpoint deduplicates on `batchId` alone (unique index or upsert). If the agent crashes, recovers with partial state, or has a bug that regenerates a previously-used `batchId` with *different* intervals, the server quietly returns `duplicate: true` and drops the new data. The existing ACK-before-purge discipline does not protect against this — the agent sees a 200, marks rows synced, purges them, and the new intervals are lost forever.
**Recommendation:** Include a `contentHash` field in the ingest payload — a SHA-256 of the serialized intervals array. On the server, when `batchId` is known, verify `contentHash` matches the stored hash. On mismatch: return `409 Conflict` so the agent retries with a fresh `batchId` (and ideally logs the anomaly as a critical bug). Store `contentHash` alongside `batchId` in the dedup record.

### C2 — `batchId` format has no `deviceId` scope; cross-device collisions are plausible
**File:** `docs/API-CONTRACT.md` §2 ("`batchId`: `b-2026-06-22T09:10:00Z-001`")
**Problem:** The batchId format `b-<timestamp>Z-<seq>` is only unique *per agent instance*. Two devices with the same wall clock and sequence counter will generate identical `batchId`s. If the server treats `batchId` as globally unique (single unique index on `batchId`), one device's ingest gets silently discarded as a duplicate of the other's — permanent data loss with no error. At 100 agents syncing every 5 minutes, collisions are unlikely but not impossible (especially after reboot when seq resets to 001).
**Recommendation:** Either (a) prepend `deviceId` to `batchId` (e.g., `b-7f3a9c2e-2026-06-22T09:10:00Z-001`) — simplest and makes the idempotency key self-scoping, or (b) make the server-side unique index compound on `{deviceId, batchId}`. Option (a) is preferred because it makes the invariant obvious to future maintainers.

### C3 — Helper launch/session-management design unspecified; fast user switching and RDP will break silent capture
**File:** `docs/ARCHITECTURE.md` §2, `docs/PLAN.md` Phase 1
**Problem:** The plan says "service launches + supervises" the helper and "restart on death / on user login." But launching a process from Session 0 into a user session on Windows requires `WTSQueryUserToken` + `CreateProcessAsUser` — a non-trivial dance with specific token-privilege requirements and careful handle cleanup. Worse, the helper must be relaunched on session-change events (fast user switching, RDP connect/disconnect), but the plan defers these edge cases to Phase 7 (`docs/PLAN.md` line 163). If the session-change detection (`WTSRegisterSessionNotification`) and relaunch logic aren't designed into the supervision loop from Phase 1, Phase 7 will require significant rework of the helper supervisor.
**Recommendation:** Add a session-change detection design to Phase 1. The service should register for `WTS_SESSION_CHANGE` notifications and relaunch the helper into the new active session. Define which session to monitor when multiple are active (the console/physical session, not RDP, per the "one employee, one desktop" model). Move the session-switch restart test from Phase 7 into the Phase 1 exit gate — it is core correctness, not an edge case.

### C4 — Auto-update binary swap mechanics undefined; risk of partial/failed swap corrupting the agent
**File:** `docs/ARCHITECTURE.md` §9, `docs/DECISIONS.md` #17
**Problem:** The update flow says "verify signature → swap → restart." The agent is a .NET self-contained single-file `.exe`. A running `.exe` cannot overwrite itself on Windows. The standard pattern — download to a staging path, then either use `MoveFileEx(MOVEFILE_DELAY_UNTIL_REBOOT)` (requires a reboot) or stop the service, rename the old binary, rename the new binary in place, and restart — is not specified. Without a concrete swap strategy, Phase 6 implementation will stall on basic platform constraints that should be designed now.
**Recommendation:** Document the update swap strategy explicitly in `docs/ARCHITECTURE.md` §9. Recommended pattern: download new `.exe` to `AtlasAgent\update\atlas-agent-v0.2.0.exe`, verify signature + SHA-256, write a `pending-update.json` marker, signal the service to stop (or call `sc stop`), have a small bootstrap/launcher process perform the atomic rename + restart, or use the service's own `OnStop` to perform the rename before terminating. The SCM auto-restart then launches the new binary. This avoids a mandatory reboot.

---

## High

### H1 — SQLite encryption method unspecified
**File:** `docs/DECISIONS.md` #8, `atlas-agent/AGENTS.md` §Tech choices
**Problem:** "SQLite, encrypted" is stated everywhere, but no encryption mechanism is chosen. SQLite has no built-in encryption. Options: SQLCipher (mature, requires a custom/native build), `Microsoft.Data.Sqlite` with a SEE/WX extension (paid license), or application-layer encryption (encrypt each row's payload before insert, decrypt on read). The decision materially affects the NuGet dependency tree and the buffer implementation approach. Without it, Phase 2 cannot write correct buffer code.
**Recommendation:** Choose and document the encryption approach in `docs/DECISIONS.md`. For this threat model (standard users only), **application-layer encryption** using .NET's `AesGcm` with a random key stored via DPAPI machine scope (alongside the device token) is the simplest — zero native dependencies, fully managed, and the ACL on the ProgramData folder already blocks file-level read. If file-level encryption (SQLCipher) is preferred for defense-in-depth, add it as a decision with the build/CI implications (native SQLite extension packaging for self-contained publish).

### H2 — `batchId` collision probability needs jitter on sequence reset
**File:** `docs/API-CONTRACT.md` §2
**Problem:** Even with `deviceId` scoping (C2 above), the `batchId` sequence number resets on agent restart. An agent that crashes and restarts within the same second would generate the same `batchId` with different content, triggering C1 or causing a spurious dedup. The sequence counter persistence (where is it stored?) is not specified.
**Recommendation:** Persist the last-used sequence counter in the SQLite buffer (or registry) so it survives agent restart without resetting. Add millisecond precision to the timestamp component. Include a random nonce suffix as a final backstop.

### H3 — No clock-skew correction strategy; agents with wrong clocks produce permanently wrong data
**File:** `docs/API-CONTRACT.md` §5, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §6
**Problem:** The design explicitly says "the agent trusts its own UTC clock for timestamps" and "never rewrites already-buffered timestamps." The server returns `serverTimeUtc` "to detect/sync drift" but no correction is applied. If an agent's CMOS clock is set to the wrong year (a common issue after CMOS battery failure or manual misconfiguration), all intervals will have permanently wrong `startUtc`/`endUtc` values. The dashboard will show activity in the wrong day/week/month, and "indefinite retention" will carry bad data forever.
**Recommendation:** Add a clock-skew policy: on every sync/heartbeat, if `|serverTimeUtc - localUtc| > N` (e.g., 5 minutes), log a warning. If `> M` (e.g., 1 hour), either (a) flag the device as "clock-skewed" in the dashboard, (b) store a `clockOffsetMs` on the device record and have the dashboard apply it at display time, or (c) have the agent correct its buffer timestamps if the offset is stable across two consecutive heartbeats. Option (a) + (b) is the least intrusive and preserves raw data integrity.

### H4 — UI Automation may silently fail on hardened browsers; zero fallback
**File:** `docs/PLAN.md` Phase 3, `docs/AGENTS.md` §3
**Problem:** The URL capture strategy hinges entirely on UI Automation reading the browser address bar. Windows 11 security hardening, browsers running at higher integrity levels or with app-container isolation (Edge's "enhanced security mode"), and Chrome's historically shifting stance on accessibility API support all threaten UIA availability. If UIA returns an empty/placeholder address bar pattern for *any* browser on *any* machine, the entire URL feature produces no data — and the silent-running mandate means no user notification that it's broken.
**Recommendation:** Add a fallback chain: (1) UIA (primary), (2) window-title regex extraction (many browsers include the domain or page title in the window title — Chrome and Edge definitely do), (3) if both fail, log the window title as-is and flag the interval with `browser: null, domain: null`. Also add browser-version detection to the heartbeat so the dashboard can show "Chrome 128+ may block UIA" warnings per device. Test against Edge enhanced-security mode specifically.

### H5 — No sync jitter; 100 agents may thundering-herd the console on startup/recovery
**File:** `docs/ARCHITECTURE.md` §3 step 4, `docs/PLAN.md` Phase 2
**Problem:** The sync interval is a fixed 5 minutes. If all 100 agents boot simultaneously (power outage recovery, scheduled 9 AM office start), they all sync at exactly T+0, T+5, T+10 minutes, creating 100 concurrent `POST /ingest` calls. Next.js on a VPS handles this easily, but the exponential-backoff retry after a server hiccup also lacks jitter — all failing agents could retry in lockstep. This is a minor load concern but a well-understood anti-pattern.
**Recommendation:** Add ±30 s random jitter to the sync interval on each cycle. Add jitter to the exponential backoff (randomize within the backoff window, e.g., 5-10s for the first retry). Document in `docs/ARCHITECTURE.md` §3.

### H6 — Org key rotation mechanism not designed
**File:** `docs/PLAN.md` Phase 7 ("org-key rotation"), `docs/DECISIONS.md` #15
**Problem:** "Org-key rotation" is listed as a Phase 7 edge case. If the org key is compromised, every existing agent still has a valid per-device token and continues to work — good. But new enrollments (reformatted machines, new hires) need the new key, and agents that lose their token (DPAPI corruption, re-enroll) need to re-enroll. A single-key swap means agents re-enrolling during the transition window will fail. The rotation protocol needs a design beyond "Phase 7 will handle it."
**Recommendation:** Document the rotation protocol: the server accepts both `old-key` and `new-key` during a transition window (e.g., 30 days). The heartbeat response returns a config flag `orgKeyTransition: true` to tell agents "your token is still valid, but be aware the org key is rotating — if you ever need to re-enroll, use the new key in your MSI." After the transition window, `old-key` is rejected. This requires the `settings` collection to support multiple active org keys with expiration dates.

---

## Medium

### M1 — Buffer encryption key lifecycle tied to DPAPI; key loss = data loss
**File:** `docs/ARCHITECTURE.md` §6, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §5
**Problem:** The buffer encryption key (whatever form it takes) must persist across agent restarts. If stored via DPAPI machine scope, it's implicitly tied to the machine's DPAPI master key. A Windows update, a local security policy change, or a domain-join event can invalidate DPAPI keys. If the buffer encryption key becomes unrecoverable, all buffered (unsynced) intervals are lost. The plan doesn't address DPAPI failure recovery.
**Recommendation:** Store a DPAPI-wrapped copy of the buffer encryption key, AND a secondary recovery copy in the ACL-locked registry (ACL sufficient for standard-user threat model). If DPAPI unwrap fails, try the registry copy. This adds no meaningful attack surface (standard users can't read either) but prevents data loss from DPAPI key invalidation.

### M2 — MongoDB collection document shapes missing validation
**File:** `docs/ARCHITECTURE.md` §7
**Problem:** The architecture says "Indicative shapes; the implementing repo owns exact schemas/validation." While true, the absence of any mention of MongoDB schema validation (JSON Schema validators on the collection) means the ingest endpoint could accept garbage intervals (missing `startUtc`, wrong `state` enum, negative `durationSec`) and store them permanently. The zod validation at the API layer should be the first line of defense, but collection-level validation is a safety net.
**Recommendation:** Add MongoDB collection-level JSON Schema validators for `intervals` (enforce required fields, enum values, UTC timestamp format) and `devices` (enforce DeviceId format). Document in `docs/ARCHITECTURE.md` §7.

### M3 — Dashboard "per-employee daily timeline" aggregation across UTC days is ambiguous for multi-timezone teams
**File:** `docs/ARCHITECTURE.md` §8, `docs/PLAN.md` Phase 4
**Problem:** The design says "aggregations group by UTC; the client converts for display." A Bangladesh employee (UTC+6) working 9 AM-6 PM produces intervals from 03:00-12:00 UTC — all within one UTC day. But aggregating "daily activity" in UTC when an admin in a different timezone views the data means the admin sees the employee's UTC-day summary, not the employee's local-day summary. For Bangladesh, this happens to align (the employee's midnight is 18:00 UTC the prior day, and their 6 PM end-of-day is 12:00 UTC — all within the same UTC day). But for the admin in UTC+0, "Tuesday" for the employee is actually "Monday evening through Tuesday afternoon" — confusing.
**Recommendation:** Decide explicitly: are dashboard aggregations by the *viewer's* local day (simplest), or by the *employee's* local day (requires storing the employee's timezone offset)? For v1 with most employees in one zone (Bangladesh, UTC+6), the viewer's-local-day approach works. Document the decision in `docs/ARCHITECTURE.md` §8 to prevent Phase 4 implementation drift.

### M4 — No `updateStatus` field in heartbeat; agent can't report update outcomes
**File:** `docs/API-CONTRACT.md` §3
**Problem:** The heartbeat request body lacks an update-status field. The design says "A failed verification... is reported on next heartbeat" (API-CONTRACT.md §4). But there's no `updateStatus`, `lastUpdateAttemptUtc`, or `updateError` field. This means the dashboard has no visibility into update failures across the fleet — a critical operations blind spot.
**Recommendation:** Add optional fields to the heartbeat request: `updateStatus: "idle" | "downloading" | "verifying" | "success" | "failed"`, `lastUpdateAttemptUtc`, `updateError` (string). The dashboard heartbeat panel can then show devices with failed updates.

### M5 — `machineHints` in enrollment collects data that v1 explicitly does not use
**File:** `docs/API-CONTRACT.md` §1, `docs/DECISIONS.md` #14
**Problem:** The enrollment payload includes `machineHints.machineGuid` and `machineHints.mac`, but Decision #14 says "no hardware dedup in v1." This data is collected, transmitted, and stored without any v1 purpose. It is a privacy/security liability (the server now has hardware fingerprints it doesn't need) and adds implementation complexity (MAC address retrieval across multiple adapters, error handling, etc.).
**Recommendation:** Remove `machineHints` from the v1 enrollment schema. Re-add it in the phase where hardware dedup is actually implemented. If the argument is "collect now so we have history later," the history won't include machines enrolled before dedup was built, so there's no value.

### M6 — Auto-update abort-on-failure doesn't handle partial download resume
**File:** `docs/ARCHITECTURE.md` §9
**Problem:** The update flow says "download from R2 → verify signature → swap → restart; abort safely on failure." A 90 MB `.exe` download over a slow or unstable connection (common in Bangladesh) can fail mid-transfer. Without range-request resume support, the agent retries from byte 0, wasting bandwidth and potentially never completing on a flaky connection.
**Recommendation:** Use HTTP Range requests (`If-Range`) to resume interrupted downloads. Store the partial download + expected SHA-256 in a temp file; on resume, verify the existing bytes' SHA-256 matches the expected partial hash (or just trust the byte count + re-download if mismatched). Document in `docs/ARCHITECTURE.md` §9.

### M7 — Interval duration sanity bounds not specified
**File:** `docs/API-CONTRACT.md` §2
**Problem:** The aggregator produces intervals by flushing on app/domain/state change. If a user leaves a media stream playing for 8 hours, the interval `durationSec` is 28,800. This is correct behavior, but the API contract doesn't specify maximum bounds. The ingest endpoint should have a ceiling (e.g., 86,400s = 24 hours) and reject or clamp intervals exceeding it, which would indicate a clock bug or aggregator defect.
**Recommendation:** Add a note to `docs/API-CONTRACT.md` §2: intervals with `durationSec > 86400` (24h) should be rejected with `400` and logged as a server-side alert. The agent should also sanity-check its own intervals before buffering.

### M8 — No alert/logging when buffer cap is hit and data is dropped
**File:** `docs/ARCHITECTURE.md` §3, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §4
**Problem:** The buffer cap (~2-3 months) is a safety net. When it's hit (long offline period), the plan says "dropping oldest only as a last resort." There's no mention of logging a warning or surfacing this condition on the next heartbeat. If an agent silently drops 2 months of data because the cap was exceeded, the first anyone knows is when dashboard data has a gap — with no indication of WHY.
**Recommendation:** When the agent drops rows due to cap pressure, log a structured warning with the count and date range of dropped rows. On the next successful heartbeat or ingest, include a `droppedIntervals` count + date range so the dashboard can show "Data gap: 45 days dropped due to buffer overflow" on that device's timeline. Add fields to the heartbeat request.

---

## Low

### L1 — `osUsername` duplication in interval creates denormalization without rules
**File:** `docs/API-CONTRACT.md` §2, `docs/ARCHITECTURE.md` §7
**Problem:** The `osUsername` field appears on both the `devices` collection and every `intervals` document. Denormalizing it on intervals is fine for query performance (avoid a join), but there's no procedure for what happens when `osUsername` changes on the device record — do historical intervals get backfilled? Do they retain the old value? Without a rule, the data gets inconsistent.
**Recommendation:** Document: interval `osUsername` is a snapshot at capture time and is **never backfilled**. Dashboard queries that need the current `osUsername` should read it from the `devices` collection. The interval's `osUsername` is kept for auditing (who was logged in at that moment). Add a note to `docs/ARCHITECTURE.md` §7.

### L2 — Phase 0 has no SQLite or buffer skeleton; Phase 2 starts from scratch
**File:** `docs/PLAN.md` Phase 0
**Problem:** Phase 0 includes heartbeat and enrollment (the wire plumbing) but not even a SQLite buffer skeleton. Phase 2 then adds the full buffer + sync loop. This means the data model for interval storage doesn't exist until Phase 2, and Phase 1 (which generates intervals locally) has nowhere to persist them except logs. This is workable but means Phase 1 cannot test interval persistence at all.
**Recommendation:** Consider adding a minimal SQLite schema migration + open/write/read skeleton to Phase 0 (after enrollment). It doesn't need encryption or the sync loop — just creating the database file, creating the `intervals` table, and inserting/reading a test row. This gives Phase 1 a persistence target and validates the SQLite library choice early.

### L3 — Token revocation doesn't specify what happens to buffered data
**File:** `docs/API-CONTRACT.md` §5, `docs/ARCHITECTURE.md` §6
**Problem:** When a device token is revoked (admin action), the agent gets a `401` on its next API call, attempts a single re-enroll, and if that fails "backs off and keeps buffering." But the buffered data will never be accepted (the device is revoked). The agent will buffer forever until the cap drops old data. There's no terminal state.
**Recommendation:** After X consecutive re-enroll failures (e.g., 5 attempts over several hours), the agent should enter a "revoked" state: stop generating new intervals, flush existing buffer to a cold-storage backup file, and only heartbeat (to keep the device visible as "offline/revoked"). Document this flow in `docs/API-CONTRACT.md` §5.

### L4 — `GET /api/v1/update/manifest` endpoint misplaced under API routes
**File:** `docs/API-CONTRACT.md` §4, `docs/ARCHITECTURE.md` §9
**Problem:** The manifest is served from Cloudflare R2, not from the console API. Yet the API contract lists `GET /api/v1/update/manifest` as a console endpoint. The architecture diagram shows R2 serving the manifest directly. These are contradictory: either the manifest is an API route (served by Next.js) or an R2 static file (served by Cloudflare). The heartbeat response already provides the `manifestUrl` pointing to R2, so a console endpoint for the manifest is redundant.
**Recommendation:** Remove `GET /api/v1/update/manifest` from the API contract. The heartbeat response's `update.manifestUrl` is the sole source for the manifest location. If the console needs to serve the manifest (e.g., before R2 is set up), make that a deployment detail, not part of the permanent contract.

### L5 — No mention of structured log levels or sensitive-data redaction
**File:** `atlas-agent/AGENTS.md` §Conventions, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §1
**Problem:** The agent logging rules say "Never log secrets (org key, device token) or full page contents." But there's no mention of log levels (Debug/Info/Warning/Error), structured logging field names, or redaction for non-secret sensitive data (window titles containing file paths or personal info). Without guidance, developers may log full window titles at Info level, creating a privacy issue even though they're not "secrets."
**Recommendation:** Add to the agent conventions skill: log window titles at `Debug` level only in production; at `Information` level only in dev. Define a `SanitizedWindowTitle` log property that truncates to 80 chars. Define a `SensitiveData` attribute or marker interface for DTO properties that should never appear in logs.

### L6 — No delivery-guarantee specification for heartbeats
**File:** `docs/API-CONTRACT.md` §3
**Problem:** Heartbeats have no retry logic specified. If a heartbeat fails (network blip), does the agent retry or wait for the next cycle? If the heartbeat is the sole driver of online/offline status, a single failed heartbeat could briefly show the device as offline. The plan says devices not seen within "~3× the heartbeat interval" are offline — which is 15 minutes at 5-min intervals. A single dropped heartbeat is tolerated, but the agent should still retry.
**Recommendation:** Specify: heartbeats retry once after 30 s on failure, then wait for the next scheduled cycle. This is lightweight and prevents brief network hiccups from creating 5-minute gaps.

### L7 — Phase ordering: MSI packaging (Phase 5) is too late for self-contained publish validation
**File:** `docs/PLAN.md` Phase 5
**Problem:** Phase 5 is the first time the agent is built as a self-contained single-file `.exe` and packaged as an MSI. If `dotnet publish --self-contained` produces a binary that fails on clean Windows (missing VC++ redistributable, trimming issues, native library linkage failures), it's discovered only after all capture/sync/dashboard logic is built. This is a high-cost failure point.
**Recommendation:** Add a "Phase 0.5" task: `dotnet publish --self-contained` + run the resulting `.exe` on a clean test VM (no .NET SDK installed) to validate the single-file publish works. This is a 30-minute task that saves massive rework. Keep MSI packaging in Phase 5, but validate self-contained publish in Phase 0.

### L8 — No database migration strategy for MongoDB schema evolution
**File:** `docs/ARCHITECTURE.md` §7, `atlas-console/AGENTS.md` §Data model
**Problem:** As the product evolves, the interval document shape, new collections, or index changes will be needed. There's no mention of a migration tool (e.g., `migrate-mongo`, Mongoose migrations, or a custom script). Without a migration strategy, ad-hoc changes accumulate and the database drifts from what the code expects.
**Recommendation:** Choose a lightweight migration approach for MongoDB. For a project this size, a simple `migrations/` folder with numbered-up scripts and a `migrations` collection to track applied migrations is sufficient. Document in `atlas-console/AGENTS.md` under "Data model."

---

## Nice-to-Have

### N1 — Dashboard could expose raw interval export (CSV/JSON) for admin auditing
The plan defers CSV/PDF export post-v1. A raw JSON export of intervals for a date range (via the dashboard) would give admins an escape hatch for their own analysis without waiting for v2 dashboard features.

### N2 — Consider a `/api/v1/config` endpoint for push-based config updates
Currently config changes only reach the agent at heartbeat time (up to 5-min delay). A dedicated config endpoint the agent polls (or the heartbeat response already covers this) is sufficient, but if near-instant config push is ever needed, a WebSocket or long-poll endpoint would solve it. Not needed for v1.

### N3 — GPU/VRAM monitoring for gaming detection
Gaming produces input → `active`, which is correct. But admins might want to differentiate "active in Excel" from "active in a game." Adding GPU utilization or VRAM usage to the helper's sampling (low cost via `PerformanceCounter`) could add a lightweight "gaming" hint to intervals without changing the 3-state model. V2 consideration.

### N4 — Named pipe message framing design should be documented
The helper-to-service transport is "named pipe" with no framing specification (length-prefix? newline-delimited JSON? protobuf?). This is an implementation detail but getting it wrong creates hard-to-debug IPC bugs (partial reads, buffer overruns). A paragraph in `docs/ARCHITECTURE.md` §2 would prevent implementor thrash.

### N5 — Auto-update should support a "downgrade" path for emergency rollbacks
The manifest specifies `latestVersion` with `minSupportedVersion`. If a bad update is pushed, the CI pipeline should support setting `latestVersion` to a prior version and bumping the package URL to the older binary, so agents auto-downgrade. The update client logic needs to handle "latestVersion < currentVersion" as a valid downgrade signal.

### N6 — Agent installer should register with Windows Event Log for IT visibility
While the agent is silent to the user, IT staff running the MSI should be able to verify installation success/failure via Windows Event Log entries. The MSI custom action can write install/uninstall events to the Application log, and the agent can write startup/shutdown/fatal-error events.

---

## Where the Design Is Solid

The following decisions are well-reasoned and need no change:

- **Three-state activity model** (`active`/`passive_media`/`idle` via GSMTC + WASAPI fallback). Correctly distinguishes "watching media" from "away" — more honest than binary idle detection and captures the most common ambiguous scenario (video consumption).
- **ACK-before-purge sync discipline with `batchId` idempotency.** The core reliability primitive is correctly designed; the recommendations above (C1, C2, H2) are hardening, not redesign.
- **Per-device tokens via DPAPI machine scope.** Correct for this threat model. Single-device revocation without fleet-wide key rotation is the right property.
- **DeviceId as opaque GUID with mutable attributes.** The identity model is clean: identity never changes, labels can. Reformatted machine = new device is honest and simple.
- **Self-contained single-file .NET 8 publish.** The right choice for "target PCs need nothing pre-installed." No .NET runtime dependency on 100 machines is the correct operational posture.
- **Sampling model (interval aggregation, Model B).** ~100-300 rows/day per device vs ~17k raw samples is a massive reduction with no information loss. Makes queries trivial and payloads tiny.
- **Service + Helper split.** Session 0 isolation for persistence/secrets/network + user-session helper for observation is the standard Windows monitoring pattern. Helper holds no secrets, does no networking — correct separation of concerns.
- **Cloudflare R2 for update packages.** Zero egress cost, CDN-fast for remote staff, decoupled from the console VPS. At 80 MB × 100 agents = 8 GB per release wave, R2's zero-egress model is a real cost savings.
- **Phase 0 as thin vertical slice.** Proving the enrollment → heartbeat → dashboard "online" loop before building capture is the right risk-reduction strategy. The exit gate (live "online" in dashboard) is concrete and falsifiable.
- **Test policy: unit-test pure logic, manually verify OS-interop.** Pragmatic. UIA/GSMTC reads are inherently integration-level; trying to mock them would produce tests that pass CI but fail on real hardware.
- **Agent runs as console in dev, service in prod.** This is critical for developer velocity — the capture loop is debuggable with live terminal output, but silent in production.

---

## Summary of Recommendations by Phase Impact

| Phase | Issues to resolve before starting |
|-------|----------------------------------|
| **Phase 0** | L7 (validate self-contained publish early), L2 (SQLite skeleton), N6 (Event Log registration) |
| **Phase 1** | C3 (session-change detection in helper supervision) |
| **Phase 2** | C1 (batchId content hash), C2 (batchId deviceId scoping), H1 (SQLite encryption method), H2 (batchId seq persistence), H5 (sync jitter), M1 (buffer encryption key recovery), M8 (cap-hit logging) |
| **Phase 3** | H4 (UIA fallback chain), L5 (window title redaction) |
| **Phase 4** | M3 (timezone aggregation semantics) |
| **Phase 5** | L3 (revoked-device terminal state) |
| **Phase 6** | C4 (update swap mechanics), H3 (clock-skew correction), H6 (org key rotation protocol), M4 (heartbeat update status), M6 (download resume), L4 (remove redundant manifest endpoint) |
| **Phase 7** | M2 (MongoDB schema validation), M5 (remove v1 machineHints), M7 (interval duration bounds), L1 (osUsername snapshot rule), L6 (heartbeat retry), L8 (DB migration strategy) |
