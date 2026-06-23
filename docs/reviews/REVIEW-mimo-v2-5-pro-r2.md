# Atlas Green — Iteration 2 Design Review

**Model:** mimo-v2.5-pro
**Date:** 2026-06-23
**Round:** 2 (v1.1 stress-test)

## Executive Summary

The v1.1 revision successfully addressed the most critical findings from the first review round, particularly around `batchId` collision (now UUID), atomic ingest (now transactional), and the missing dashboard auth (now Phase 0). However, the revision introduced new edge-case ambiguities and left several operational concerns underspecified. The most significant new finding is a **state-machine conflict** where idle hysteresis can suppress the WASAPI media-detection fallback during the "sustained input" window, potentially misclassifying media playback as idle. Additionally, the `batches` receipt collection lacks a TTL policy (will grow indefinitely), the sleep/resume handling has an edge case with post-resume media detection, and the single-node replica set transaction failure mode is underspecified. The phase redistribution from Phase 7 was incomplete — several capture-correctness items remain in Phase 7 that should be in Phase 1 or 3. Overall, the plan is sound and Phase 0 can begin, but the state-machine interaction and operational TTL need resolution before Phase 1 and Phase 2 respectively.

---

## Critical

### C1 — Idle hysteresis "sustained input" window can suppress WASAPI media detection, misclassifying media as idle

**File:** `docs/ARCHITECTURE.md` §4, `docs/DECISIONS.md` #7, #34
**Problem:** The v1.1 spec introduces idle hysteresis: enter `idle` at 300 s no-input, **leave only after ~30 s sustained input** (Decision #34). The three-state model defines `passive_media` as "no recent input, but media is actively playing." Consider this scenario:
1. User is `idle` (no input for >300 s, no media).
2. User starts playing music on Spotify (WASAPI detects audio, GSMTC reports Playing).
3. User provides brief input (e.g., moves mouse once at t=5 s).

The state machine is now in a conflict: WASAPI/GSMTC says `passive_media`, but the sustained-input rule says "not yet `active`" (only 5 s of 30 s completed). The spec does not define which signal wins. If the implementation keeps the user in `idle` until the 30 s sustained-input window completes (even though media is playing), then a user who starts music and then provides brief input will be incorrectly classified as `idle` for up to 25 seconds — defeating the "media = not idle" invariant.

Worse, if the user then stops providing input but the music continues, the 300 s idle timer has already reset. The user could be `idle` for another 300 s (despite music playing) because the hysteresis window interrupted the normal media→idle transition path.

**Recommendation:** Define explicitly: **WASAPI/GSMTC media detection bypasses the sustained-input hysteresis window**. If media is detected playing, the state is `passive_media` regardless of the sustained-input timer. The sustained-input rule applies only to the `idle → active` transition (no media involved). Update `docs/ARCHITECTURE.md` §4 with a state-machine table that explicitly shows: `idle + media_detected → passive_media` (immediate, no hysteresis). Add a unit test for this exact scenario.

---

### C2 — `batches` receipt collection grows indefinitely with no TTL or retention policy

**File:** `docs/API-CONTRACT.md` §2, `docs/DECISIONS.md` #28, `docs/ARCHITECTURE.md` §7
**Problem:** Decision #28 specifies a `batches` receipt collection for idempotency, retained independently of `intervals`. At 100 agents syncing every 5 minutes, this collection grows by ~28,800 documents/day (~10.5M/year). Each document is small (~80 bytes), so storage is ~840 MB/year — manageable, but the collection is append-only and never cleaned up. Over 3-5 years, it becomes a performance drag on the unique-index maintenance and the initial dedup check at ingest. More critically, the "independently retained" rule means an admin who deletes a device's intervals (Decision #21) cannot also clean up its batch receipts without a separate action — orphaned receipts accumulate forever.

**Recommendation:** Add a `batches` TTL policy to `docs/ARCHITECTURE.md` §7: `receivedAt` TTL of 180 days (configurable via `settings`). A batch receipt older than 180 days is irrelevant — if an agent replays a 6-month-old batch, the intervals are either already stored (dedup works) or were admin-deleted (stale replay should be rejected, which the receipt handles). Document that deleting a device's intervals should also optionally clean its `batches` receipts (admin action). Add a MongoDB TTL index on `receivedAt` as a Phase 2 checklist item.

---

## High

### H1 — Sleep/resume + media detection edge case: post-resume `idle` interval may be wrong if media resumes immediately

**File:** `docs/ARCHITECTURE.md` §3, `docs/DECISIONS.md` #33, §4
**Problem:** Decision #33 specifies: "the open interval is closed on sleep/suspend (`PowerModeChanged`); a new `idle` interval starts on resume." But consider: a user is watching YouTube (`passive_media`), the laptop lid closes (sleeps), and 10 minutes later the lid opens (resumes). YouTube auto-resumes playback. The spec says a new `idle` interval starts on resume — but media is already playing within seconds of resume. The helper's GSMTC/WASAPI check runs on its next sample (3-5 s later), detects media, and transitions to `passive_media`. The result is a 3-5 second `idle` interval between the resume and the media detection, which is trivially short but creates noise in the data (a "blip" interval).

More importantly, if the helper's media detection is slow (GSMTC session re-registration takes >5 s after resume), the `idle` interval could be 10-30 seconds, which is material. The spec does not define a post-resume grace period for media detection.

**Recommendation:** After resume, **delay the interval start by one sample cycle** (~3-5 s) to allow media sessions to re-register before classifying the state. If media is detected on the first post-resume sample, start the interval as `passive_media` (no `idle` blip). If no media is detected, start as `idle`. This is a 5-line change in the resume handler. Document in `docs/ARCHITECTURE.md` §3.

### H2 — Single-node replica set transaction failure mode: crash between journal commit and agent ACK

**File:** `docs/DECISIONS.md` #29, `docs/ARCHITECTURE.md` §8
**Problem:** Decision #29 specifies atomic ingest via MongoDB transactions on a single-node replica set. The transaction is all-or-nothing (receipt + intervals commit together). But consider: the transaction commits (journaled), MongoDB crashes before sending the HTTP response to the agent. The agent times out, retries with the same `batchId`. The receipt exists (transaction committed), so the server returns `duplicate: true` with the original `accepted` count. The agent purges the rows. **No data loss** — this is correct.

However, if the MongoDB node crashes *between journal write and journal commit* (before `j: true` write concern is acknowledged), the transaction is lost. The agent retries, no receipt exists, the server inserts the intervals again. **No data loss** — but the `accepted` count in the receipt is now wrong (the receipt was not written, so a new one is created). This is a very narrow window and acceptable for v1, but it should be documented as a known limitation.

**Recommendation:** Document in `docs/DECISIONS.md` #29: "Transactions on a single-node replica set have a narrow window (between journal write and journal commit) where a crash can lose the transaction. The agent's retry logic handles this safely (no data loss, but a new receipt is created). For v1, this is acceptable. If higher durability is needed, add a second replica-set node." Also specify that the MongoDB connection string should use `w: "majority"` and `j: true` for the ingest operation. Add a Phase 2 checklist item: "verify transaction behavior on single-node replica set with `w: majority, j: true`."

### H3 — Org-timezone (Asia/Dhaka) day-bucketing is wrong for non-Dhaka/remote employees

**File:** `docs/DECISIONS.md` #40, `docs/ARCHITECTURE.md` §8, `docs/PLAN.md` Phase 4
**Problem:** Decision #40 specifies day/week bucketing uses a fixed org timezone (Asia/Dhaka, UTC+6). This works for 99% of staff. But consider: a remote employee in London (UTC+0) works 9 AM - 6 PM local (09:00-18:00 UTC). In Dhaka timezone, their workday is 15:00-00:00 (3 PM to midnight). A "daily summary" for Tuesday Dhaka time includes 3 hours of the London employee's Tuesday and 0 hours of their Wednesday — but the London employee's actual Tuesday work (09:00-18:00 UTC) falls entirely within Dhaka's Tuesday bucket (00:00-06:00 UTC = 06:00-12:00 Dhaka). Wait — that's actually fine. Let me recalculate.

Actually, let me reconsider: Dhaka is UTC+6. Dhaka midnight = 18:00 UTC previous day. So a Dhaka "Tuesday" bucket spans 18:00 UTC Monday to 18:00 UTC Tuesday. A London employee working 09:00-18:00 UTC Tuesday falls entirely within the Dhaka "Tuesday" bucket. **This works.**

But a London employee working 09:00-22:00 UTC Tuesday (late shift): the 18:00-22:00 portion spills into the Dhaka "Wednesday" bucket (18:00 UTC Tuesday = Dhaka Wednesday 00:00). The London employee's Tuesday is split across two Dhaka buckets.

**Recommendation:** The current design is correct for the 99% case and acceptable for v1. Add a note to `docs/DECISIONS.md` #40: "For employees in timezones far from Dhaka (e.g., UTC+0 or UTC-5), activity near the Dhaka midnight boundary (18:00 UTC) may be split across two day buckets. This is a known limitation for v1; per-employee timezone bucketing is deferred." No code change needed; just document the edge case.

### H4 — `machineHints.machineGuid` retrieval path is unspecified; many machines have no `ProductId` registry key

**File:** `docs/API-CONTRACT.md` §1, `docs/DECISIONS.md` #42
**Problem:** Decision #42 dropped `mac` and kept only `machineHints.machineGuid`. The `machineGuid` is typically read from `HKLM\SOFTWARE\Microsoft\Cryptography\MachineGuid`. However, this registry key may not exist on all Windows installations (e.g., some Windows Server editions, or machines where the key was manually deleted). If the agent cannot read the key, it must send `machineGuid: null` or omit the field. The contract does not specify the agent's behavior when the hint is unavailable.

**Recommendation:** Specify in `docs/API-CONTRACT.md` §1: `machineHints.machineGuid` is optional; the agent sends `null` if the registry key is unavailable. The server stores whatever is provided (including `null`) as informational data. Add a note to the agent conventions skill: "Read `MachineGuid` from `HKLM\SOFTWARE\Microsoft\Cryptography\MachineGuid`; if the key is missing or unreadable, send `null`."

### H5 — `dropped` block in heartbeat does not specify whether dropped rows were `idle`, `active`, or `passive_media`

**File:** `docs/API-CONTRACT.md` §3, `docs/DECISIONS.md` #29 (new)
**Problem:** The v1.1 heartbeat includes a `dropped` block (`{count, oldestUtc, newestUtc}`) when the buffer cap forces a drop. But the agent does not report *which* states were dropped. If the agent drops `idle` rows first (least valuable), the dashboard knows data was lost but not whether productive time was also lost. The agent's drop policy (idle first, then passive, then active) is mentioned in the plan but not in the contract.

**Recommendation:** Add an optional `droppedByState` field to the `dropped` block: `{idle: 5, passive_media: 2, active: 0}`. This lets the dashboard show "5 idle intervals dropped" vs "5 active intervals dropped" — very different signals. Document the agent's drop-preference order in the contract: idle → passive_media → active. Add to Phase 2 checklist.

---

## Medium

### M1 — Helper-as-logged-on-user cannot read DPAPI machine-scope secrets from LocalSystem service context

**File:** `docs/ARCHITECTURE.md` §2, §6, `docs/DECISIONS.md` #30, #32
**Problem:** Decision #32 specifies the helper runs as the logged-on user (not SYSTEM) for UIA compatibility. Decision #30 stores the buffer encryption key via DPAPI machine scope. The helper does not need secrets (it only observes and reports to the service). However, the spec does not explicitly state that the helper never reads from the buffer or the DPAPI-protected key. If an implementer gives the helper buffer-read access (for debugging or local diagnostics), the helper running as the logged-on user cannot decrypt the DPAPI-protected key (DPAPI machine scope is accessible only to SYSTEM and LocalSystem, not to the user's token).

**Recommendation:** Add to `docs/ARCHITECTURE.md` §2: "The helper **never** reads from the SQLite buffer or accesses DPAPI-protected secrets. All buffering, encryption, and networking live in the service. The helper only writes samples to the named pipe." This is already implied but should be explicit to prevent an implementer from giving the helper buffer access.

### M2 — `configSchemaVersion` ignore-unknown policy can silently drop new mandatory config fields

**File:** `docs/API-CONTRACT.md` §1, §3, `docs/ARCHITECTURE.md` §9
**Problem:** The v1.1 spec adds `configSchemaVersion` and says the agent "ignores unknown fields if this is newer" (forward-compatibility). But consider: v1.1 of the server sends a new mandatory field `mediaDetectionMode: "gsmtc+wasapi"` (changing the media detection behavior). A v1.0 agent ignores this field and uses its hardcoded default (`gsmtc` only). The server expects `gsmtc+wasapi` behavior; the agent delivers `gsmtc` only. The dashboard shows fewer `passive_media` intervals for v1.0 agents — a silent behavioral mismatch.

**Recommendation:** Distinguish between "informational" config fields (agent can ignore) and "behavioral" fields (agent must understand or reject). Add a `configCompatibility` array to the config block listing field names the agent must understand. If the agent sees an unknown field in `configCompatibility`, it should log a warning and use the server's default (not its own hardcoded default). Document in `docs/API-CONTRACT.md` §1.

### M3 — `batches` receipt collection unique index on `{deviceId, batchId}` may cause insert contention at 100 agents

**File:** `docs/API-CONTRACT.md` §2, `docs/ARCHITECTURE.md` §7
**Problem:** The unique index on `{deviceId, batchId}` in the `batches` collection is checked on every ingest. At 100 agents syncing every 5 minutes, that's ~0.33 index lookups/second — trivial. But the index is also written on every ingest (new receipt). On a single-node MongoDB, the unique-index write-lock is held briefly per insert. At 100 concurrent inserts (post-outage thundering herd), the write-lock contention could cause ingest latency spikes. The jitter (Decision #34) helps, but doesn't eliminate the window.

**Recommendation:** This is a low-risk concern at 100 agents. Document the expected behavior: "At 100 agents with jitter, the `batches` collection sees ~2-3 concurrent writes on average, with occasional spikes to ~10-15 during recovery. Single-node MongoDB handles this easily. If contention becomes an issue, consider a sharded collection or a separate receipt service (not needed for v1)." No code change needed.

### M4 — Sleep/resume: `PowerModeChanged` event may not fire on all Windows configurations (e.g., connected standby / Modern Standby)

**File:** `docs/DECISIONS.md` #33, `docs/PLAN.md` Phase 1
**Problem:** Decision #33 specifies closing the open interval on sleep/suspend via `PowerModeChanged`. On Modern Standby (connected standby) machines — increasingly common on laptops — the machine may enter a low-power state without firing `PowerModeChanged`. The helper continues running (CPU is technically on) but input and media may be suspended. The result: the helper samples the same state for minutes/hours, producing a single long interval that spans the standby period.

**Recommendation:** Add a standby-detection heuristic: if the time between two consecutive samples exceeds 2× the sample interval (e.g., >10 s for a 4-s interval), treat it as a standby gap and close the interval at the last sample time. Document in `docs/PLAN.md` Phase 1 as a known edge case. The heuristic is conservative (false positives are harmless — closing a 10-second interval is a no-op).

### M5 — `agentVersion` in heartbeat vs `minSupportedVersion` in manifest: version comparison semantics are unspecified

**File:** `docs/API-CONTRACT.md` §3, §4
**Problem:** The heartbeat sends `agentVersion: "0.1.0"`. The manifest specifies `minSupportedVersion: "0.1.0"`. The contract says agents below `minSupportedVersion` are flagged `outdated`. But the comparison semantics are not specified: is `0.1.0-beta.1` below `0.1.0`? Is `0.1.0.1234` (four-part version) below `0.1.0`? Semver comparison is well-defined, but .NET assembly versions can be four-part. If the agent uses `AssemblyInformationalVersion` (three-part + prerelease tag), the comparison is straightforward. If it uses `AssemblyVersion` (four-part), the server needs to handle four-part versions.

**Recommendation:** Specify: `agentVersion` is a three-part semver string (e.g., `0.1.0` or `0.1.0-beta.1`). The agent uses `AssemblyInformationalVersion` (not `AssemblyVersion`). The server compares using semver rules (prerelease < release, patch-level comparison). Document in `docs/API-CONTRACT.md` §3.

### M6 — Phase 0 seed script has no spec for synthetic interval shapes

**File:** `docs/PLAN.md` Phase 0
**Problem:** The Phase 0 checklist includes a "dev seed script" with "3 sample devices, one with 24 h of synthetic intervals." But the spec does not define what the intervals should look like: how many intervals per day? What states/domains? Should there be edge cases (e.g., an interval spanning midnight, a very short interval, a very long media interval)? Without a spec, each implementer generates different test data, and the dashboard is built against unrepresentative shapes.

**Recommendation:** Add a seed-data spec to `docs/PLAN.md` Phase 0: "Seed script generates 1 device with 24 h of intervals: ~100 intervals across all 3 states, 5+ domains (including `null`), 3+ browsers, at least one interval spanning the Dhaka midnight boundary (18:00 UTC), at least one interval >4 h (media binge), at least one interval <10 s (edge case). Use a fixed seed for reproducibility."

### M7 — `configSchemaVersion` is returned by the server but the agent never sends its own schema version

**File:** `docs/API-CONTRACT.md` §1, §3
**Problem:** The enroll/heartbeat response includes `configSchemaVersion: 1`. The agent uses this to decide whether to ignore unknown fields. But the agent never sends its own `configSchemaVersion` to the server (the heartbeat sends `agentVersion` but not `configSchemaVersion`). The server cannot know whether the agent understands a given config version — it can only infer from `agentVersion`, which is fragile.

**Recommendation:** Add `configSchemaVersion` to the heartbeat request body (the agent sends its own supported config schema version). The server can then detect mismatches and respond with a compatible config block or a warning. Add to `docs/API-CONTRACT.md` §3.

---

## Low

### L1 — `settings.collection` schema: `orgTimeZone` has no validation; admin can set an invalid timezone

**File:** `docs/ARCHITECTURE.md` §7, `docs/DECISIONS.md` #40
**Problem:** The `settings` collection includes `orgTimeZone: "Asia/Dhaka"`. An admin could set this to `"Invalid/Timezone"` or `"UTC+6"` (non-IANA). The server-side aggregation pipeline using `$dateToString` with an invalid timezone will throw, breaking all dashboard queries.

**Recommendation:** Validate `orgTimeZone` against the IANA timezone database on the server before saving. Reject invalid values with a `400` response. Document in `docs/ARCHITECTURE.md` §7.

### L2 — `settings.collection` has `retention.intervalDays: null` (indefinite) but no admin-facing way to set a finite retention

**File:** `docs/ARCHITECTURE.md` §7
**Problem:** The `settings` schema includes `retention: { intervalDays: null, screenshotDays: 7 }`. `null` means indefinite. But the admin dashboard UI (Phase 4) needs a way to change this to a finite value (e.g., 365 days). The schema supports it, but the dashboard spec does not mention a retention-settings UI.

**Recommendation:** Add to Phase 4 checklist: "admin can set `intervalDays` retention via dashboard settings page; server enforces via TTL index on `intervals.receivedAt`."

### L3 — `heartbeatIntervalSec` config range [60, 3600] allows 1-hour heartbeats; offline threshold becomes 3 hours

**File:** `docs/API-CONTRACT.md` §1
**Problem:** The valid range for `heartbeatIntervalSec` is [60, 3600]. At 3600 s (1 hour), the offline threshold is 3 hours. A device that crashes at t=0 is shown as "online" for 3 hours. This is a long blind spot for a monitoring system. The range is technically valid but operationally risky.

**Recommendation:** Add a warning in the dashboard when `heartbeatIntervalSec > 600`: "Long heartbeat intervals increase the time to detect offline devices." No contract change needed; this is a UI concern.

### L4 — Agent `dropped` block specifies `oldestUtc` and `newestUtc` but not the `batchId` of the dropped rows

**File:** `docs/API-CONTRACT.md` §3
**Problem:** When the buffer cap forces drops, the agent reports `{count, oldestUtc, newestUtc}`. But the dropped rows may have been part of a batch that was partially synced (some rows synced, some dropped). The server has no way to know which specific intervals were lost. This is acceptable for v1 (the dashboard shows a gap in the date range), but limits future reconstruction.

**Recommendation:** No change needed for v1. Document as a known limitation: "Dropped-row reporting is aggregate (count + date range), not per-interval."

### L5 — `urlCaptureMethod: "title-fallback"` domain extraction from window title is fragile and browser-specific

**File:** `docs/API-CONTRACT.md` §2, `docs/DECISIONS.md` #35
**Problem:** Decision #35 specifies a fallback chain: UIA → window-title parse → `domain: null`. The window-title parse step is browser-specific: Chrome uses `"Page Title - Google Chrome"`, Edge uses `"Page Title - Microsoft Edge"`, Firefox uses `"Page Title — Firefox"` (em-dash). The domain is not always in the title (e.g., `"New Tab - Google Chrome"`). The agent needs a per-browser regex library for title parsing, which is maintenance-heavy.

**Recommendation:** Document the known title formats in the agent conventions skill (Chrome, Edge, Firefox, Brave, Opera). For the fallback, extract the domain from the title if possible (e.g., `"YouTube - Google Chrome"` → `youtube.com`); if not, set `domain: null` and `pageTitle` to the raw window title. Accept that the fallback is imperfect and document it as such.

---

## Nice-to-have

### N1 — `batches` receipt collection could carry a `deviceDeleted` flag for admin cleanup tracking

When an admin deletes a device's intervals, the `batches` receipts remain (by design). A `deviceDeleted: true` flag on the receipts would let a cleanup job identify orphaned receipts without a separate tracking mechanism. Low cost, moderate operational value.

### N2 — Dashboard could show a "config version mismatch" warning for agents with old `configSchemaVersion`

If the server sends config fields the agent doesn't understand (detected via `configSchemaVersion` in the heartbeat — see M7), the dashboard could show a "this agent may not support all config features" badge. Useful for fleet management during rolling updates.

### N3 — `GET /api/v1/health` endpoint could include MongoDB replica-set status

The health endpoint checks MongoDB connectivity. Adding `rs.status()` output (primary/secondary, lag) would let the uptime monitor also detect replica-set degradation. Low cost, useful for the single-node replica set operational model.

---

## Sequencing Assessment

The revised phase order is generally correct. Two observations:

1. **Phase 7 is still overloaded.** The redistribution helped (helper/session/sleep → Phase 1, buffer-cap/revocation → Phase 2, browser/DPI → Phase 3, AV → Phase 5), but Phase 7 still contains: multi-monitor, PWAs/Electron, 100-device soak test, token revocation end-to-end, and edge-case sweeps. Multi-monitor and PWAs/Electron are capture-correctness items that belong in Phase 3 (URL/app capture). The soak test belongs in Phase 6 (after CI/CD is built). Token revocation end-to-end belongs in Phase 2 (where revocation is implemented). This leaves Phase 7 as "sweep any remaining edge cases" — a short, well-defined phase.

2. **Phase 0 is well-scoped.** The SQLite skeleton, self-contained publish validation, dashboard auth, and seed script are all correctly in Phase 0. No items are missing or over-specified.

---

## Where the v1.1 Plan Is Already Correct

The following v1.1 changes are well-specified and need no further revision:

- **`batchId` as UUID v4** (Decision #28): eliminates collision risk entirely. The UUID is globally unique without needing `deviceId` in the key. The compound unique index `{deviceId, batchId}` is belt-and-suspenders but correct.
- **Atomic ingest via MongoDB transaction** (Decision #29): the transaction semantics (all-or-nothing, receipt + intervals) are correct. The single-node replica set is the right operational model for 100 agents.
- **Dashboard admin auth via Cloudflare Access + middleware** (Decision #31): the two-layer approach (Cloudflare at the edge, middleware in the app) is correct. Wired in Phase 0 is the right call.
- **Helper launch via `WTSQueryUserToken` + `CreateProcessAsUser`** (Decision #32): the mechanism is now fully specified, including `WTS_SESSION_CHANGE` handling and "console session only" for v1.
- **Idle hysteresis (300 s enter, 30 s sustained input to leave)** (Decision #34): the hysteresis values are reasonable and the sub-5 s interval merge prevents noise. The interaction with WASAPI (C1 above) needs clarification, but the hysteresis design itself is correct.
- **URL capture fallback chain + `urlCaptureMethod` field** (Decision #35): the three-tier fallback (UIA → title parse → null) with honest reporting is the right design.
- **`pageTitle` at 256 chars, `windowTitle` at 512** (Decision #36): the caps are reasonable and the privacy rationale is sound.
- **`employeeId` denormalized on intervals at ingest** (Decision #37): the hot-dashboard-query optimization is correct and the "snapshot, never backfill" rule is explicit.
- **Org key as rotatable array** (Decision #38): the rotation-with-transition-window design is correct.
- **Device `status` as explicit state machine** (Decision #39): the enum (`pending → online/offline/outdated/revoked/archived`) is well-defined and eliminates ad-hoc derived fields.
- **Drop `mac`, keep `machineGuid`** (Decision #42): the rationale (MAC is multi-NIC-ambiguous, `machineGuid` is a single clean hint) is correct.

---

*End of iteration 2 review. No existing documents were modified.*
