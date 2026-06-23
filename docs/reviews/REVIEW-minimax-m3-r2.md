# Atlas Green — Pre-Implementation Design Review (Iteration 2)

- **Model:** minimax-m3 (opencode-go/minimax-m3)
- **Date:** 2026-06-23
- **Scope:** Stress-test the v1.1 fixes (decisions #28–#42, "decisions NOT taken"), pressure-test the five items the prompt called out, surface new edge cases the v1.1 plan missed, and sanity-check Phase 0 readiness + the redistributed Phase 7.

## Executive Summary

The v1.1 changes are largely well-targeted: `batchId` → UUID v4 with a `batches` receipt collection closes the dedup-resurrection hole; atomic ingest on a single-node replica set closes the partial-insert window; sleep/suspend + monotonic clock + clock-skew offset close the phantom-interval and wrong-day holes; the WASAPI gating against comms apps closes the "3-hour call = passive_media" false positive; the UIA → title-fallback chain + `urlCaptureMethod` makes URL loss visible rather than silent; DPAPI-scoped AES-GCM with ACL-primary keeps the self-contained publish native-dep-free; status is now a single server-owned enum. The phase redistribution is correct — sleep/session/clock belong in Phase 1, buffer-cap/revocation/org-key in Phase 2, AV/AppLocker in Phase 5. What v1.1 still leaves open, and what this review flags: the `batches` collection has no retention policy and grows ~11M docs/yr at 100 devices; the org-timezone (Asia/Dhaka) day-bucketing needs an explicit per-employee override for the rare non-Dhaka hire; the `status` state machine is defined as an enum but its transitions are not enumerated; `auditLog` is in Phase 4 but Phase 0 ships admin actions (device→employee mapping) that should be audited from day one; the helper's "considered down" threshold, the `helperRestartCount` "rapid" threshold, the dropped-row "prefer idle" policy, the first-heartbeat attributes semantics, and Bangladesh power-blip debouncing are all still unspecified; and the `batches.receivedAt` vs `intervals.receivedAt` relationship should be nailed down. None of these block Phase 0; they should be resolved before the phase that owns them, and one (auditLog infrastructure) should be pulled forward into Phase 0.

---

## Critical

None. The v1.1 changes close all six iteration-1 criticals (atomic ingest, batches receipt, dashboard auth, UIA fragility, helper launch, batchId scope/format). Phase 0 can start as planned once the items below marked "before Phase 0" are absorbed.

---

## High

### H1 — `batches` collection has no retention policy; the unique receipt grows ~11M documents/year and the compound index grows with it
**File:** `docs/DECISIONS.md` #28, `docs/ARCHITECTURE.md` §7, `docs/API-CONTRACT.md` §2
**Problem:** Decision #28 explicitly says the `batches` collection is "retained independently of `intervals`" so a stale offline replay can't resurrect admin-deleted data. That correctness property is right. But "retained independently" plus the "decisions NOT taken" decision to skip time-series / skip pre-aggregation means the receipts are kept **forever** at the same write rate as intervals. At 100 devices × ~288 sync windows/day (5-min ±30s jitter) = ~28,800 batch receipts/day = **~10.5M/yr**. After 3 years, ~31M. After 5 years, ~52M. The compound unique index `{deviceId, batchId}` is the same size. The "indefinite retention" rule is correct for intervals; it is wrong for receipts — the dedup property is the only thing the receipts buy you, and the dedup window is bounded by the buffer cap (75 days per `config.bufferCapDays`), not by the admin's deletion horizon. An admin deleting intervals 10 years from now is not a realistic use case; the receipts only need to outlive the **oldest** row an agent could still be holding in its local buffer (≤ 75 days) plus a safety margin.
**Recommendation:** Add a hard retention rule for `batches`: drop receipts older than `max(bufferCapDays × 2, 180 days)` (i.e. ~6 months default) via a TTL index on `receivedAt`. Document the property: "a stale replay more than 6 months old is treated as a new batch — the agent's local buffer cannot hold rows that old." Keep `intervals` retention indefinite per Decision #21. Add a Phase 2 checklist item: "TTL index on `batches.receivedAt`; manual override in `settings.retention.batchReceiptDays` for IT to extend." This is a one-line MongoDB index plus a setting; it stops the collection from being the long-term scaling cliff.

### H2 — Org-timezone (Asia/Dhaka) day-bucketing has no per-employee override; the first non-Dhaka hire silently mis-buckets their work
**File:** `docs/DECISIONS.md` #40, `docs/ARCHITECTURE.md` §8, `atlas-console/.agents/skills/atlas-api-contract/SKILL.md` §6
**Problem:** Decision #40 is correct that 99% of staff are in Bangladesh and per-employee timezone bucketing is over-engineering for v1. But the decision is "fixed org timezone" with no escape hatch. The minute a single remote hire joins (a Singapore-based consultant, a Dubai sales rep, an NYC contractor — all realistic for a Bangladesh company expanding), their work is bucketed into Dhaka days: a 9 AM Singapore call lands on the prior Dhaka day, a 5 PM Dubai close lands on the next. The dashboard for that one employee is wrong, and management reading "Tuesday" for that employee sees activity that's actually Monday evening + Tuesday morning. There is no flag, no per-employee setting, no way to spot the bug. The "decisions NOT taken" section explicitly defers per-employee timezone — but doesn't add the small "per-employee override" knob that would make the deferral safe.
**Recommendation:** Add a `tzOverride?: string` (IANA tz, e.g. `"Asia/Singapore"`) field on the `employees` document, defaulting to the org timezone when unset. The dashboard's day-bucketing logic reads `employees.tzOverride ?? settings.orgTimeZone`. The agent's interval still stamps UTC (no behavior change for the agent); only the dashboard grouping changes. This is a 5-line server change and a 1-line dashboard change. Document in Decision #40's "future override" note: "v1 buckets by org timezone; a single per-employee IANA override exists for non-org hires; per-employee bucketing is still deferred."

### H3 — `auditLog` collection is in Phase 4, but Phase 0 ships an admin action (device→employee mapping) that must be audit-logged from day one
**File:** `docs/PLAN.md` Phase 0 + Phase 4, `docs/ARCHITECTURE.md` §7, `docs/DECISIONS.md` "Open/deferred" (audit log)
**Problem:** Phase 0's checklist includes "Device → employee mapping stub (assign a name to a device)" — a real admin action that retitles intervals, remaps who-management-thinks-they're-watching, and (once Phase 4 lands re-resolution) re-categorizes historical intervals. Phase 4 adds the `auditLog` collection. From Phase 0 through Phase 3, admin actions are written nowhere. Once Phase 4 ships, the schema is in place but the history is gone — there's no "who mapped PC-07 to Sadman on 2026-07-12?" The privacy/legal value of an audit log is exactly that it was running when the first decision happened; retrofitting it leaves the most sensitive period (the trial rollout) un-audited.
**Recommendation:** Split the audit log into infrastructure (now) and viewer (later). Move **only** the `auditLog` collection + the write helper + the device→employee mapping event into Phase 0; defer the dashboard "audit log viewer" to Phase 4. The collection shape (`{ _id, adminId, action, target, before?, after?, atUtc }`) is already in `ARCHITECTURE.md` §7 — just create it and call `auditLog.insert(...)` from the Phase 0 mapping endpoint. Phase 4 then only builds the viewer page. This is <50 lines of code in Phase 0 and prevents the "first month is invisible" hole.

### H4 — `status` state machine enum is defined but transitions are not; multi-source-of-truth risk persists in v1.1
**File:** `docs/DECISIONS.md` #39, `docs/ARCHITECTURE.md` §7, `atlas-console/.agents/skills/atlas-api-contract/SKILL.md` §7
**Problem:** Decision #39 collapsed the three ad-hoc derived fields (`lastSeenAt`, `archived` boolean, `revokedAt`) into a single server-owned `status` enum: `pending | online | offline | outdated | clock_skewed | revoked | archived`. That's the right design. But the **transitions** are not specified. Without them, implementers will guess: can `pending → revoked` skip `online`? Can `online → archived` happen while a device is still heart-beating? Can `revoked` and `archived` co-exist (revoked = token dead, archived = hidden — usually sequential but not defined)? Can `outdated` and `clock_skewed` co-exist? Can `revoked` be un-set by a successful re-enroll (Decision #38 says no — but only by implication)? The risk is a re-emergence of the multi-source-of-truth problem Decision #39 was meant to fix, just inside one enum.
**Recommendation:** Add a transition table to `docs/ARCHITECTURE.md` §7 next to the enum. Minimum cells: from × (event) → to. Example:
- `pending` + (first heartbeat) → `online`
- `online` + (no heartbeat in `heartbeatIntervalSec × 3`) → `offline`
- `online` + (admin `revoke` action) → `revoked`; sets `deviceTokens.revokedAt` and `devices.blocked = true`
- `online` + (admin `archive` action) → `archived` (still heartbeats, hidden from default dashboard views)
- `online` + (agent version < `minSupportedVersion`) → `outdated` (orthogonal flag, can co-exist with `online`/`offline`/`clock_skewed`)
- `online` + (stable `|clockOffset| > 1h` over 2 heartbeats) → `clock_skewed` (orthogonal)
- `*` + (`devices.blocked = true`) → `revoked` (re-enroll returns 403)
- `revoked` is terminal (no auto-revert); admin must explicitly `unrevoke` to re-enroll

Document that `outdated` and `clock_skewed` are orthogonal flags (a device can be both `online + outdated + clock_skewed`), while `pending/online/offline/revoked/archived` are the "main" state. This costs 10 lines of doc and prevents a Phase 4 rework.

---

## Medium

### M1 — Helper "considered down" threshold and `helperRestartCount` "rapid" threshold are unspecified
**File:** `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §2–§3, `docs/API-CONTRACT.md` §3
**Problem:** The heartbeat `capture` block reports `helperRunning`, `lastSampleUtc`, `helperRestartCount` (Decision #41). The skill says "repeated rapid restarts raise `helperRestartCount` (surfaced via heartbeat — see §6)" and the watchdog "restarts a dead helper." Neither threshold is defined. Without a "helper is down" threshold, the service can't decide when to set `helperRunning = false`; without a "rapid" threshold, the dashboard can't flag tamper-suspected. Defaults will diverge between implementers.
**Recommendation:** Add to the skill §2: "Helper is considered down (`helperRunning = false`) when the named pipe has not received a sample in **60 s** AND the service has retried the connection once." Add to skill §3: "A restart is `rapid` (raises a tamper flag) when **5 restarts occur within 60 s**; the dashboard surfaces `helperRestartCount > 10` as a yellow indicator and `> 50` as a red 'tamper suspected' indicator." Both are easy to test (mock the pipe clock; assert the field). This pairs with the existing `helperRestartCount` field — currently the field exists but its semantics are unwritten.

### M2 — Bangladesh power blips will create hundreds of "idle" intervals per day unless suspend events are debounced
**File:** `docs/DECISIONS.md` #33, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §3
**Problem:** Decision #33 closes the open interval on `PowerModeChanged` (suspend) and starts fresh `idle` on resume — correct for an 8-hour sleep. Bangladesh office power is notoriously flaky: a typical day has 5–20 sub-second to multi-second blips. A 1-second blip fires `PowerModeChanged` to Suspend and back to Resume. The aggregator closes the open `active` interval at the suspend timestamp and opens a new `idle` interval at the resume timestamp — for a 1-second blip, this produces two phantom intervals (1s `active` and ~1s `idle` after, until the next sample runs). Over 20 blips/day, the timeline fragments into 20+ extra intervals and the dashboard's "active" hours are slightly under-counted. The same applies to laptop sleep on lid-close-and-immediate-open.
**Recommendation:** Add to skill §3: "Debounce `PowerModeChanged`: ignore Suspend/Resume pairs whose `ResumeTimestamp − SuspendTimestamp < 30 s`. For genuine sleep, the open interval closes once and the new `idle` starts on resume. This prevents BD power-blip fragmentation." The 30 s threshold is well under the 5-min idle threshold, so a real "lunch break sleep" still triggers the close. One line in the PowerModeChanged handler; easy to unit-test (feed in a synthetic blip timeline, assert one open interval survives).

### M3 — Buffer-cap drop policy says "prefer dropping idle" but the contract doesn't enforce it
**File:** `docs/API-CONTRACT.md` §3 (`dropped` block), `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §4
**Problem:** The skill says "prefer dropping `idle` rows over `active`/`passive_media`" — a sensible triage, since idle is the least informative state. The contract §3 `dropped` block is "present only when the buffer cap forced a drop" with `count, oldestUtc, newestUtc`, but doesn't say **which** rows were dropped. The server stores the count and dates, but the dashboard can't tell the operator "we dropped 14,000 idle rows" vs "we dropped 1,200 active rows" — two very different stories.
**Recommendation:** Extend the `dropped` block in the contract: `dropped: { count, oldestUtc, newestUtc, byState: { active: N, passive_media: M, idle: K } }`. The agent tallies the per-state count at drop time. The dashboard then displays "Buffer overflow: 12,400 idle + 600 active + 200 passive_media dropped" — instantly tells the operator whether the situation is "machine was idle for 2 months" (cheap loss) or "machine was capturing valuable work and we threw it away" (expensive loss, page IT). This is a 3-field addition and a `GROUP BY state` at the drop query.

### M4 — First-heartbeat "send only when changed" is undefined for `attributes` and `os`
**File:** `docs/API-CONTRACT.md` §3
**Problem:** The heartbeat `attributes` block is "sent only when a mutable attribute changed." The `os` field is "sent when changed." But "changed since when?" isn't specified. The first heartbeat after enrollment has no prior baseline, so the agent has to send everything (otherwise the server's view is empty until a change occurs). On agent restart, does the agent re-send everything (the values are still in memory) or only the changed-from-last-sent ones (the last-sent baseline is gone)? Without an explicit rule, two implementers will pick two behaviors and the dashboard will be wrong on a fixed-rate basis.
**Recommendation:** Add to the contract §3: "On the **first** heartbeat after enrollment and on the **first** heartbeat after an agent restart, the agent sends `attributes` and `os` unconditionally. On subsequent heartbeats, they are sent only when the current value differs from the last value the agent sent. The agent persists the last-sent `os` and the last-sent `attributes` in the SQLite `agent_state` table (survives restart, so restart is not treated as a baseline reset)." This is one paragraph in the contract and one tiny table in the agent's SQLite buffer. The skill should also note: "If the agent has never sent `attributes` (fresh install, no post-enroll heartbeat yet), it must send on the next heartbeat."

### M5 — `batches.receivedAt` and `intervals.receivedAt` are both server-set but their relationship is undefined
**File:** `docs/API-CONTRACT.md` §2, `docs/ARCHITECTURE.md` §7
**Problem:** Both the `batches` receipt doc and each `intervals` doc have a `receivedAt` (server-set at insert). In the atomic-ingest transaction they're inserted together, but the contract doesn't say they must be the same timestamp. Two natural readings: (a) the same value (transaction commit time, one `DateTime.UtcNow` for the whole batch), or (b) the per-insert time (a few microseconds apart, but technically different). The dashboard's "data freshness" panel will compare them; if they differ by milliseconds the dashboard may show "intervals received at T+2ms after batch — anomaly?" which is a non-event but a confusing panel.
**Recommendation:** Add to the contract §2 (and ARCHITECTURE §7): "In an atomic ingest, `batches.receivedAt` and every `intervals[i].receivedAt` are set to the same value — the transaction commit time, captured once before the transaction begins and stamped on every doc written inside it." This is the natural implementation (one `DateTime.UtcNow` captured at the top of the transaction handler) and makes the dashboard's freshness queries straightforward (`MAX(receivedAt) FROM intervals WHERE deviceId = X` = `MAX(receivedAt) FROM batches WHERE deviceId = X`).

### M6 — Console-session-only helper means WFH-via-RDP days produce zero intervals; the dashboard will silently look "idle" with no flag
**File:** `docs/DECISIONS.md` #32, `docs/ARCHITECTURE.md` §2.1, `docs/PLAN.md` Phase 1
**Problem:** Decision #32 says "v1 monitors the console (physical) session only — RDP sessions are ignored." This is the right call (one employee, one desktop), but it has a non-obvious consequence. When a Bangladesh employee works from home via RDP, the **console** session is still the local desktop (locked or with another user signed in); the RDP session is a separate session ID. The helper, watching only the console session, sees either a locked screen or no foreground. The result: a full WFH day produces zero intervals, and the dashboard shows that employee as "idle" for the entire day. To management, this is indistinguishable from "employee was idle all day at their desk." For a post-COVID hybrid-work world, this is a meaningful false negative that the plan doesn't surface.
**Recommendation:** In the dashboard's daily summary, distinguish "no intervals" into two cases: (a) `lastSampleUtc` is recent and `state = idle` for the whole day → "idle" (low signal); (b) `lastSampleUtc` is **stale** (no helper activity) and the service is still heart-beating → "no console session / WFH" (a different signal). This is a 2-line dashboard query: `lastSampleUtc > (now − 2 × heartbeatInterval)` distinguishes "helper alive and idle" from "helper dead or session changed." Document the behavior in the dashboard README: "WFH-via-RDP days show as 'no console activity' (helper not running) — not the same as 'idle' (helper running, no input)."

### M7 — Single-node replica set has no automatic failover; outage = 2-3 month ingest pause risk
**File:** `docs/DECISIONS.md` #29, `docs/ARCHITECTURE.md` §8, `docs/PLAN.md` Phase 6 (backup)
**Problem:** Decision #29 mandates a single-node replica set so transactions work. At 100 devices, this is operationally fine — the load is ~0.3 req/s and MongoDB on a VPS handles it easily. But: a single-node "replica set" is a single point of failure. If the VPS disk corrupts, the MongoDB data is gone (backups from Phase 6 help, but a full day's data is lost). If the VPS is unreachable, the agents buffer locally — fine for the 2-3 month cap, but the dashboard shows "no data" for the entire outage. The current plan mentions "daily `mongodump` → R2" in Phase 6 as the mitigation, but the failure window between detection and restore is not bounded.
**Recommendation:** Document in `docs/ARCHITECTURE.md` §8 and the operational runbook: (a) "VPS outage = data ingest pauses; agents buffer locally; the 2-3 month buffer cap protects against long outages; restore from R2 `mongodump` resumes the dashboard with a `dataGap` window." (b) "VPS disk loss = restore from the most recent `mongodump` (≤ 24 h of data lost). Document the restore procedure and rehearse it quarterly." (c) For a single-node RS, the `writeConcern: majority` setting has no meaning (no secondaries) — document that the durability guarantee is `writeConcern: { w: 1, j: true }` (journal committed), which is what a single-node RS actually provides. This costs nothing operationally and lets the runbook say "here's what survives a failure" with confidence.

### M8 — The agent's `machineGuid` (Decision #42) requires SYSTEM-level registry read; the `/enroll` is in the service, not the helper, so this works — but it should be documented
**File:** `docs/DECISIONS.md` #42, `docs/ARCHITECTURE.md` §5, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §5
**Problem:** Decision #42 keeps `machineHints.machineGuid` (informational, no dedup). `machineGuid` lives at `HKLM\SOFTWARE\Microsoft\Cryptography` and a standard user **cannot** read it. The architecture says the service runs as LocalSystem, and the helper runs as the user with no networking. The natural reading: the agent helper can't read `machineGuid`; the service (as LocalSystem) can. The skill says "Persist the **DeviceId GUID** once" but doesn't say **where** `machineGuid` is read. An implementer might put the read in the helper (fails silently for standard users) or in the service at `/enroll` time (works). This is the kind of detail that bites during Phase 0.
**Recommendation:** Add to skill §5: "The agent's helper does **not** read `HKLM` (it runs as a standard user and cannot). The service reads `machineHints.machineGuid` at `/enroll` time and includes it in the enrollment request body. If the read fails, the field is sent as `null` — enrollment still succeeds." This is one paragraph that prevents a Phase 0/Phase 1 misimplementation. Bonus: it pairs with the "no dedup in v1" decision — even if the field is sometimes null, that's fine because it's not used.

### M9 — `agentVersion < minSupportedVersion` has no `outdated` device-side behavior; the agent keeps heart-beating and ingesting but is never forced to update
**File:** `docs/API-CONTRACT.md` §4 (manifest), `docs/DECISIONS.md` #39 (`outdated` state)
**Problem:** The contract §4 says below `minSupportedVersion`: "the server still **accepts ingest** (never lose data) but marks the device `outdated` so IT can intervene." Good. But there's no **agent-side** behavior: the agent receives the manifest, sees its version is below `minSupportedVersion`, and... just keeps running? The "force update" path is implicit (the manifest says "newer version exists" → agent downloads). If the manifest's `mandatory: true` is set, the agent must update. If `mandatory: false`, the agent can defer. The contract doesn't define `mandatory: true` semantics — is it "must update before next sync" or "should update before next sync but the server won't reject ingest either way"? If mandatory, what stops the agent from ignoring the flag?
**Recommendation:** Add to contract §4: "If `update.mandatory == true`, the agent blocks the next `/ingest` (and the next `/heartbeat` other than a `updateStatus: failed` report) until the update completes or fails. Failed updates re-attempt on the next cycle; the agent does not block forever. Below `minSupportedVersion` AND `update.mandatory == true` is the only 'force' gate; the server still accepts ingest per the 'never lose data' rule, but the agent chooses to withhold it locally until the update attempt resolves." This is the rare case where "never lose data" (server-side) and "must update" (agent-side) need to be reconciled explicitly. Without it, a fleet-wide `mandatory: true` rollout either does nothing (agents ignore) or breaks the data-pipeline guarantee (agents block forever on a flaky download).

### M10 — The 413 split strategy assumes linear sub-batches; a single sub-batch can itself be too large
**File:** `docs/API-CONTRACT.md` §2 (413), §5 (Payload size)
**Problem:** Contract §2: "On `413`, the agent splits into sub-batches, each with its own fresh UUID `batchId`, and retries them independently. Hard cap: ≤ 5000 intervals or ≤ 5 MB decompressed per batch." Good. But: if the unsynced rows are 20,000 intervals and the cap is 5,000, the agent splits into 4 sub-batches of 5,000 each. If, by coincidence, all 5,000 intervals in sub-batch 1 are 2 KB each, sub-batch 1 is 10 MB and 413s again. The agent must split by row count AND by byte budget, and the recursive case must be handled (split a too-large sub-batch further). With 5-min sync and ~300 intervals/day, hitting 5,000 intervals in a single batch is rare; with offline accumulation (1 month of data on a laptop that's been off), it's normal.
**Recommendation:** Add to contract §2 / §5: "On `413`, the agent splits the offending batch by row count to fit the cap. If a sub-batch still 413s, the agent further splits by halving row count and/or by switching from a single gzip body to a per-interval page (the API stays a single endpoint; the body shape is unchanged). The agent keeps splitting until each sub-batch is under the cap or the sub-batch is 1 interval (which would 400 and be reported as a poison row). The agent must persist the sub-batch plan across restarts (so a restart mid-split doesn't lose track of which sub-batches were sent)." This is a 4-line clarification; the implementation is straightforward (recursive halving). Without it, an offline laptop coming back online in a single 20,000-interval payload hits a 413 loop.

### M11 — The dropped `attributes` change semantics interact badly with admin-side `pcName` renames
**File:** `docs/API-CONTRACT.md` §3 (`attributes`), `docs/DECISIONS.md` #16
**Problem:** Decision #16 says "PC name set at install, changeable from dashboard." The dashboard can rename `pcName`; the agent's local `pcName` is unchanged. The next heartbeat from the agent reports `attributes.pcName = <old>`, the server compares to the device doc's `pcName` (which the admin just changed to `<new>`), and... what? If the server accepts the agent's report, the admin's rename is reverted by the next heartbeat. If the server rejects the agent's report, the agent retries forever. Neither is right.
**Recommendation:** Add to contract §3: "The server treats `attributes` from the agent as **informational** for `pcName` (admins own `pcName`); for `hostname` and `osUsername` (which only the OS/agent knows), the server **applies** the agent's value. The agent's `pcName` in the heartbeat is ignored for storage; it is logged for debugging only." This is a 1-line rule that prevents an agent-vs-admin ping-pong on `pcName`. Document which fields are admin-owned (pcName) vs agent-owned (hostname, osUsername). Pair with Decision #16's "changeable from dashboard" to make the ownership explicit.

### M12 — Heartbeat `health` thresholds are not defined (qwen L3 in iter-1 still open)
**File:** `docs/API-CONTRACT.md` §3, `atlas-console/.agents/skills/atlas-api-contract/SKILL.md` §7
**Problem:** The `health` block (`cpuPct`, `memMb`) is collected on every heartbeat but no threshold defines "unhealthy." The data is collected but not actionable. A misbehaving agent chewing 200 MB of RAM looks identical to a healthy one.
**Recommendation:** Add to the contract §3 (or skill §7): "Server flags the device `health_warning` in the dashboard if `cpuPct > 5` sustained across 3 heartbeats, or `memMb > 200`." The flag is a non-blocking signal — the agent keeps ingesting. This is one MongoDB comparison per heartbeat at 100 devices = trivial. v1.1's status enum should add `health_warning` as an orthogonal flag (like `outdated` and `clock_skewed`) so the dashboard can show "agent consuming too much CPU" without taking the device offline.

---

## Low

### L1 — The "console session only" rule needs a fallback when the service is installed before the first user login
**File:** `docs/DECISIONS.md` #32, `docs/ARCHITECTURE.md` §2.1
**Problem:** The MSI installs the service, which starts at boot. If the machine boots to the login screen (no user logged in), `WTSGetActiveConsoleSessionId` returns the session ID of the login screen. `WTSQueryUserToken` on the login screen's session returns... a token for the login process? Or fails? The Windows logon screen is a separate session; the helper, if launched with that token, runs as SYSTEM (the login process) and sees nothing useful. The service should simply **not** launch the helper until a real user is logged in. Decision #32 implies this ("when no user is logged in, the service still heartbeats from Session 0 but produces no intervals") but doesn't say "the service must wait for `WTS_SESSION_LOGON` of a real user, not a session-0 / logon-screen session."
**Recommendation:** Add to skill §2: "The service launches the helper only after receiving `WTS_SESSION_LOGON` for the console session with a **non-zero** user token (i.e., a real user, not the logon UI). During the login screen and during the lock screen, no helper is running. `WTSRegisterSessionNotification` events drive launch and teardown."

### L2 — `batches` collection's `accepted` count is set at insert time but the contract never re-states it
**File:** `docs/API-CONTRACT.md` §2, `docs/ARCHITECTURE.md` §7
**Problem:** The `batches` receipt has `accepted` (the count of intervals in the batch). On a duplicate replay, the contract says "returns the original `accepted` count with `duplicate: true`." The `batches` doc stores `accepted` from the original successful insert. Good. But: if the original transaction inserted a different number of intervals than the request body says (e.g., the server dropped 1 poison row at validation time, returning `accepted: 36` instead of 37), the `batches.accepted` must match the response. If it doesn't, the agent purges the wrong number of rows.
**Recommendation:** Make this explicit in contract §2: "`batches.accepted` equals the `accepted` value returned to the agent in the response. The two are set together in the transaction; the agent trusts both to mean 'N intervals durably stored — purge those N from the buffer.' If the server filters out malformed intervals during validation, `accepted` is reduced and the `batches.accepted` reflects the actually-stored count."

### L3 — The `helperRunning` field is read by the server but the *agent* doesn't know if the helper is "running" from the service's perspective
**File:** `docs/API-CONTRACT.md` §3 (`capture` block)
**Problem:** The `capture.helperRunning` field is the service's view of the helper (pipe connected? last sample < N seconds ago?). It's set on the service side and shipped in the heartbeat. Fine. But the contract example shows `helperRunning: true` in the request body, which implies the agent (service) is reporting it. The service knows this — it's the one tracking the pipe. The naming is fine; the only nit is that the `capture` block in the request is the **service's** view of the helper, not the helper's view of itself. The helper could ship a "I'm alive" message too, but the service can already tell.
**Recommendation:** Document in contract §3: "`capture.helperRunning` is the service's view (pipe state + sample-recency), not the helper's self-report. The helper has no way to know if the service thinks it's running, and the service has no reason to ask." One line, prevents a misreading.

### L4 — `os` field comparison is string-equal; Windows Update 23H2 → 24H2 should be a real "changed" event
**File:** `docs/API-CONTRACT.md` §3 (`os`)
**Problem:** The `os` field is `"Windows 11 Pro 23H2"` — a free-form string. The agent reports it "only when changed." String equality works for the obvious case. But a Windows feature update (23H2 → 24H2) changes the string; a Windows cumulative update (23H2 → 23H2 build 12345) does not. For tracking purposes, the dashboard might want build-level granularity; for storage purposes, feature-update-level is enough. This is a low priority nit but worth flagging.
**Recommendation:** Document the granularity: "The `os` field is the **feature-update** string (e.g. `Windows 11 Pro 23H2`), updated on feature-update events only. Cumulative update build numbers are not reported in v1 (the dashboard's OS column shows the feature update at a glance; deep build-level reporting is a v2 concern)."

### L5 — R2 package retention for downgrade is unspecified
**File:** `docs/API-CONTRACT.md` §4 (manifest), `docs/ARCHITECTURE.md` §9
**Problem:** The contract §4 says `latestVersion < current` is a valid downgrade signal. The agent downloads the older package from R2. But: how many old packages does R2 keep? If R2 lifecycle policy deletes packages older than 30 days, a 31-day-old "downgrade target" is 404. The agent reports `updateStatus: failed` and the fleet can't roll back.
**Recommendation:** Add to ARCHITECTURE §9: "R2 retains the **last 5** package versions (not just `latestVersion`) and the matching signed manifest for each. The lifecycle policy is: never delete the current `latestVersion` and the previous 4 versions. The CI/CD pipeline enforces this on release." This is a one-line R2 lifecycle policy + a CI guard.

### L6 — `auditLog.adminId` doesn't yet exist as an identity in the plan
**File:** `docs/ARCHITECTURE.md` §7, `docs/DECISIONS.md` "Open/deferred" (audit log), `docs/PLAN.md` Phase 4
**Problem:** The `auditLog` doc has `adminId`. The dashboard has Cloudflare Access + (Phase 4) NextAuth. But Cloudflare Access identifies the admin by **email**; NextAuth uses its own user table. Until Phase 4 lands NextAuth, the only identity available to a Phase-0 admin action is the Cloudflare-Access JWT's `email` claim. The audit log entry needs an `adminId` — if it's the email, document that; if it's a synthetic UUID assigned at first login, document that too.
**Recommendation:** In the Phase 0 audit-log decision (per H3), document: "Phase 0 `adminId` is the Cloudflare-Access-authenticated email. Phase 4 NextAuth assigns a per-admin UUID; the audit log retains both the email (for human readability) and the UUID (for stability across email changes)." One paragraph.

### L7 — Helper's process identity: an admin running `tasklist | findstr Atlas` shouldn't be able to tell "this is the agent" from the process name
**File:** `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §1, `docs/DECISIONS.md` #17
**Problem:** The agent is silent to the user, but a determined standard user can open Task Manager, see `AtlasAgent.Helper.exe` in the process list, and either kill it (fine — watchdog restarts) or research what it is. The threat model says standard users can kill it; that's accepted. But the process name should not be a flashy "atlas-monitor.exe" — it should look like a generic Windows component.
**Recommendation:** Document the service+helper process name in skill §1: "`AtlasAgent.Service` and `AtlasAgent.Helper` are the dev names. The published binary is renamed to a generic Windows-component-like name (e.g. `WindowsDeviceHealth.exe`, `MicrosoftEdgeUpdateHelper.exe`-style) so the process list looks like a Microsoft component. The mapping from the friendly name to the published name is in `installer/Service.naming.md` (or equivalent)." The published name should not contain "atlas", "monitor", "agent", "spyware", or anything searchable. Low priority (the threat is "standard user kills it" which is already accepted) but cheap to do right the first time.

### L8 — The `batches` collection's `deviceId` should be a foreign-key reference, not a free string
**File:** `docs/ARCHITECTURE.md` §7
**Problem:** The `batches` doc stores `deviceId` (the device's GUID). MongoDB doesn't enforce FK relationships, but the schema should still note it. A buggy script that renames a `devices._id` would orphan `batches` rows. (The receipt's uniqueness is on `batchId`, not `deviceId`, so orphans are not correctness bugs — but a dashboard query "show me receipts for device X" silently misses them.)
**Recommendation:** Add a note to ARCHITECTURE §7: "`batches.deviceId` is logically a foreign key to `devices._id`. The DB does not enforce this; orphan `batches` rows (if a device doc is hard-deleted) are a known consequence and are handled by the receipts' retention policy (H1) — orphans are eventually TTL-evicted." Pairs naturally with H1.

---

## Nice-to-Have

### N1 — Add a chaos-test matrix to Phase 7
**File:** `docs/PLAN.md` Phase 7
**Problem:** Phase 7's "100-device soak test" is defined as "24 h, zero dropped batches, dashboard queries < 2 s." That's a load test, not a chaos test. Real failure modes (helper crash, pipe death, network partition, MongoDB transaction abort, agent re-enroll loop, blocked-device retry) aren't in the matrix.
**Recommendation:** Add to Phase 7: "Chaos scenarios: (1) kill helper on 5% of agents for 5 min (verify `helperRunning=false` surfaces, no data gap > 2 × heartbeat). (2) Block all 100 agents' network for 10 min (verify backlog flushes in < 30 min after restore, zero duplicates). (3) Re-enroll 1% of agents mid-stream (verify no orphan rows). (4) Revoke + block 2% of agents (verify they go to `revoked`, not `online`). (5) Inject a single bogus `batchId` from one agent (verify the receipt rejects without affecting other agents). Each scenario has a pass criterion." The scenarios are testable, the criteria are concrete, and the matrix is a checklist.

### N2 — MongoDB change streams for the dashboard's "live data" view
**File:** `docs/ARCHITECTURE.md` §7
A v2 enhancement: the dashboard polls for new intervals every N seconds. With MongoDB change streams, the dashboard can subscribe to `intervals` and update in real-time as ingest commits. Adds operational value (the demo to management is "see, the data appears instantly") at the cost of a connection per dashboard session. Defer to v2 but mention in ARCHITECTURE §7 as a possible optimization.

### N3 — A `batches.lastIntervalEndUtc` denormalized field for "agent freshness" queries
**File:** `docs/ARCHITECTURE.md` §7
A common dashboard query is "when did this device last capture activity?" (not just last heartbeat). The current data model requires `MAX(endUtc) FROM intervals WHERE deviceId = X` — fast with the index, but a denormalized `devices.lastIntervalEndUtc` updated at ingest-commit time is O(1). Optional; defer.

### N4 — A `tokens.lastUsedAt` field on `deviceTokens` for "stale token" detection
**File:** `docs/ARCHITECTURE.md` §7
A `deviceTokens` doc with `lastUsedAt` updated on each /ingest or /heartbeat call lets the admin surface "this device has a valid token but hasn't used it in 30 days — is it dead?" Optional.

---

## What v1.1 Already Has Right (do not change)

These are the v1.1 changes that correctly address iteration-1 findings. Calling them out so future agents don't re-litigate.

- **Decision #28 — UUID v4 `batchId` + `batches` receipt collection.** The UUID removes the timestamp/sequence-collision and restart-reuse risk; the receipt collection decouples dedup from interval lifecycle. The reasoning is correct.
- **Decision #29 — Atomic ingest via MongoDB transaction on a single-node replica set.** This is the only correct way to make "ACK before purge" actually safe. The single-node RS is the standard workaround and is the right choice at 100 devices (load is trivial, no need for a real multi-node cluster).
- **Decision #30 — App-layer AES-GCM with key in DPAPI machine scope.** No native dependency, full defense-in-depth pairing with the ACL, and the encryption-approach is now concrete. Good.
- **Decision #31 — Cloudflare Access + `middleware.ts` in Phase 0.** The dashboard never deploys open. The Phase 0 wiring prevents the worst single privacy failure.
- **Decision #32 — `WTSQueryUserToken` + `CreateProcessAsUser` + `WTS_SESSION_CHANGE` + console session only.** The mechanism is named, the privilege requirement (LocalSystem + `SE_TCB`) is implied, the policy (console only) is explicit. The single trickiest Windows-interop piece is no longer hand-waved.
- **Decision #33 — `Stopwatch` for durations, `PowerModeChanged` for sleep, `clockOffset` on the device.** Closes the phantom-interval and wrong-day holes. The "never rewrite buffered timestamps" rule preserves raw data integrity.
- **Decision #34 — Idle hysteresis (300 s in / 30 s out, sub-5 s merged).** Prevents the state-machine oscillation that defeats Model B. The numbers are right.
- **Decision #35 — UIA → title-fallback → `domain: null` with `urlCaptureMethod`.** The URL capture is no longer a single point of failure; the dashboard knows which method was used; eTLD+1 normalization is the rule.
- **Decision #36 — `pageTitle` ≤ 256, `windowTitle` ≤ 512.** Caps both the storage cost and the privacy surface.
- **Decision #37 — `employeeId` denormalized at ingest.** The hot dashboard query (`{employeeId, startUtc}`) is now index-friendly with no join.
- **Decision #38 — Org key array with `expiresAt`/`revokedAt`; blocked device → `403` at `/enroll`.** Rotation is now concrete; a revoked device can't un-revoke itself by re-enrolling.
- **Decision #39 — Single `status` enum, server-owned.** Collapses the three ad-hoc fields into one. The "dashboard reads only `status`" rule is the right invariant.
- **Decision #40 — Org-timezone (Asia/Dhaka) day-bucketing.** Correct for 99% of staff. The override hook (H2) is the small fix.
- **Decision #41 — `capture` block in heartbeat + rate limits.** "Online ≠ capturing" is now visible; rate limits cap a rogue agent.
- **Decision #42 — Drop `mac`, keep `machineGuid` (informational).** Multi-NIC ambiguity removed.
- **Phase redistribution.** Helper / session / sleep / clock correctly in Phase 1; buffer-cap / revocation / org-key in Phase 2; browser variants / DPI in Phase 3; AV / AppLocker / WDAC in Phase 5; only the genuinely-deferable items (multi-monitor, PWAs, soak test, end-to-end revocation flow) remain in Phase 7. The "Phase 7 drag" risk is gone.
- **v1.1 contract clarifications.** Same `config` shape in `/enroll` and `/heartbeat`; `batchId` echo in ingest response; `X-Atlas-Contract: 1.1` header; `Retry-After` honored; 413 → fresh-UUID sub-batches; config validation ranges.

---

## Phase 0 Readiness — what's missing or under-specified

Phase 0 is mostly ready to start cleanly. Three small additions and a few under-specified items:

1. **Add `auditLog` collection + write helper to Phase 0** (H3). Just the infrastructure; the viewer is Phase 4.
2. **Add `/api/v1/health` endpoint to Phase 0** (`docs/ARCHITECTURE.md` §8 mentions it but no phase owns it). The endpoint is trivial; the external uptime monitor can be wired in Phase 6 but the route should exist from day one so a deploy failure is detected.
3. **Add `npm run seed:dev` (or equivalent) location and contents to Phase 0 README** (currently the seed script is mentioned but its file path is not in the checklist).
4. **Specify log-rotation policy** in Phase 0 (max 10 MB × 5 files is a reasonable default; qwen L1 in iter-1).
5. **Specify config defaults** in Phase 0 (sample=4 s, idle=300 s, sync=300 s, heartbeat=300 s, bufferCap=75 d — the contract gives the ranges, not the defaults).

None of these are blockers; they are paperwork that the implementer will need to invent otherwise. The exit gate is concrete and falsifiable; that's the important property.

---

## Phase Ordering — the redistribution is correct

The redistributed Phase 7 is right. Sleep / session / clock / foreground-hijack / idle-hysteresis / windowless-helper all correctly belong in Phase 1 (the aggregator that owns them). Buffer-cap / revocation / org-key-rotation / rate-limiting / status-state-machine correctly belong in Phase 2 (the data path that owns them). Browser variants / DPI manifest / eTLD+1 normalization correctly belong in Phase 3 (the URL feature that owns them). AV / SmartScreen / Defender / AppLocker / WDAC / single-file-publish-smoke correctly belong in Phase 5 (the packaging that owns them). What remains in Phase 7 is genuinely deferable: multi-monitor, PWAs/Electron `appName` edge cases, 100-device soak test, and end-to-end revocation flow on a real agent. The single suggestion is to add the chaos-test matrix (N1) to Phase 7 so the soak test is more than just a load test.

The only sequencing concern is **`auditLog` belongs in Phase 0** (H3), not Phase 4, because Phase 0 ships the device→employee mapping admin action.

---

## Summary by Severity

| Severity | Count | Theme |
|---|---|---|
| Critical | 0 | v1.1 closes all iter-1 criticals |
| High | 4 | `batches` retention; org-tz override; auditLog in Phase 0; `status` transitions |
| Medium | 12 | Helper thresholds, debounce, drop policy, first-heartbeat semantics, `receivedAt` alignment, WFH flag, single-node-RS runbook, machineGuid read path, force-update semantics, 413 recursion, attributes ownership, `health` thresholds |
| Low | 8 | Logon-screen, `accepted` count alignment, `helperRunning` source, `os` granularity, R2 retention, adminId identity, process naming, `batches` FK note |
| Nice-to-have | 4 | Chaos matrix, change streams, denormalized freshness fields, stale-token detection |

**Before Phase 0 starts:** H3 (auditLog infra), Phase 0 readiness items 1–5.
**Before the phase that owns them:** H1 (Phase 2), H2 (Phase 4), H4 (Phase 4), M1–M12 (their respective phases), L1–L8 (their respective phases).
**Not blockers:** All Nice-to-haves.

---

*End of review. No code or existing documents were modified; only this file was created.*
