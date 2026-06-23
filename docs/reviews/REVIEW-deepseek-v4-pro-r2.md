# Atlas Green — Iteration 2 Design Review

**Model:** deepseek-v4-pro (r2)
**Date:** 2026-06-23
**Scope:** Stress-test the v1.1 revisions (API-CONTRACT.md v1.1, DECISIONS rows #28–#42, redistributed Phase 7) against the six iteration-1 review reports. Find what they missed; do not repeat.

## Executive Summary

The v1.1 plan closes every material gap the iteration-1 reviews raised. The atomic-ingest transaction model, UUID `batchId` + `batches` receipt collection, dashboard auth via Cloudflare Access + middleware, helper-launch mechanism via `WTSQueryUserToken`, idle hysteresis, timezone-aware day-bucketing, and the redistributed Phase 7 are all correct and well-reasoned. Six new issues survive: (1) the console AGENTS.md still documents a v1.0 manifest route the v1.1 contract deleted — a mother→child documentation drift that would cause Phase 0 to build dead code; (2) there is no recovery procedure for a MongoDB single-node replica set that loses its replica-set configuration (the daily `mongodump` backs up data, not the `local` database); (3) the named-pipe helper seq resets on helper restart, risking false dedup of the first post-restart sample; (4) a Next.js function timeout racing against a MongoDB transaction timeout can produce a window where the agent retries while the original transaction still holds the `batches` index lock; (5) changing `orgTimeZone` in settings retroactively alters all historical day-bucketed summaries with zero audit trail; (6) the heartbeat carries no `reauthorized` signal, so an admin who un-revokes a device has no push-path to tell the agent to resume capture — the agent stays in its self-imposed "revoked" state heartbeat loop until manually restarted.

---

## Critical

### C1 — Console AGENTS.md documents `/api/v1/update/manifest` route that the v1.1 contract deleted; Phase 0 will build dead code
**Severity:** Critical
**File:** `atlas-console/AGENTS.md:30` vs `docs/API-CONTRACT.md:235`

**Problem:** The API contract v1.1 explicitly removed `GET /api/v1/update/manifest` (see API-CONTRACT.md §4: "There is no separate console `/update/manifest` route — it was removed in v1.1 to avoid two sources of truth"). The heartbeat `update` block is now the sole channel. But the console's local context file (`atlas-console/AGENTS.md:30`) still lists it verbatim: "API routes the agents call: `POST /api/v1/enroll`, `POST /api/v1/ingest`, `POST /api/v1/heartbeat`, `GET /api/v1/update/manifest`." An AI agent implementing Phase 0 from the console AGENTS.md will build this endpoint — creating a second source of truth that the contract explicitly removed to prevent. The console skill (`atlas-api-contract/SKILL.md:24`) correctly notes "No `/update/manifest` route," but the AGENTS.md was not updated when the contract changed. This is a mother→child documentation drift: the mother repo's authoritative contract says one thing, the child repo's context file says another.

**Recommendation:** Replace line 30 of `atlas-console/AGENTS.md` to remove the manifest route reference. The line should read: `POST /api/v1/enroll`, `POST /api/v1/ingest`, `POST /api/v1/heartbeat`, `GET /api/v1/health`. As a process rule: when the API contract is bumped, a checklist item must verify both child AGENTS.md files are consistent with the new version — the plan currently has no such step.

---

## High

### H1 — No recovery procedure for MongoDB single-node replica set that loses its replica-set configuration
**Severity:** High
**File:** `docs/PLAN.md` Phase 0/6, `docs/ARCHITECTURE.md §7–8`

**Problem:** Decision #29 requires MongoDB as a single-node replica set (so transactions work). The Phase 0 checklist includes `rs.initiate()` and Phase 6 includes "daily mongodump → R2 backup with a restore test." But `mongodump` backs up user databases — it does **not** back up the `local` database, which holds the oplog and the replica set configuration document (`local.system.replset`). If the VPS disk fails and only the user-data dump is available, restoring to a fresh MongoDB node requires re-`rs.initiate()` and the data is standalone — transactions don't work until the replica set is reconfigured, and the oplog is fresh (historical oplog entries that might be needed for a resumed transaction are gone). More practically: if the replica set config becomes corrupted (a rare but real MongoDB failure mode on single-node setups after an unclean shutdown), the node may refuse to start, and the documented recovery path is "restore from backup" — but the backup doesn't contain what's needed to restart the replica set.

**Recommendation:** Add to the Phase 6 backup runbook: (a) include a `mongodump --db local --collection system.replset` alongside the user-data dump, or (b) document the recovery procedure explicitly: "restore user data → `rs.initiate()` on the new node → transactions resume." Option (b) is simpler and sufficient — the key is that the restore test must validate the procedure end-to-end, not just the data export. Also add an oplog size cap (`--oplogSize 512` or similar) to the Phase 0 MongoDB setup instruction so the oplog doesn't grow unbounded on a small VPS disk.

### H2 — Named-pipe helper monotonic seq resets on helper restart; first post-restart sample can be falsely de-duped by the service
**Severity:** High
**File:** `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md:43` ("per-sample monotonic seq for dedupe")

**Problem:** The helper sends each sample with a monotonic seq number so the service can drop re-sent samples on pipe reconnect. When the helper restarts (crash, watchdog, user kills it), the seq counter resets to 0 (it's in-process memory). The service sees seq=0 and compares it against the last-seen seq from the previous helper instance — which could be e.g. seq=150. The service correctly rejects seq=0 as "behind" (a stale retransmission). But what happens on the **third** helper instance? The second instance ran for a while (say seq reached 50), then died. The third instance starts at seq=0. The service's last-seen seq is 50. It rejects seq=0,1,2,...49 as "duplicates" of the second instance, and only starts accepting at seq=50. This silently drops up to 50 samples (~3–4 minutes of activity) after every multiple-restart scenario. The seq is genuinely per-instance, but the service treats it as globally monotonic.

**Recommendation:** Prefix the seq with a per-helper-instance UUID (generated at helper startup and sent in a handshake message when the pipe connects). The service tracks `(instanceId, seq)` pairs and only dedupes within the same instance. The instance UUID is re-generated on every helper start, avoiding the inter-instance seq collision. Alternatively: use a combined `(helperStartUtc, seq)` key. This is 10 lines in the pipe handshake and removes the data-loss window.

### H3 — MongoDB transaction timeout (default 60s) can race against Next.js API route timeout; agent retry blocks on in-flight transaction
**Severity:** High
**File:** `docs/API-CONTRACT.md §2`, `docs/ARCHITECTURE.md §3`, `atlas-console/.agents/skills/atlas-api-contract/SKILL.md:46–50`

**Problem:** The ingest API route starts a MongoDB transaction (write `batches` receipt + insert intervals → commit). MongoDB transactions have a default 60-second timeout. Next.js API routes on a VPS also have a timeout (typically 30–60s depending on configuration). Scenario: a slow disk or a large batch (approaching the 5000-interval cap) causes the transaction to take 45s. The Next.js function times out at 30s. The agent sees a 504/timeout, keeps the rows, retries immediately with the same `batchId`. The original MongoDB transaction is STILL RUNNING — it holds a write lock on the `batches` unique index entry for `{deviceId, batchId}`. The retry's transaction tries to insert the same `batches` doc, and blocks waiting for the first transaction to commit or abort. If the first transaction commits successfully (the server just didn't respond in time), the retry gets a duplicate-key violation, the transaction aborts, the agent gets `duplicate: true`, and purges — correct, but the retry was blocked for potentially 30+ seconds. If the first transaction eventually aborts (timed out), the retry inserts normally — also correct. The data integrity is preserved by the unique index, but the **latency** is worse than expected: the agent's HTTP client might time out its retry too, cascading the delay. At 100 agents, a single slow batch doesn't matter, but if a fleet-wide issue causes all agents to sync large backlogs simultaneously (e.g., post-outage recovery), 100 concurrent transactions with lock contention on the `batches` collection could create a brief queue.

**Recommendation:** The data-integrity story is actually correct thanks to the `{deviceId, batchId}` unique index — the worst case is latency, not data loss. Document the transaction-timeout interaction explicitly in the console skill: "Set the MongoDB transaction timeout (`transactionLifetimeLimitSeconds`) to ≤ the Next.js function timeout minus 5s, so transactions abort cleanly before the HTTP response times out." For the agent: its HTTP client should use a timeout slightly longer than the sync interval (e.g., 4 min 30s) to avoid timing out before the server finishes, and the retry backoff should not start at 0. This is an implementation detail the skills should mention.

### H4 — `configSchemaVersion` was added in v1.1 but the agent's behavior on an unsupported version is underspecified
**Severity:** High
**File:** `docs/API-CONTRACT.md §1, §3`, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md:94`

**Problem:** v1.1 added `configSchemaVersion` to both `/enroll` and `/heartbeat` responses (addressing REVIEW-minimax M10). The agent skill §4 says "Validate server config: clamp/ignore out-of-range values (see API-CONTRACT §1 ranges); keep the previous value and log a warning." This covers *value-range* validation but not *schema-version* handling. If the server returns `configSchemaVersion: 3` and the agent only knows up to version 1, the agent should: (a) log a prominent warning that it received an unsupported config schema, (b) NOT apply config fields it doesn't understand (forward-compat by ignore-unknown, which the skill already implies), and (c) **report this version mismatch back to the server** via the heartbeat so the dashboard can flag the device as needing an update. Currently, the agent silently ignores unknown config fields, and the server has no way to know the agent didn't apply a critical new setting. A server admin tweaking a v2-only config knob would see "config applied" with no feedback that 100 agents ignored it.

**Recommendation:** Add to the agent skill §4: "If `configSchemaVersion > agentSupportedSchemaVersion`, log a warning and report the server's version back in the heartbeat (new field: `agentConfigSchemaVersion` vs `serverConfigSchemaVersion`). The server can then flag the device as `config_mismatch` and surface it in the dashboard heartbeat panel." The heartbeat already sends `configSchemaVersion` (the agent's supported version) — the server just needs to compare it against what it sent and act on the mismatch. This is a 5-line server-side check.

### H5 — Changing `orgTimeZone` in settings retroactively alters all historical day-bucketed summaries with no audit trail
**Severity:** High
**File:** `docs/ARCHITECTURE.md §7` (`settings.orgTimeZone`), `docs/API-CONTRACT.md`, `atlas-console/.agents/skills/atlas-api-contract/SKILL.md:88–91`

**Problem:** Decision #40 locks day/week bucketing to a fixed org timezone (Asia/Dhaka). The `orgTimeZone` lives in the `settings` collection as a single global value. If an admin changes it (e.g., the company opens a remote office and shifts to a different zone, or the admin makes a mistake), EVERY historical daily summary report changes retroactively. An employee's "Monday" activity report that previously showed 8 hours now shows a different split because the day boundary moved. There is no audit log for `settings` changes (the `auditLog` collection is scoped to "destructive/rules-changing admin actions" — orgTimeZone arguably qualifies but isn't explicitly listed), and there's no warning in the dashboard that "timezone was changed on date X; reports before this date use the old zone." The change is silent and irreversible (you can change it back, but there's no record of when or who changed it).

**Recommendation:** (a) Add `settings` changes (specifically `orgTimeZone` and rate limits) to the `auditLog` collection scope. (b) Store a `timezoneChangedAtUtc` timestamp on the settings doc. The dashboard shows "Day bucketing uses Asia/Dhaka (changed 2026-08-15 by admin@company.com)" in the report footer. (c) Bonus: the dashboard could offer a toggle "show using timezone as of [date]" for historical consistency, but this is v2. The minimum is the audit log entry so the change is not invisible.

---

## Medium

### M1 — Phase 0 "unit-test scaffolding wired and green" is vacuous with zero tests
**Severity:** Medium
**File:** `docs/PLAN.md` Phase 0

**Problem:** The Phase 0 checklist item reads: "Unit-test scaffolding wired (`dotnet test` / `vitest`) and green." A test runner with no tests passes with "0 tests run, 0 passed, 0 failed" — trivially green. This doesn't validate that the test framework, assertion library, CI runner, or any actual test infrastructure works. The first real test written in Phase 1 could discover that xUnit isn't properly configured, `dotnet test` can't find the test project, or vitest has a config error — and that failure is attributed to Phase 1 work, not Phase 0 setup.

**Recommendation:** Change the checklist item to: "One smoke-test unit test per repo (e.g., `Assert.True(true)` in xUnit; `expect(true).toBe(true)` in vitest) — both run and pass in `dotnet test` / `npm test`." A single trivial test that exercises the full `[Fact]` / `it()` lifecycle validates the toolchain without coupling to any feature logic.

### M2 — The `dropped` heartbeat block has no lifecycle; the dashboard doesn't know when a drop condition was resolved
**Severity:** Medium
**File:** `docs/API-CONTRACT.md §3` (heartbeat `dropped`), `docs/ARCHITECTURE.md §7`

**Problem:** v1.1 added the `dropped: { count, oldestUtc, newestUtc }` block to the heartbeat (addressing REVIEW-deepseek M8, REVIEW-mimo M9). The agent reports it when the buffer cap forced a drop. But there is no specification for what happens when the condition is resolved: the device reconnects, syncs, clears the buffer, and the next heartbeat sends `dropped.count: 0`. Should the dashboard clear the "data gap" warning? Should it persist the last-known drop event for historical reference? Neither the contract nor the architecture defines the server-side lifecycle of this field. An implementor might either: (a) persist a `lastDropped` event on the device doc and never clear it (cluttering the dashboard permanently), or (b) clear the warning each heartbeat (so a transient drop is visible for only 5 minutes — easy to miss). Neither is wrong, but the inconsistency across two implementors will produce a confusing dashboard.

**Recommendation:** Specify in `docs/ARCHITECTURE.md §7` (devices collection): add an optional `lastDropped: { count, oldestUtc, newestUtc, reportedAtUtc }` field to the device doc. The server sets it when `dropped.count > 0` and never clears it (it's a historical event). The heartbeat panel shows the most recent drop event with its date range. Add a separate `currentBufferPressure` indicator (from `queuedRows` vs `bufferCapDays`) for the "live" status. The `dropped` block in the heartbeat is the trigger; the server persists the event.

### M3 — No server behavior specified for a heartbeat from a deviceId not in the `devices` collection
**Severity:** Medium
**File:** `docs/API-CONTRACT.md §3`, `atlas-console/.agents/skills/atlas-api-contract/SKILL.md:21–29`

**Problem:** The heartbeat endpoint updates `lastSeenAt`/`status` on the device document. But the contract doesn't specify what happens if the token is valid (passes token-hash lookup) but the `deviceId` doesn't exist in the `devices` collection — an impossible state in normal operation (enrollment creates the device doc before issuing the token), but reachable if an admin manually deleted the device document without revoking its token. The token→device binding check (skill §1) rejects a token whose bound `deviceId` doesn't match the body — but the token exists and IS bound to that `deviceId`. The device doc is simply gone. Current behavior is undefined: does the server return `404`? `500`? Silently create an orphaned device doc with partial fields? Each choice creates a different edge case for the dashboard.

**Recommendation:** The server should return `404` with a clear error message. The agent on `404` to heartbeat should attempt a single re-enroll (the device doc was deleted, so the server will treat it as a new enrollment — the token was valid so the device isn't blocked). If re-enroll succeeds, the device gets a fresh device doc and continues. If re-enroll returns `403` (blocked), the agent enters the revoked state (skill §4). Document this in the contract §3 Behavior and the console skill.

### M4 — `settings.orgKeys` array grows unbounded with no cleanup policy for expired+revoked keys
**Severity:** Medium
**File:** `docs/ARCHITECTURE.md §7` (`settings`), `docs/DECISIONS.md #38`

**Problem:** Decision #38 specifies org keys as "an array in `settings` with created/expiry/revoke per key" to enable rotation with a transition window. When keys expire or are revoked, they remain in the array indefinitely. At typical rotation cadences (e.g., one new key per quarter), this is ~4 entries/year — trivial. But if keys are rotated during a security incident (multiple revocations in quick succession), or if the array is never pruned, old key hashes sit in the settings doc forever. More importantly: an implementor writing the `/enroll` validation loop iterates the entire array each time to find a matching key hash. With 50 entries, this is fine. With 500 entries after years of aggressive rotation, it's still fine (~500 SHA-256 comparisons per enroll). The real risk is that an expired-but-not-revoked key's hash is still in the array, and if that hash was ever leaked, an attacker could enroll using an expired key — the contract says "any currently-valid key is accepted," and "valid" is determined by `expiresAt > now` and `!revokedAt`. This is correct as specified. But without a cleanup policy, the array is a growing list of sensitive hashes with no reason to keep the expired+revoked ones.

**Recommendation:** Add to the `settings` schema: expired+revoked keys are retained for 90 days after revocation (for audit), then pruned by a periodic server-side cleanup job (or manually by an admin). The `/enroll` validation already checks `expiresAt` and `revokedAt` — pruning just reduces the iteration space. Document the retention policy in `docs/ARCHITECTURE.md §7`.

### M5 — `PowerModeChanged` / `WM_POWERBROADCAST` may not fire reliably on all Windows configurations and VMs
**Severity:** Medium
**File:** `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md:68–71`

**Problem:** Decision #33 and the agent skill §3 specify closing the open interval on `PowerModeChanged` (suspend). This is correct for physical hardware. But on virtual machines (common for testing, and some BD offices run Windows VMs on thin clients), the hypervisor's suspend/resume may not generate a `WM_POWERBROADCAST` message to guest applications. On some Windows configurations with fast-startup enabled, a "shutdown" is actually a hybrid suspend — the `PowerModeChanged` event fires, but the subsequent "resume" on the next boot may not fire a corresponding event because the kernel session was restored from hibernation. In both cases, the open interval is never closed, and `durationSec` inflates to cover the suspended period. The monotonic clock (`Stopwatch`) should catch this because its elapsed time would be enormous — but the agent skill doesn't specify a sanity-check fallback for when the power event is missed.

**Recommendation:** Add a safety net to the aggregator: if the open interval's monotonic duration exceeds `idleThresholdSec × 3` (e.g., 900s = 15 min) without any state change, force-close the interval (with `state: idle` for the excess period) and log a warning. This catches missed power events, VM suspend/resume, and aggregator bugs. The power event is still the primary mechanism (fast and precise), but the duration-sanity fallback prevents the worst-case phantom-interval scenario cross-platform.

### M6 — Code-signing cert + GPO trust process is listed in both Phase 5 and Phase 6 checklists as distinct items
**Severity:** Medium
**File:** `docs/PLAN.md` Phase 5, Phase 6

**Problem:** Phase 5 checklist: "Self-signed code-signing cert + GPO trust process documented." Phase 6 checklist: "Self-signed code-signing cert + GPO trust process documented." These are the same cert and the same GPO trust process. Phase 5 needs it for the MSI installer (Decision #18: "IT pushes the self-signed cert to LocalMachine\Root via Group Policy on every target PC BEFORE the MSI rollout"). Phase 6 needs it for the auto-update pipeline (the same cert signs the update packages and manifest). Having two separate checklist items invites two separate documents that drift apart, or worse, two different certs.

**Recommendation:** Consolidate: Phase 5 owns the cert generation and GPO trust documentation (it's the pre-rollout prerequisite). Phase 6's item should be "Build pipeline signs packages and manifest with the cert from Phase 5 (no new cert)." Reference the Phase 5 doc, don't recreate it. The cert is generated once; the GPO is pushed once; both phases use the same artifact.

### M7 — `batches` receipt collection has no retention discussion; grows ~15 MB/year — trivial but undocumented
**Severity:** Medium
**File:** `docs/ARCHITECTURE.md §7` (`batches` collection), `docs/DECISIONS.md #28`

**Problem:** The `batches` receipt collection receives one document per sync per device. At 100 agents × 1 sync/5 min = ~28,800 receipts/day = ~10.5M/year. At ~100 bytes/document (batchId UUID + deviceId + accepted + receivedAt + indexes), that's ~1 GB/year including indexes. This is trivial for MongoDB, but "forever" growth (Decision #28 says "retained independently of intervals") means ~10 GB after a decade. Still fine, but no TTL index, no retention policy, and no mention of it in the data model. A future admin doing capacity planning won't find this collection in the retention docs and may not know it exists.

**Recommendation:** Add a note to `docs/ARCHITECTURE.md §7` (batches collection): "At ~100 agents, the batches collection grows ~15 MB/year. No TTL is needed for v1; if fleet size grows 10×, add a TTL index on `receivedAt` (e.g., 2 years) — a receipt older than the agent's maximum offline buffer cap cannot be replayed." This is a documentation-only fix.

---

## Low

### L1 — After an NTP correction, `startUtc`, `endUtc`, and `durationSec` can be mutually inconsistent
**Severity:** Low
**File:** `docs/API-CONTRACT.md §2` (interval schema), `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md:67–68`

**Problem:** The agent measures `durationSec` from a monotonic clock (`Stopwatch`) and stamps `startUtc`/`endUtc` from the wall clock (Decision #33). If NTP corrects the wall clock by −2 seconds during an open interval, the computed `endUtc = startUtc + durationSec` will be internally consistent at the agent, but a dashboard query doing `endUtc - startUtc` will get a value 2 seconds less than `durationSec`. The server currently rejects `durationSec` outside `[1, 86400]` but doesn't cross-check it against `endUtc - startUtc`. This inconsistency is cosmetic (2 seconds across a 745-second interval = 0.3% error) and self-correcting (future intervals use the corrected clock). But the three-field redundancy exists precisely to catch these errors.

**Recommendation:** At ingest, the server should compute `(endUtc - startUtc)` and log a warning (not reject) if it differs from `durationSec` by more than 5 seconds. The dashboard should always use `durationSec` (monotonic) for time calculations and `startUtc`/`endUtc` (wall clock) only for date bucketing. Document this in the API contract §2 Behavior and the console skill.

### L2 — `capture.helperPid` or `helperStartedAtUtc` missing from heartbeat; kill-loop diagnosis is coarse
**Severity:** Low
**File:** `docs/API-CONTRACT.md §3` (heartbeat `capture` block)

**Problem:** The v1.1 heartbeat `capture` block carries `helperRunning`, `lastSampleUtc`, and `helperRestartCount`. These tell an admin "the helper restarted 15 times today" — but not whether those restarts were clustered (kill-loop in a 2-minute window) or spread over the day (normal logoff/logon). Without a `helperStartedAtUtc` or `helperPid`, the dashboard can't show a timeline of helper restarts. An admin investigating a tamper-suspected device needs to correlate restart timestamps with interval gaps.

**Recommendation:** Add an optional `helperStartedAtUtc` (ISO-8601) to the `capture` block. The agent sets it to the helper process start time. The server persists the most recent value on the device doc. The dashboard heartbeat panel can show "Helper running since 09:14 UTC (restarted 3 times today)." This is one field, backward-compatible (optional in the JSON), and materially improves the tamper-investigation UX.

### L3 — Phase 7 still has a "Sweep any edge cases found during earlier phases" catch-all with no triage policy
**Severity:** Low
**File:** `docs/PLAN.md` Phase 7

**Problem:** The v1.1 plan redistributed most Phase 7 items to earlier phases (addressing REVIEW-minimax L9). The remaining Phase 7 checklist includes "Sweep any edge cases found during earlier phases (expand as discovered)." This catch-all is open-ended — it could accumulate items indefinitely, turning Phase 7 back into the tar pit the redistribution was meant to prevent. There's no mechanism for deciding whether an edge case found in Phase 3 gets fixed in Phase 3, deferred to Phase 7, or tracked as a post-v1 issue.

**Recommendation:** Add a triage rule to the plan: "Edge cases discovered during a phase are either: (a) fixed in the current phase if ≤1 day of work, (b) added to a 'Known Limitations' section in the relevant README if acceptable for v1, or (c) tracked as a GitHub issue with a `phase-7` label if they genuinely need post-feature hardening." Phase 7's catch-all becomes "Resolve all `phase-7` GitHub issues filed during earlier phases." This caps Phase 7 to known, triaged work rather than an unbounded bucket.

### L4 — Revoked-then-reauthorized device has no push-path to resume capture; agent stays in self-imposed revoked state
**Severity:** Low
**File:** `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md:96–98`, `docs/API-CONTRACT.md §3, §5`

**Problem:** The agent skill §4 specifies: on `401` → re-enroll → if `/enroll` returns `403` (blocked) or re-enroll keeps failing → "stop generating intervals, keep the buffer, and only heartbeat (revoked state)." This is a terminal state from the agent's perspective. But what if an admin later un-revokes the device (clears the `blocked` flag, un-revokes the token)? The agent is still in its self-imposed revoked state, doing heartbeat-only. Nothing in the heartbeat response tells the agent "you've been reauthorized — resume capture and try re-enrolling." The only recovery path is: the admin restarts the agent service (via dashboard command or IT remote access), the agent re-reads its token, tries `/ingest`, gets `200` (token is valid again), and resumes. This works but requires manual intervention — a slightly awkward UX for what should be a self-healing fleet.

**Recommendation:** The server, on heartbeat from a device whose `status` is `revoked` but whose token is currently valid (admin cleared the block), includes a `reauthorized: true` flag in the heartbeat response. The agent, on seeing this flag, exits its revoked state: resumes interval generation, attempts re-enroll (to get a fresh config), and continues normal operation. The flag is optional and backward-compatible. This closes the push-path gap and makes the fleet self-healing after admin un-revoke.

---

## Where the v1.1 plan is correct and needs nothing

The following v1.1 changes were stress-tested and found correct:

- **UUID batchId + `batches` receipt collection (Decision #28).** The UUID removes all timestamp/sequence collision and restart-reuse risks the iter-1 reviews flagged. The independent `batches` collection correctly prevents stale replay from resurrecting admin-deleted data. No gap found.

- **Atomic ingest via MongoDB transaction on single-node replica set (Decision #29).** The data-integrity story is sound. H3 above notes a latency edge case, but zero data can be lost — the unique index on `{deviceId, batchId}` is the backstop regardless of transaction timing.

- **App-layer AES-GCM buffer encryption with DPAPI machine scope (Decision #30).** Correct for the self-contained single-file constraint and the standard-user threat model. The ACL is explicitly the primary control; DPAPI/encryption is defense-in-depth (addressing REVIEW-glm L1). No gap found.

- **Dashboard admin auth via Cloudflare Access + middleware (Decision #31).** Wired in Phase 0, with the middleware blocking direct-IP bypass. This closes the largest security gap the iter-1 reviews flagged (REVIEW-glm H3, REVIEW-mimo C3, REVIEW-minimax H7, REVIEW-qwen C3, REVIEW-kimi — implicit). No gap found.

- **Helper launch via WTSQueryUserToken + CreateProcessAsUser (Decision #32).** The mechanism, privilege requirements (LocalSystem), session-change handling, and console-only policy are all specified. REVIEW-glm M1, REVIEW-mimo M1, REVIEW-deepseek C3, REVIEW-kimi C3, and REVIEW-qwen C2 all flagged the missing helper-launch spec — v1.1 completely resolves this. No gap found.

- **Monotonic clock for durations, close interval on sleep/suspend (Decision #33).** The power-event hook, monotonic-vs-wall-clock split, and "never rewrite buffered timestamps" policy are correctly specified. M5 above adds a sanity-check fallback for missed power events, but the core design is solid. No architectural gap.

- **Idle hysteresis (Decision #34).** The 300s-in / 30s-out hysteresis and sub-5s merge rules correctly prevent state oscillation (addressing REVIEW-minimax H4). The interaction with the gated WASAPI media state is clean: media presence overrides idle entry, and hysteresis prevents brief input bursts from toggling between active and passive_media. No gap found.

- **URL fallback chain + eTLD+1 normalization (Decision #35).** UIA → title-fallback → `domain: null` with `urlCaptureMethod` is the correct three-tier approach (addressing REVIEW-deepseek H4, REVIEW-mimo H1, REVIEW-kimi C2, REVIEW-qwen C1). Domain normalization before buffering (addressing REVIEW-minimax H2) is correct. No gap found.

- **Page title cap at 256, window title at 512 (Decision #36).** Title truncation at the agent bounds the privacy surface (addressing REVIEW-minimax M2, REVIEW-glm L5). Correct. No gap found.

- **employeeId denormalized at ingest (Decision #37).** The snapshot-and-never-backfill rule + bulk re-resolve endpoint is the correct hot-query optimization (addressing REVIEW-minimax H8). No gap found.

- **Org key array with expiry/revoke (Decision #38).** The multi-key rotation with transition window and blocked-device 403 (addressing REVIEW-deepseek H6, REVIEW-glm M15, REVIEW-kimi H3/M6, REVIEW-qwen H5) is correct. M4 above notes cleanup of expired keys, but the core rotation protocol is sound.

- **Device `status` as explicit server-owned state machine (Decision #39).** The single `status` enum replacing ad-hoc derived fields (addressing REVIEW-minimax H5) is correct. No gap found.

- **Org-timezone day-bucketing with viewer-local timeline rendering (Decision #40).** The two-zone approach (Dhaka for day boundaries, viewer-local for timeline display) correctly handles the 99%-Bangladesh fleet while keeping the multi-timezone admin experience honest. The rare non-Dhaka employee gets Dhaka-day bucketed summaries — a known, documented limitation. H5 above notes the retroactive-change audit gap, but the design is correct.

- **Heartbeat capture health (Decision #41).** `helperRunning`, `lastSampleUtc`, `helperRestartCount` close the "service online ≠ helper capturing" gap (addressing REVIEW-glm M2, REVIEW-mimo — implicit). L2 above suggests adding `helperStartedAtUtc` for richer diagnosis, but the core capture-health signal is present. No architectural gap.

- **Drop `mac` from enrollment, keep only `machineGuid` (Decision #42).** Correctly removes the multi-NIC-ambiguous and unused field (addressing REVIEW-deepseek M5, REVIEW-mimo M5, REVIEW-qwen M4). No gap found.

- **Redistributed Phase 7.** Moving helper/session/sleep/clock to Phase 1, buffer-cap/revocation/org-key to Phase 2, browser variants/DPI to Phase 3, and AV/AppLocker to Phase 5 (addressing REVIEW-minimax L9, REVIEW-deepseek L7, REVIEW-kimi C1) is correct. L3 above notes the remaining catch-all, but the redistribution itself is sound. No architectural gap.

---

## Summary by severity

| Severity | Count | Key themes |
|----------|-------|------------|
| Critical | 1 | Mother→child documentation drift (console AGENTS.md still has deleted v1.0 route) |
| High | 5 | MongoDB recovery, pipe seq reset, transaction timeout, configSchemaVersion, orgTimeZone audit |
| Medium | 7 | Vacuous test scaffolding, dropped field lifecycle, unknown-device heartbeat, orgKeys cleanup, PowerModeChanged reliability, duplicate checklist item, batches TTL |
| Low | 4 | NTP correction vs field consistency, helperPid, Phase 7 catch-all triage, revoked-device reauthorization |
