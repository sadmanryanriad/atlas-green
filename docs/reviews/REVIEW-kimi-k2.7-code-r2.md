# Atlas Green — Pre-Implementation Design Review (Round 2)

**Model:** kimi-k2.7-code (opencode-go/kimi-k2.7-code)  
**Date:** 2026-06-23

## Executive Summary

v1.1 correctly closes most iteration-1 gaps: UUID `batchId`s, atomic ingest, the independent `batches` receipt collection, dashboard auth, helper launch via `WTSQueryUserToken`, clock/sleep handling, URL fallback + eTLD+1 normalization, and idle hysteresis. The remaining risks are mostly *new* contradictions or edge cases exposed by those fixes: fixed Asia/Dhaka day-bucketing silently breaks for any non-Dhaka employee; the sleep/resume fix hard-codes `idle` on resume and can fight the 3-state media model; single-node replica-set transactions introduce operational brittleness; the `batches` collection has no retention or erasure story; and running the helper as the logged-on user opens a standard-user spoofing path on the named pipe. A few Phase-0 items are still under-specified (dev auth bypass, replica-set bootstrap, dev org-key handling). The redistributed Phase 7 is better, but token-revocation end-to-end belongs in Phase 2, not Phase 7.

---

## Critical

### C1 — Fixed Asia/Dhaka day-bucketing misattributes work for any non-Dhaka employee and collides with viewer-local timeline rendering
**Files:** `docs/DECISIONS.md` #40; `docs/ARCHITECTURE.md` §8 (Dashboard / Time)
**Problem:** Decision #40 buckets daily/weekly summaries by the org timezone (`Asia/Dhaka`) and rejects per-employee timezone bucketing for v1. But the architecture also says timelines render in the *viewer's* local zone. The result is confusing at best and wrong at worst: a remote employee in the UK or a night-shift worker will have their work split across two Dhaka-day buckets, and an admin viewing the timeline in another zone will see activity times that do not align with the summary day boundaries. Because the capture-time UTC offset is not stored anywhere, the data cannot be re-bucketed later without guessing.
**Recommendation:** Store the capture-time UTC offset (or a timezone identifier snapshot) on each interval or on the device record. Add an optional per-employee timezone field to the `employees`/`settings` model and use it for that employee's summaries while keeping the org timezone as the default. At minimum, label every summary view "Day boundaries: Asia/Dhaka" and document the limitation before launch.

### C2 — `PowerModeChanged` is unreliable for a Session-0 Windows Service, undermining the sleep/resume fix
**Files:** `docs/ARCHITECTURE.md` §3 (Data flow step 3); `docs/PLAN.md` Phase 1; `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §3
**Problem:** v1.1 moved sleep/resume handling to Phase 1 and says the agent closes the open interval on sleep via `PowerModeChanged`. In a console/dev process `Microsoft.Win32.SystemEvents.PowerModeChanged` works, but in a Session-0 Windows Service it may never fire because it relies on a hidden window and a message pump that services do not have. If the dev implementation uses `PowerModeChanged` and the production service path is not explicitly different, laptops/desktops will again log phantom multi-hour intervals.
**Recommendation:** The service must handle `SERVICE_CONTROL_POWEREVENT` through the SCM service control handler. Use `SystemEvents.PowerModeChanged` only when running as a console in dev. Have the helper (in the user session) also detect suspend/resume and send an explicit message to the service. Add a Phase-1 exit-gate test that puts the service-mode agent through a real sleep/resume cycle and verifies the pre-sleep interval is closed.

### C3 — Helper running as the logged-on user can spoof the named pipe, defeating the "standard-user" threat model
**Files:** `docs/ARCHITECTURE.md` §2.2; `docs/DECISIONS.md` #32; `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §2
**Problem:** v1.1 correctly runs the helper as the interactive user (for UIA) and restricts the helper→service pipe to LocalSystem + that user. Because the user is an allowed pipe client, a determined standard user can write their own process under their own token, connect to the pipe, and inject fabricated samples. The service currently has no way to distinguish the real helper from a fake client, so the "defend against standard users" claim is weaker than stated.
**Recommendation:** On every pipe connection, the service must verify the peer process identity: obtain the client PID via `GetNamedPipeClientProcessId`, query `QueryFullProcessImageName`, and confirm the image path is the installed helper executable and is signed by the agent code-signing certificate. Reject any other client. Document the exact pipe security descriptor and this validation step in `ARCHITECTURE.md` §2.2 and the agent skill.

### C4 — "Revoked device only heartbeats" contradicts the Bearer-token auth model
**Files:** `docs/API-CONTRACT.md` §5 (Auth failures); `docs/ARCHITECTURE.md` §6 (Security model)
**Problem:** The contract says that after a blocked/403 re-enroll, the agent "stops generating new intervals, keeps its buffer, and only heartbeats (`status` reflects `revoked`)." However, `/heartbeat` is listed under "everything else" and requires a valid `Authorization: Bearer <device token>`; a revoked token returns `401`. A revoked device therefore cannot heartbeat at all, so it disappears from the dashboard instead of appearing as `revoked`.
**Recommendation:** Define revoked-token behavior explicitly: the `/heartbeat` endpoint accepts a revoked token and returns `200` with `status: "revoked"` (and no update/config), while `/ingest` and `/enroll` continue to reject it with `401`/`403`. Alternatively, define a separate unauthenticated heartbeat ping for revoked devices. Update the contract and the device-status state machine accordingly.

---

## High

### H1 — Atomic ingest on a single-node replica set is operationally brittle and not sized for worst-case batches
**Files:** `docs/DECISIONS.md` #29; `docs/ARCHITECTURE.md` §7, §8; `docs/PLAN.md` Phase 0 / Phase 2
**Problem:** v1.1 mandates MongoDB transactions on a single-node replica set. While load at 100 agents is trivial, the operational path is not: `rs.initiate()` must be run on the VPS, the replica-set config must use a resolvable hostname, oplog sizing matters for restore, and a single-node replica set has no real failover. A batch near the 5,000-interval cap that also resolves categories and `employeeId` per interval could exceed a default transaction timeout, causing the whole batch to abort and the agent to retry indefinitely.
**Recommendation:** Cap the per-transaction batch to 1,000 intervals (well below the 5,000 wire cap), set `maxCommitTimeMS` to 30 s, add retry logic for `TransientTransactionError`, and document the exact `rs.initiate()` command, connection string, and oplog sizing in the Phase-0 console runbook. Test restore from a `mongodump` of a replica set, not just a standalone instance.

### H2 — The `batches` receipt collection has no retention policy and conflicts with erasure
**Files:** `docs/DECISIONS.md` #28; `docs/ARCHITECTURE.md` §7 (`batches` schema)
**Problem:** v1.1 keeps batch receipts forever so a stale offline replay cannot resurrect admin-deleted intervals. With ~100 devices syncing every 5 minutes, the collection grows by tens of thousands of small documents per day. There is no sizing estimate, no TTL, no archive plan, and no documented "forget this device" flow. Worse, if an admin deletes a device's intervals for legal erasure but leaves the receipts, the data is arguably still retained (it links the device to activity timestamps), and a replay could partially restore it.
**Recommendation:** Model the growth (e.g., ~2–5 GB/year at 100 devices) and add a `receivedAt` index. Define a single admin "forget device" action that (a) revokes the device token, (b) blocks the device, (c) deletes all `intervals`, and (d) deletes all `batches` receipts for that device. Outside that action, retain receipts for the same period as `intervals`. Document that a device cannot be fully erased without also blocking it.

### H3 — Sleep/resume fix hard-codes `idle` on resume, contradicting the 3-state model when media is playing
**Files:** `docs/ARCHITECTURE.md` §3; `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §3
**Problem:** v1.1 says "close the open interval on sleep/suspend; a new `idle` interval starts on resume." If the user was watching a video or listening to music before sleep and the player resumes playback on wake (common for Spotify, YouTube, media apps), the correct state is `passive_media`, not `idle`. Hardcoding `idle` under-counts media time and violates the state-machine rules.
**Recommendation:** Change the rule to: on resume, start a new interval whose initial state is determined by the first post-resume sample (`active` if recent input, `passive_media` if media is playing, otherwise `idle`). Never default the post-resume state to `idle`.

### H4 — Idle hysteresis and the gated WASAPI fallback are missing transition rules and a staleness bound
**Files:** `docs/DECISIONS.md` #7, #34; `docs/ARCHITECTURE.md` §4; `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §3
**Problem:** v1.1 adds hysteresis (enter `idle` at 300 s, leave only after ~30 s sustained input) and gates WASAPI audio by the most-recent GSMTC source. It does not specify: (a) when `passive_media` transitions back to `idle` after media stops, (b) how old the "most-recent GSMTC source" may be before it is ignored, or (c) the behavior when a call starts during media playback. An old non-communications GSMTC source could wrongly gate a later Teams call as `passive_media`, and the sub-5-second merge rule could hide rapid state flips at media boundaries.
**Recommendation:** Add explicit rules: `passive_media` → `idle` when media stops and there is no recent input; the gating GSMTC source must be within the last 60 seconds; a communications GSMTC source always overrides the fallback to `active`/`idle`. Add unit tests for "media stops," "call starts during media," and "WASAPI audio with stale GSMTC source."

### H5 — Phase 0 dashboard auth has no documented dev fallback
**Files:** `docs/PLAN.md` Phase 0; `docs/ARCHITECTURE.md` §6
**Problem:** v1.1 wisely adds Cloudflare Access + Next.js `middleware.ts` to Phase 0, but in local development there is no Cloudflare proxy. If the middleware blindly requires a Cloudflare Access JWT, developers will be locked out of the dashboard before they can build it.
**Recommendation:** Implement the middleware so that in production it validates the Cloudflare Access JWT (signature, issuer, expiry, audience) and in development it accepts a dev-only session cookie or `Authorization` header controlled by an environment flag (e.g., `DEV_AUTH_BYPASS=admin@example.com`). Document the dev flow and never allow the bypass in production.

### H6 — Single-node replica-set bootstrap is under-specified for Phase 0
**Files:** `docs/PLAN.md` Phase 0; `docs/ARCHITECTURE.md` §8
**Problem:** Phase 0 requires MongoDB "initialized as a single-node replica set (`rs.initiate()`)," but the plan does not say how to do this in dev, in CI, or on the production VPS. A common failure is that `rs.initiate()` configures the replica set with the machine's hostname, which the driver cannot resolve from `localhost`, causing transactions to fail with "node is not in primary or recovering state."
**Recommendation:** Provide a deterministic bootstrap script (e.g., `scripts/mongo-init-replica-set.js`) and a `docker-compose.yml` that starts MongoDB with `--replSet rs0` and runs `rs.initiate()` with `127.0.0.1`. Use a connection string like `mongodb://127.0.0.1:27017/atlas?replicaSet=rs0` everywhere. Add this to the Phase-0 checklist.

### H7 — `/enroll` rate limit of 1/hour/IP can block the initial fleet rollout behind a NAT
**Files:** `docs/API-CONTRACT.md` §5
**Problem:** The contract limits `/enroll` to one per hour per IP. In a Bangladesh office where all ~100 PCs share one public IP, IT may image and install several machines in an hour; after the first enrollment, the rest receive `429` and cannot enroll until the bucket resets.
**Recommendation:** Replace the hard 1/hour/IP limit with a token bucket that allows a burst (e.g., 10 enrollments per hour per IP, refilling at 1/hour). Alternatively, add a corp-network IP allowlist to `settings.rateLimits` that bypasses the per-IP limit.

### H8 — `CreateProcessAsUser` parameters for the helper are not specified
**Files:** `docs/ARCHITECTURE.md` §2.1; `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §2
**Problem:** v1.1 names `WTSQueryUserToken` + `CreateProcessAsUser` but does not specify the desktop (`Winsta0\Default`), creation flags (`CREATE_NO_WINDOW`, `CREATE_UNICODE_ENVIRONMENT`), or environment block. If the helper is created without the correct desktop, it will not be able to read the interactive desktop via UIA even though it runs as the user, producing a "helper running but no samples" failure that looks like the helper is healthy.
**Recommendation:** Document the exact call: `lpDesktop = "Winsta0\\Default"`, `dwCreationFlags = CREATE_NO_WINDOW | CREATE_UNICODE_ENVIRONMENT`, no console, and a minimal environment block. Verify in Phase 1 that the helper can read foreground windows from a non-admin account.

---

## Medium

### M1 — `batches` receipt schema is confused about identity
**Files:** `docs/ARCHITECTURE.md` §7 (`batches` schema)
**Problem:** The schema says `_id: batchId` and also "unique on `{deviceId, batchId}`". Because `_id` is already globally unique, the compound unique index is redundant for deduplication and suggests two competing identity concepts. It also complicates the choice of `_id` generation.
**Recommendation:** Pick one: (a) keep `_id` as an auto-generated `ObjectId` and enforce the `{deviceId, batchId}` unique index, or (b) set `_id = batchId` and drop the compound unique in favor of a `{deviceId, receivedAt}` index for purge queries. Update the contract and skill to match.

### M2 — Daily-summary API can return dates that the client misinterprets as viewer-local
**Files:** `docs/ARCHITECTURE.md` §8
**Problem:** Summaries are bucketed by Asia/Dhaka, but the dashboard timeline is rendered in the viewer's local zone. If the client-side timezone conversion is applied to summary dates as well as timeline timestamps, an admin in another zone will see Dhaka-day buckets shifted into their local date, creating a mismatch between the summary and the timeline.
**Recommendation:** Return summary bucket identifiers as ISO date strings in the org timezone (e.g., `bucketDate: "2026-06-22"`) and add a `bucketTimeZone: "Asia/Dhaka"` field. Do not client-side convert summary dates; only convert timeline timestamps.

### M3 — Stored clock-skew offset is a single point-in-time value, not a history
**Files:** `docs/API-CONTRACT.md` §5; `docs/ARCHITECTURE.md` §7 (`devices.clockOffsetMs`)
**Problem:** v1.1 stores one `clockOffsetMs` on the device and says the dashboard can apply it at display time. If the agent's clock is corrected by NTP or an admin after being skewed for weeks, applying the latest offset to all historical intervals will misplace older data.
**Recommendation:** Either store a small offset history (timestamped samples) or limit the display offset to recent data and simply flag skewed devices. Better yet, record the offset at the time each batch is captured so intervals carry their own offset snapshot.

### M4 — Pending-update crash recovery is not specified
**Files:** `docs/ARCHITECTURE.md` §9
**Problem:** The update flow writes a `pending-update` marker and performs the file swap in `OnStop`. If the service process crashes after writing the marker but before `OnStop` runs, the next SCM restart will launch the old binary and leave the update stranded.
**Recommendation:** On every service startup, check for a `pending-update` marker. If it exists and the staged package passes signature + SHA-256 verification, complete the rename before entering the main loop. If verification fails, clear the marker and report `updateStatus: "failed"` on the next heartbeat.

### M5 — Phase 0 has no story for the dev org key or agent secrets
**Files:** `docs/PLAN.md` Phase 0; `docs/API-CONTRACT.md` §1
**Problem:** Phase 0 requires the agent to call `/enroll` with an org key, but the org key is described as embedded in the MSI. There is no plan for how a developer running `dotnet run` provides the key, or how the console seeds the first dev org key.
**Recommendation:** Add a dev-secrets step: the console repo includes a seed script that creates one dev org key, and the agent reads the org key from an environment variable or a local dev config file (never committed). Document this in both READMEs.

### M6 — Token-revocation end-to-end test is misplaced in Phase 7
**Files:** `docs/PLAN.md` Phase 2, Phase 7
**Problem:** Phase 2 already implements org-key arrays, blocked devices, and revocation. Phase 7 still lists "End-to-end token revocation → re-enroll-blocked (`403`) → revoked state on a real agent." Moving this to Phase 7 means the core security feature is not verified until the final phase.
**Recommendation:** Move the end-to-end revocation test to the Phase-2 exit gate: revoke a real device, confirm `/enroll` returns `403`, confirm `/ingest`/`/heartbeat` return `401` (or the revoked heartbeat behavior defined in C4), and confirm the dashboard shows `status: "revoked"`.

---

## Low

### L1 — No contract-level test harness exists to detect agent↔console drift
**Files:** `docs/API-CONTRACT.md` (overall)
**Problem:** The contract is authoritative, but there is no shared test fixture that exercises both sides. As different AI agents work on `atlas-agent` and `atlas-console`, subtle drift (e.g., a missing field, a changed status enum) will only surface at integration time.
**Recommendation:** Add a small contract test suite in the mother repo (e.g., a Postman collection or `curl` + JSON-schema checks) that spins up the console and validates `/enroll`, `/ingest`, and `/heartbeat` shape and status codes. Run it in CI before any release.

### L2 — No local IT diagnostics endpoint for the agent
**Files:** `docs/ARCHITECTURE.md` §2; `atlas-agent/AGENTS.md`
**Problem:** When the helper is dead or the buffer is not syncing, IT currently has to read log files on the machine. A localhost-only HTTP endpoint would let IT diagnose state without exposing the agent to the network.
**Recommendation:** Consider a localhost-only diagnostic endpoint (e.g., `http://127.0.0.1:<random-port>/diag`) that returns the last sample, helper state, buffer row count, and last sync attempt. Bind it to `127.0.0.1` only and require no secrets.

---

## Nice-to-have

### N1 — Add `changelogUrl` to the manifest schema
The manifest already supports signatures and version metadata. A `changelogUrl` lets IT review what changed before approving a fleet update.

### N2 — Dark mode for the dashboard
Minor quality-of-life improvement for admins reviewing timelines for long periods.

---

## Where v1.1 is already correct

These fixes are sound and should not be relitigated:

- **UUID `batchId` + `{deviceId, batchId}` dedup + atomic transaction.** Removes the collision and partial-insert risks found in iteration 1.
- **Independent `batches` receipt collection.** Correctly prevents stale offline replay from resurrecting admin-deleted data (though retention needs definition — see H2).
- **Cloudflare Access + Next.js `middleware.ts` for dashboard auth.** The right minimum viable auth for Phase 0.
- **Helper launch via `WTSQueryUserToken`/`CreateProcessAsUser`, console-session-only, `WTS_SESSION_CHANGE`.** The correct Windows interop pattern; just needs the desktop/flags details in H8.
- **App-layer AES-GCM + DPAPI machine scope + ACL.** The right choice for a self-contained, native-dependency-free buffer.
- **URL fallback chain, eTLD+1 normalization, and `urlCaptureMethod`.** Honest handling of UIA fragility.
- **Idle hysteresis and sub-5-second merge.** Solves the oscillation problem that would have defeated Model B.
- **`employeeId` denormalized at ingest.** Unblocks the hottest dashboard query path.
- **Explicit device `status` state machine.** Replaces the ad-hoc composition of `lastSeenAt`/`archived`/`revokedAt`.
- **Config schema version and clamped ranges.** Prevents a bad remote config from corrupting the fleet.
- **Self-contained publish smoke test moved to Phase 0.** De-risks the packaging path early.
- **Signed manifest + package, HTTP Range/resume, downgrade support.** Correct update security and resilience.

---

*End of review. Only this file was created; no other documents were modified.*
