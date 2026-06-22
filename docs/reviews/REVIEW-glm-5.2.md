# Atlas Green — Pre-Implementation Design Review

- **Model:** glm-5.2 (opencode-go/glm-5.2)
- **Date:** 2026-06-22
- **Scope:** Critical pre-implementation review of the atlas-green mother repo docs, atlas-agent, and atlas-console, before Phase 0 coding begins.
- **Mode:** Design review only — no code or existing docs were modified.

## Executive summary

1. The architecture is fundamentally sound: the phased, doc-first, contract-first discipline, the two-part Service+Helper, interval aggregation (Model B), the 3-state model, ACK-before-purge, and per-device DPAPI tokens are all the right calls for a 100-device silent fleet built by AI agents.
2. **Three High issues must be fixed before Phase 2 ships:** (a) `/ingest` is not specified as atomic per batch, so a partial insert + retry with `batchId` dedup can silently lose intervals; (b) there is no batch-receipt collection independent of interval lifecycle, so a stale agent retry can resurrect data an admin already archived/deleted; (c) the dashboard has no admin authentication anywhere in the plan.
3. Several Mediums cluster around under-specified Windows interop: the helper launch mechanism (`WTSQueryUserToken` + `CreateProcessAsUser` as LocalSystem, one helper *per* session for fast-user-switching/RDP) is never named; the heartbeat reports service health but not capture health, so "online" ≠ "capturing"; and sleep/resume + clock-jump handling must be designed into the Phase 1 aggregator, not deferred to Phase 7.
4. URL-via-UIA is feasible and is the right choice for incognito coverage, but it is the most fragile piece (a Chrome update can degrade it fleet-wide with no detection) — add a `domain == null` canary metric and consider event-driven UIA. Firefox UIA is the weakest browser and needs empirical validation in Phase 3.
5. Decisions challenged with specific reasoning: the "aggregations group by UTC" rule (misattributes work crossing a local-day boundary and complicates multi-tz viewing), org-key-in-MSI (enables rogue-device enrollment by a standard user), and buffer-cap "drop oldest" (silent data loss in a system whose purpose is honesty).

---

## Critical

None. Nothing blocks starting Phase 0. The findings below should be resolved before the phases that own them.

---

## High

### H1 — `/ingest` is not specified as atomic per batch → silent data loss on partial insert + retry
**Where:** `docs/API-CONTRACT.md §2` ("Idempotent on `batchId`"), `docs/ARCHITECTURE.md §3` step 5, `atlas-console/.agents/skills/atlas-api-contract/SKILL.md §3`.

**Problem:** The contract guarantees idempotency on `batchId` and the agent purges only after a 200 with an `accepted` count. But the contract does not require the server's insert to be **atomic per batch**. Consider: the server inserts 20 of 37 intervals, then the connection drops before the agent receives the 200. The agent keeps the rows and retries with the same `batchId`. Now two outcomes are both wrong:
- If the `batchId` receipt is written *before* inserting intervals (pending receipt), the retry sees a known `batchId` and no-ops → the 17 missing intervals are **never stored**, and the agent purges believing all 37 were accepted.
- If the `batchId` receipt is written *after* inserting, the partial insert left no receipt, so the retry re-inserts all 37 — but there are no per-interval client ids, so the 20 already stored are **duplicated**.

Either way, "ACK before purge" is not actually safe without atomicity.

**Recommendation:** Mandate in the contract that `/ingest` applies the whole batch **transactionally** (MongoDB transaction on a replica set): create a `batches` receipt doc for `batchId`, insert all intervals, commit — all atomic. A retry then sees the committed `batchId` and returns `duplicate: true` with `accepted` = the original count. Add an integration test for partial-insert-then-retry (the Phase 2 "deduped on retry" test should specifically cover this, not just a clean retry). This also fixes H2.

---

### H2 — No batch-receipt collection independent of interval lifecycle → stale retry resurrects admin-deleted data
**Where:** `docs/ARCHITECTURE.md §7` (collections list has no `batches` collection), `docs/API-CONTRACT.md §2`, `docs/DECISIONS.md #21` (manual per-PC archive/delete).

**Problem:** Dedup is described as "unique index involving `batchId`" on intervals. If an admin archives/deletes a device's intervals (Decision #21) while that device is offline with an un-ACKed batch still in its buffer, the device later comes online and replays the batch. If the dedup key lives *on the intervals* (which were just deleted) or in a receipt that was deleted with them, the `batchId` is now unknown → the server re-inserts → **deleted data silently returns**. This directly undermines an admin action on a system whose retention model is "manual per-PC archive/delete."

**Recommendation:** Add a **`batches` collection** (`{ _id: batchId, deviceId, accepted, receivedAt, intervalIds }`) with a unique index on `batchId`, whose retention is **independent** of the `intervals` collection (long-lived; TTL or manual purge only). Ingest checks `batches` first; interval archive/delete never touches `batches`. Then a stale replay always no-ops. Combined with H1's transaction, this makes both dedup and admin deletion robust.

---

### H3 — The dashboard has no admin authentication anywhere in the plan
**Where:** `docs/PLAN.md` (no phase item), `docs/ARCHITECTURE.md §8`, `atlas-console/AGENTS.md` (covers agent-token auth only), `docs/DECISIONS.md` "Open/deferred" (consent/AUP listed, but not dashboard access control).

**Problem:** The API routes the *agents* call have auth (org key / Bearer device token). The *dashboard UI for admins* has no auth specified at all. A Next.js app on a VPS behind Cloudflare that displays per-employee activity timelines, app/domain breakdowns, and online/offline status, with no login, is a serious privacy and security exposure — anyone who reaches the host can see the monitoring data.

**Recommendation:** Add admin authentication as an explicit checklist item — ideally in **Phase 0** (so the thin slice is never deployed "open") or at latest in Phase 4. Minimum: HTTP basic auth + Cloudflare Access in front for v1; better: NextAuth/Credentials with a single admin account (or a small admin list) and session cookies. Add it to `docs/PLAN.md` Phase 0 or Phase 4, and to the console skill's "Endpoints" section as a non-agent route concern.

---

## Medium

### M1 — The helper launch mechanism is never specified (the single trickiest Windows-interop piece)
**Where:** `docs/ARCHITECTURE.md §2`, `docs/PLAN.md Phase 1` ("User-session helper process; service launches + supervises it"), `atlas-agent/AGENTS.md "Shape"`, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md §2`.

**Problem:** A Session-0 service cannot `CreateProcess` into the interactive desktop directly. The standard mechanism is `WTSGetActiveConsoleSessionId` / `WTSEnumerateSessions` → `WTSQueryUserToken` → `CreateProcessAsUser` with the user's token, which requires the service to run as **LocalSystem** (or an account with `SE_TCB_NAME`). The docs say "the service launches the helper into the active user session" but never name this mechanism or its privilege requirement. If IT installs the service under a normal service account, the helper silently never launches and no data is captured — while the service still heartbeats (looks "online").

**Recommendation:** Specify in `docs/ARCHITECTURE.md §2` and the agent skill: service runs as LocalSystem; helper launch via `WTSQueryUserToken` + `CreateProcessAsUser`; the service must enumerate sessions and launch **one helper per active session** (handles fast user switching and RDP). Add a Phase 1 checklist item: "verify helper launches into a real user session; service runs as LocalSystem in prod." This is the highest-risk Phase 1 item and should be called out as such.

### M2 — Heartbeat reports service health but not capture health (online ≠ capturing)
**Where:** `docs/API-CONTRACT.md §3` (`health: {cpuPct, memMb}`), `docs/ARCHITECTURE.md §6` ("the heartbeat is the backstop — an agent that goes silent is itself a flag").

**Problem:** The heartbeat proves the *service* is alive. It does not prove the *helper* is alive and producing samples. A device whose helper has died (or is being killed in a loop by a determined standard user) will still heartbeat "online" with `queuedRows` frozen — gappy data with no explicit flag. The "silent agent = red flag" backstop only fires if someone notices the gap.

**Recommendation:** Add a `capture` block to the heartbeat body: `{ helperRunning: bool, lastSampleUtc, intervalsSinceLastHeartbeat, helperRestartCount }`. Surface `helperRunning: false` or `lastSampleUtc` stale in the dashboard heartbeat panel. This closes the gap between "service online" and "actually capturing."

### M3 — Sleep/resume and clock-jump handling must be designed into the Phase 1 aggregator, not deferred to Phase 7
**Where:** `docs/PLAN.md Phase 1` (state machine + aggregator) vs `Phase 7` (sleep/hibernate/resume; clock changes; timezone travel), `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md §3`.

**Problem:** Sleep/resume and clock jumps interact with the *core* interval logic, not with edge-case polish. When a machine sleeps, `GetLastInputInfo` does not advance but the wall clock does; on resume the idle delta can be hours, and an open interval's `endUtc`/`durationSec` can be wrong. A backward clock jump during an open interval can produce negative/zero duration or overlapping intervals. If the Phase 1 aggregator is built without hooks for these, Phase 7 will require reworking it.

**Recommendation:** In Phase 1, design the aggregator to: (a) compute open-interval `durationSec` from a **monotonic clock** (`Stopwatch`) while stamping `startUtc`/`endUtc` from wall-clock UTC; (b) subscribe to power events (`WM_POWERBROADCAST` / `Microsoft.Win32.SystemEvents.PowerModeChanged`) to **close the pre-sleep interval at sleep time and start fresh on resume**; (c) detect wall-clock jumps > N seconds and close+reopen the interval. Full edge-case *verification* can stay in Phase 7, but the hooks must exist from Phase 1.

### M4 — Daily aggregation groups by UTC day, which misattributes work crossing a local-day boundary
**Where:** `atlas-console/.agents/skills/atlas-api-contract/SKILL.md §6` ("Aggregations group by UTC; the client converts for display"), `docs/PLAN.md Phase 4` ("All times rendered in the viewer's local zone (stored UTC)"), `docs/DECISIONS.md #22`.

**Problem:** Storing UTC is correct. But **grouping daily summaries by UTC day** is wrong for a Bangladesh (UTC+6) team at the day boundaries. UTC midnight = 06:00 local. A standard 09:00–18:00 local shift maps to 03:00–12:00 UTC and happens to fit one UTC day, so day shift is fine — but **night shifts, late overtime past local midnight, and early starters before 06:00 local get split across two UTC-day buckets**, and a manager in a different timezone viewing a Bangladesh employee sees the split too. The skill's rule "aggregations group by UTC" encodes this bug.

**Recommendation:** Challenge Decision #22's application: keep storing UTC, but **group per-employee daily/weekly summaries by the employee's local date** (convert `startUtc` → employee tz → group by local `YYYY-MM-DD`). This requires storing a tz on the employee/device (or deriving from the agent's local tz at capture and sending it up). Use the **viewer's** tz only for timeline *rendering*, not for per-employee *day bucketing*. Add an explicit Phase 4 checklist item and a unit test: "a workday crossing local midnight appears as one local day, not two." Update the console skill §6 accordingly.

### M5 — Re-enroll does not specify whether the old device token is revoked/superseded
**Where:** `docs/API-CONTRACT.md §1` ("If `deviceId` is already enrolled, the server may return the existing record and (re)issue a token"), `atlas-console/.agents/skills/atlas-api-contract/SKILL.md §2`/`§5`.

**Problem:** Enrollment is idempotent on `deviceId` and may reissue a token. The contract does not say what happens to the *previous* token. If old tokens are not revoked, a re-enroll accumulates valid tokens; if they are revoked, a re-enroll from one context invalidates another. The 401→re-enroll flow (`§5 Conventions`) depends on this being well-defined.

**Recommendation:** State explicitly: re-enroll **revokes (supersedes)** all prior non-revoked tokens for that `deviceId` and issues exactly one new one. Record `supersededAt` on the old `deviceTokens` rows. Document this in the contract §1 Behavior and the skill §5.

### M6 — Mutable device attributes (hostname, osUsername, pcName) have no update channel after enrollment
**Where:** `docs/API-CONTRACT.md §1` (sent at enroll) vs `§3` (heartbeat body has no attributes), `docs/ARCHITECTURE.md §5` ("mutable attributes").

**Problem:** `hostname`, `osUsername`, and `mac` can change (user renamed, PC renamed, NIC swap). They are sent only at `/enroll`. `agentVersion` is in the heartbeat, but the other mutable attrs are not, so the server's view goes stale until a re-enroll. `pcName` is "changeable from the dashboard" (admin-side), but `hostname`/`osUsername` have no update path.

**Recommendation:** Add an optional `attributes: { pcName?, hostname?, osUsername?, mac? }` block to the heartbeat body (agent sends only when changed), or add a lightweight `PATCH /api/v1/device`. Server updates the mutable fields on the `devices` doc; identity (DeviceId) is unchanged.

### M7 — `413 batch too large` has no split/batchId strategy defined
**Where:** `docs/API-CONTRACT.md §2` ("413 — agent should split"), `§5` ("Payload size").

**Problem:** If the agent splits a too-large batch, each sub-batch needs a **different** `batchId` (else the second half is deduped away as a "duplicate" of the first). The contract doesn't specify how to derive sub-batch ids, so an implementer could easily get this wrong and lose half the data.

**Recommendation:** Specify: on 413, the agent splits into sub-batches with deterministic, distinct ids, e.g. `b-<utc>-<seq>#a`, `#b`, …, and retries each independently. Also define a hard cap (e.g. ≤ 5000 intervals or ≤ 5 MB decompressed) so agent and server agree on the split threshold.

### M8 — Token-to-deviceId binding is not enforced at `/ingest`/`/heartbeat`
**Where:** `docs/API-CONTRACT.md §2`/`§3` (Bearer token + `deviceId` in body), `atlas-console/.agents/skills/atlas-api-contract/SKILL.md §2`.

**Problem:** The body carries `deviceId` and the header carries a device token. Nothing in the contract says the server must verify the token was issued **for that `deviceId`**. Without enforcement, a token from device A could submit intervals under device B's `deviceId` (or under a nonexistent one), polluting attribution.

**Recommendation:** Bind token→deviceId at enroll; at `/ingest` and `/heartbeat`, reject with 401/400 if the body `deviceId` does not match the token's bound `deviceId`. State this in the contract Behavior and the skill §2/§5.

### M9 — Buffer-cap "drop oldest" is silent data loss in a system whose goal is honesty
**Where:** `docs/AGENTS.md §4` ("Cap ~2–3 months"), `docs/DECISIONS.md #10`, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md §4` ("drop oldest only as a last resort").

**Problem:** "Drop oldest" discards **unsynced** rows when over cap — the most valuable rows (the offline period). SQLite handles tens-to-hundreds of MB easily, so the cap is more a safety net than a need, but the behavior at cap is under-specified and risks silent loss.

**Recommendation:** (a) Raise the effective cap (SQLite is fine well beyond the ~90 MB worst case). (b) If unsynced rows *must* be dropped, **never silently**: emit a log, set a `bufferNearCap` / `droppedRows` flag in the heartbeat, and prefer dropping **idle** intervals (least valuable) over active/passive_media. (c) Always drop *synced* rows aggressively first (already the plan). Challenge Decision #10's "drop oldest" wording accordingly.

### M10 — `batchId` sequence must persist across agent restarts
**Where:** `docs/API-CONTRACT.md §2` (format `b-2026-06-22T09:10:00Z-001`), `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md §4`.

**Problem:** If the sequence suffix (`001`) is in-memory and resets on restart, a restart that lands on the same UTC minute stamp can reuse a `batchId`, causing the server to dedupe a genuinely new batch → data loss.

**Recommendation:** Persist the batch sequence (or a monotonic batch counter) in SQLite so `batchId` is never reused across restarts. State this requirement in the agent skill §4.

### M11 — Fast user switching / multiple sessions: attribution and per-session helpers
**Where:** `docs/ARCHITECTURE.md §5` ("one dedicated desktop per employee, one Windows user per machine"), `docs/PLAN.md Phase 7` ("fast user switching; RDP/remote sessions"), `docs/DECISIONS.md "Open/deferred"` (shared-login attribution).

**Problem:** The identity model is one DeviceId per machine, and employee→device is 1:1 (`employees.deviceIds: []`). Fast user switching means two *different* users can be logged in simultaneously on one box, each with their own session. The service must run a helper per session (M1), and intervals carry `osUsername` — but both users' activity lands under one DeviceId, which maps to one employee. The "one user per machine" assumption is stated but fast-user-switching is also listed as a Phase 7 edge case, so the boundary is unclear.

**Recommendation:** Clarify in `docs/ARCHITECTURE.md §5`: v1 supports one active employee per machine; if two sessions are active, the agent attributes by `osUsername` and the dashboard should flag a device whose intervals show multiple distinct `osUsername`s in a short window (so a mis-mapped machine is visible). Decide explicitly whether to crash, flag, or "attribute to mapped employee" — do not leave it undefined.

### M12 — Silent-running verification is first a Phase 5 item, but the helper arrives in Phase 1
**Where:** `docs/PLAN.md Phase 1` (helper introduced) vs `Phase 5` ("Confirm zero UI footprint"), `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md §1`.

**Problem:** The helper process is built in Phase 1. If it (or its dev console host) pops a window, that is a silent-running violation that won't be formally checked until Phase 5 — four phases of UI-noise regressions can accumulate.

**Recommendation:** Add a Phase 1 checklist item: "helper runs **windowless** in prod config (no console window when launched by the service); verify on a non-admin account." Keep the full hardening audit in Phase 5, but make "no UI" a continuous invariant from Phase 1.

### M13 — No DB backup / disaster-recovery runbook
**Where:** `docs/PLAN.md` (no item), `docs/ARCHITECTURE.md §8` (single VPS + MongoDB), `docs/DECISIONS.md #21` (retention indefinite).

**Problem:** Retention is "indefinite" and the data is the entire point of the system, yet backups are nowhere in the plan. A single VPS MongoDB with no backup is a single point of failure.

**Recommendation:** Add a checklist item (Phase 6 or a runbook): daily MongoDB backup (mongodump to R2 or a scheduled snapshot), with a documented restore test. Cheap insurance for irreplaceable historical data.

### M14 — No admin audit log
**Where:** `docs/ARCHITECTURE.md §7` (no `auditLog` collection), `docs/PLAN.md Phase 4` (category rules, device→employee mapping, delete/archive).

**Problem:** Admins can delete/archive a device's history, edit category rules (which retroactively change productive/unproductive splits), and remap devices to employees. For a monitoring system with privacy sensitivity, there is no record of who did what, when. This matters for both trust and legal defensibility (the "Open/deferred" AUP item).

**Recommendation:** Add an `auditLog` collection (`{ adminId, action, target, before, after, atUtc }`) and record every destructive/rules-changing admin action. Light to implement; valuable for accountability. Can land in Phase 4 or Phase 6.

### M15 — Org key in the MSI enables rogue-device enrollment by a standard user
**Where:** `docs/AGENTS.md §4` (enrollment), `docs/ARCHITECTURE.md §6` ("org key … grants nothing except the ability to enroll"), `docs/DECISIONS.md #15`.

**Problem:** The org key is embedded in the MSI that lives on ~100 employee machines. The threat model is "standard users only," and a standard user can often extract a static secret from an installed binary's files/resources. With the org key, a standard user can enroll a **rogue device** and submit fabricated activity (inflate their own numbers or pollute the dataset). The org key is fleet-wide, so one extraction compromises enrollment for everyone until rotation.

**Recommendation:** Keep the org-key approach (it's pragmatic) but: (a) document explicitly that org-key **rotation** invalidates only *new* enrollments (existing device tokens stay valid), and publish the rotation procedure (Phase 7 already lists it — make sure it covers this); (b) consider enrolling only from the corporate network/VPN (reject `/enroll` from non-corp IPs) as a cheap extra gate; (c) surface "newly enrolled devices" in the dashboard so a rogue enrollment is visible and an admin can revoke it. Acknowledging this risk in `docs/ARCHITECTURE.md §6` is the minimum.

### M16 — Dashboard aggregation performance over time (pre-aggregation)
**Where:** `docs/ARCHITECTURE.md §7`/`§8`, `docs/PLAN.md Phase 4`.

**Problem:** Interval-per-document at ~100–300/day/device × 100 devices × ~250 workdays/yr ≈ 2.5–7.5M docs/yr. Indexed `{deviceId,startUtc}` and `{employeeId,startUtc}` make per-employee timelines fine, but **company-wide weekly summaries** with on-the-fly aggregation pipelines will scan increasingly large ranges and slow down over months.

**Recommendation:** Not wrong for v1, but plan for it: add a **`dailySummary`** collection (per device/employee/day, written at ingest or via a nightly job) and drive summary views from it. Defer the build until queries get slow, but design the ingest path so adding it later is a drop-in. Note in Phase 4.

---

## Low

### L1 — DPAPI machine scope vs ACL: clarify which layer stops a standard user
**Where:** `docs/AGENTS.md §4`, `docs/ARCHITECTURE.md §6`, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md §5`.

DPAPI **machine scope** is decryptable by any process running as the machine account / LocalSystem on that box; it binds the blob to the machine (protecting against offline disk access) but does **not** by itself stop a standard user process from decrypting it if that process can read the file. The **ACL on the ProgramData folder** is what stops a standard user. The docs imply DPAPI is the protection. Make explicit that the ACL is the primary control and DPAPI is defense-in-depth + offline-disk protection.

### L2 — SQLite encryption library choice
**Where:** `atlas-agent/AGENTS.md "Tech choices"`, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md §4`.

"Encrypted SQLite" in .NET is not native to `Microsoft.Data.Sqlite`. You need **SQLCipher** (`SQLitePCLRaw.bundle_e_sqlcipher`) or the paid Microsoft encryption. Flag the dependency choice early so Phase 2 doesn't stall on it.

### L3 — Helper-kill DoS by a determined standard user → gappy data
**Where:** `docs/ARCHITECTURE.md §6` (defend vs standard users; watchdog).

The helper runs in the user's session, so a standard user can kill it (Task Manager / `taskkill`). The service watchdog restarts it, but a kill-loop makes data gappy while the service stays "online." M2's `helperRestartCount` surfaces this; consider capping restart attempts and escalating to a flagged "tamper suspected" state. Low because it requires a deliberately hostile user.

### L4 — Confirm the update-signing public key is embedded in the agent, not fetched
**Where:** `docs/API-CONTRACT.md §4` ("verified against the embedded/internal public key"), `docs/ARCHITECTURE.md §9`.

The wording is mostly clear ("embedded/internal"), but make explicit that the public key is **compiled into the agent binary** and never fetched over the network — otherwise the signature check is circular. Confirm in Phase 6.

### L5 — Page titles can carry sensitive content
**Where:** `docs/DECISIONS.md #2` (domain + page title), `docs/AGENTS.md §3`.

A webmail tab's title may contain an email subject; a cloud-doc tab's title may contain a private document name. Storing page titles is a deliberate, defensible choice (Decision #2), but flag that titles can leak sensitive strings into the DB — relevant to the AUP/consent item and to who can view the dashboard (H3).

### L6 — `config` shape is inconsistent between `/enroll` and `/heartbeat`
**Where:** `docs/API-CONTRACT.md §1` vs `§3`.

`/enroll` returns `config` with `bufferCapDays`; the `/heartbeat` example omits it. Define one canonical `config` object returned identically from both (fields may be omitted when unchanged, but the shape should be the same).

### L7 — `accepted` count + purge semantics on a duplicate retry should be explicit
**Where:** `docs/API-CONTRACT.md §2`.

State plainly: the agent purges the batch after **any 200 whose `batchId` matches**, whether `duplicate: true` or not, and `accepted` on a duplicate equals the originally-accepted count. Removes ambiguity for implementers.

### L8 — Honor `Retry-After` on 429
**Where:** `docs/API-CONTRACT.md §5` (429 → back off).

Specify that the agent honors a `Retry-After` header when present on 429/503, else uses exponential backoff. Standard but worth stating.

### L9 — Echo `contractVersion` in responses
**Where:** `docs/API-CONTRACT.md §5` ("Contract version is echoed by the server").

The conventions say the version "is echoed by the server" but no example shows it. Add a `contractVersion` field (or response header) to every response so the agent can detect mismatch programmatically.

### L10 — `/api/v1/update/manifest` vs the heartbeat `update` block: clarify redundancy
**Where:** `docs/API-CONTRACT.md §3` vs `§4`, `docs/ARCHITECTURE.md §9`.

The heartbeat already carries the `update` block (version, manifestUrl, mandatory). The standalone `GET /update/manifest` is also listed but its role is unclear (proxy? redirect? legacy?). Pick one channel (recommend: heartbeat `update` for "is there a new version", `GET /update/manifest` / R2 for the full manifest) and document the relationship.

### L11 — `minSupportedVersion` enforcement is undefined
**Where:** `docs/API-CONTRACT.md §4`.

If an agent is below `minSupportedVersion` and the update fails, what should the server do? Recommend: the server still **accepts ingest** (never lose data) but marks the device `agentVersionOutdated: true` in the dashboard so IT can intervene. Don't define a hard shutoff.

### L12 — DeviceId redundancy reconciliation rule is unspecified
**Where:** `docs/ARCHITECTURE.md §5` (stored redundantly: file + registry), `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md §5`.

Define: if either store survives, use it and restore the other; if both survive and disagree, trust the file (ACL-protected) and log an alert; if neither survives, the machine is "reformatted" → re-enroll as a new device.

### L13 — Slight over-indexing on `{domain}` and `{category}` standalone
**Where:** `docs/ARCHITECTURE.md §7`, `atlas-console/.agents/skills/atlas-api-contract/SKILL.md §7`.

Standalone `{domain}` / `{category}` indexes are only useful for global rule/coverage queries; the hot queries are compound with `deviceId`/`employeeId` + `startUtc`. Keep them only if you have a real global-filter query; otherwise drop to reduce write overhead. Minor.

### L14 — Open-interval duration should use a monotonic clock (also see M3)
**Where:** `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md §3`.

Compute the *open* interval's elapsed time from a monotonic source and stamp `startUtc`/`endUtc` from wall-clock UTC; if wall-clock jumps backward, prefer the monotonic duration and clamp. Prevents negative-duration intervals.

### L15 — Helper→service pipe break can lose or duplicate the open interval
**Where:** `atlas-agent/AGENTS.md "Shape"` (named pipe), `docs/ARCHITECTURE.md §3`.

If the pipe drops mid-flush of a closed interval, the service may store a partial interval or the helper may resend on reconnect and create a near-duplicate. Consider a client-generated **interval id** (e.g., `deviceId + startUtc + state` hash, or a monotonic seq) so the service can dedupe helper resends. Low because it affects at most one interval per pipe break.

### L16 — Disk-full on the buffer is unhandled
**Where:** `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md §4`.

If the disk fills, SQLite writes fail → data loss with no signal. Log + set a `diskFull`/`bufferWriteError` flag in the heartbeat so the dashboard shows it.

### L17 — Fullscreen / F11 browser: address bar unreadable → `domain: null`
**Where:** `docs/PLAN.md Phase 3`/`Phase 7`, `docs/ARCHITECTURE.md §3`.

In fullscreen the address bar isn't exposed via UIA. Decide: keep last-known domain with a stale flag, or mark `domain: null` (current behavior). Document as a known limit in the Phase 3 README.

### L18 — `dotnet-best-practices` skill says MSTest; project standard is xUnit
**Where:** `atlas-agent/.agents/skills/dotnet-best-practices/SKILL.md "Testing Standards"` vs `atlas-agent/AGENTS.md "Testing policy"` / `docs/DECISIONS.md #26`.

The community skill recommends **MSTest + FluentAssertions + Moq**, but the locked project standard is **xUnit**. An AI agent loading the skill could be misled. Add a one-line override at the top of the skill ("testing framework: xUnit, per project standard") or update the skill.

### L19 — `machineHints` collected but unused in v1
**Where:** `docs/API-CONTRACT.md §1` ("informational only in v1 (no dedup)"), `docs/DECISIONS.md #14`.

Collecting `machineGuid`/`mac` you explicitly don't use is mild over-engineering, but it's cheap and future-proof for hardware dedup. Fine to keep; just don't build logic on it in v1.

---

## Nice-to-have

### N1 — Event-driven UIA for the address bar instead of 3–5 s polling
**Where:** `docs/AGENTS.md §4` (sample every 3–5 s), `docs/PLAN.md Phase 3`.

Subscribe to the address-bar control's `ValuePattern.PropertyValue` **property-changed event** (`UIAutomation.AddPropertyChangedEventHandler`) to get URL changes pushed, with a polling fallback. Lower CPU, lower latency, and catches fast navigations that 3–5 s polling misses. Not required, but a material efficiency win for a "must not make the machine feel slow" agent.

### N2 — Event-driven GSMTC instead of polling
**Where:** `docs/AGENTS.md §3` (GSMTC), `docs/PLAN.md Phase 1`.

`GlobalSystemMediaTransportControlsSessionManager` exposes `CurrentSessionChanged` and `PlaybackInfoChanged` events. Subscribe rather than poll every 3–5 s. Same efficiency rationale as N1.

### N3 — Session-change notifications for helper launch instead of polling the active session
**Where:** `docs/ARCHITECTURE.md §2`, M1.

Use `WTSRegisterSessionNotification` (in the service) to receive `WTS_SESSION_LOGON`/`LOGOFF` events and launch/tear down helpers reactively, rather than polling `WTSGetActiveConsoleSessionId`. Cleaner and more responsive to fast user switching.

---

## Where the design is solid (do not change)

- **Contract-first, doc-first discipline** (`AGENTS.md §5`, `API-CONTRACT.md` header): making the wire format authoritative in one place and requiring both repos to mirror it is exactly right for a two-repo system built by different AI agents.
- **Phased plan with exit gates + manual verification + review checkpoint** (`PLAN.md`): the right process for AI-built code; Phase 0 as a thin vertical slice (enroll→heartbeat→dashboard online) is the correct first cut.
- **Two-part Service + Helper** (`ARCHITECTURE.md §2`): the standard, correct pattern for a Session-0 service that needs desktop/input access; putting all secrets/networking in the service and none in the helper is clean.
- **Interval aggregation / Model B** (`DECISIONS.md #5`): flushing only on app/domain/state change reduces ~17k raw samples/day to ~100–300 intervals/day, makes "time spent" a trivial query, and keeps payloads tiny. Excellent tradeoff.
- **3-state model (active / passive_media / idle)** (`DECISIONS.md #7`, `ARCHITECTURE.md §4`): more honest than binary active/idle; separating "watching media" from "away" is genuinely more informative. Idle-splitting keeps active time honest.
- **ACK-before-purge + `batchId` idempotency + exponential backoff** (`DECISIONS.md #9`): the correct reliability pattern — *provided* H1/H2's atomicity/receipt fix is applied.
- **Per-device DPAPI tokens, hashed server-side, revocable individually** (`ARCHITECTURE.md §6`, `DECISIONS.md #15`): clean identity/secrets model; per-device tokens make single-machine revocation possible without re-keying the fleet.
- **Opaque DeviceId GUID; mutable attributes never identity; reformat = new device** (`ARCHITECTURE.md §5`, `DECISIONS.md #13`/`#14`): simple and avoids the classic rename-forks-history bug.
- **Self-contained single-file .exe** (`DECISIONS.md #19`): correct for "target PCs need nothing pre-installed."
- **SQLite local buffer vs MongoDB-on-every-desktop** (`DECISIONS.md #8`/`#12`): right call — embedded, crash-safe, invisible; Mongo belongs on the server.
- **Testing split: unit-test pure logic, manually verify OS-interop** (`DECISIONS.md #26`, agent skill §7): pragmatic and correct; keeping OS-free logic in `AtlasAgent.Core` is good testability hygiene.
- **URL granularity = domain + page title, not full path** (`DECISIONS.md #2`): the right privacy/legality/noise balance for "working vs slacking" without messy partial paths.
- **Load estimate (~0.3 req/s at 100 agents) → one Next.js app for API + UI, no separate ingest service** (`ARCHITECTURE.md §8`, `DECISIONS.md #11`): correctly avoids over-engineering; trivial load.
- **R2 for update hosting** (`DECISIONS.md #20`): zero egress, CDN-fast, decoupled from the VPS, binary stays private. Good fit.
- **Decision log + single-source-of-truth `AGENTS.md`** (`DECISIONS.md #23`/`#24`): strong project hygiene for an AI-agent-built system.

---

## Suggested resolution order (by phase)

- **Before Phase 0:** H3 (add admin auth as a Phase 0/4 item), L18 (skill MSTest→xUnit note).
- **Phase 1:** M1 (helper launch mechanism + LocalSystem + per-session), M2 (capture health in heartbeat — contract change, do early), M3 (aggregator hooks for sleep/clock jumps), M12 (windowless helper check), L14 (monotonic duration).
- **Phase 2 (contract + data):** H1 (atomic ingest), H2 (`batches` receipt collection), M5 (re-enroll revokes old token), M6 (mutable attrs in heartbeat), M7 (413 split strategy), M8 (token↔deviceId enforcement), M10 (persistent batch seq), L6/L7/L8/L9 (contract cleanups), L2 (SQLCipher choice).
- **Phase 3:** URL canary metric (M-context), Firefox UIA empirical validation, L17 (fullscreen), N1/N2 (event-driven UIA/GSMTC) as optional.
- **Phase 4:** M4 (tz-aware daily aggregation) + test, M14 (audit log), M16 (pre-agg plan), admin auth fully built.
- **Phase 6:** M13 (DB backups), L4 (embedded pubkey confirm), L10/L11 (manifest channel + minSupportedVersion).
- **Phase 7:** M11 (fast user switching clarification), M15 (org-key rotation + corp-IP enroll gate), M9 (buffer-cap behavior), L1/L3/L15/L16 (defense-in-depth cleanups), N3 (session notifications).

---

*End of review. No code or existing documents were modified; only this file was created.*
