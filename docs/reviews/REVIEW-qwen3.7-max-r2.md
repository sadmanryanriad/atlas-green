# Atlas Green — Iteration 2 Design Review

**Model:** qwen3.7-max
**Date:** 2026-06-23
**Round:** 2 (stress-testing v1.1 revisions against iteration-1 findings)

## Executive Summary

v1.1 absorbed the iteration-1 consensus well: UUID `batchId`, atomic ingest via single-node replica set, `batches` receipt collection, dashboard auth, helper launch mechanics, monotonic-clock durations, idle hysteresis, URL fallback chain, capture health, rate limits, and the redistributed Phase 7 are all correct fixes. However, two High-severity specification contradictions were **introduced** by the revisions: a `{deviceId, batchId} unique` index on `intervals` that would reject all but the first interval in every batch, and an `employeeId` "never back-filled" rule that directly conflicts with Phase 4's "re-resolves affected intervals" requirement. Five Medium findings concern under-specified state-machine interactions (sleep-resume vs media, hysteresis scope across non-input transitions, timestamp/duration precedence), a GPO/Active Directory assumption, and Phase 1 overload. Nothing requires an architectural change; all findings are resolvable with specification edits before their owning phases.

---

## High

### H1 — `{deviceId, batchId} unique` index on `intervals` prevents multi-interval batches
**File:** `docs/ARCHITECTURE.md` §7, `intervals` collection indexes
**Problem:** The intervals index list includes `{ deviceId, batchId } unique`. A batch contains **multiple** intervals that all share the same `batchId` (that is the whole point of a batch). A unique compound index on `{deviceId, batchId}` allows only **one** interval per device per batchId. The second interval in every batch will violate the unique constraint, the transaction will roll back, and every `/ingest` call with more than one interval will fail. This is a show-stopper for Phase 2 implementation.

The dedup belongs at the **batch** level (the `batches` collection, which already has `_id: batchId` and a unique constraint on `{deviceId, batchId}`). The `intervals` collection should carry a **non-unique** index on `{batchId}` (useful for admin-delete-by-batch and debugging) but no uniqueness constraint involving `batchId`.

**Recommendation:** Remove `unique` from the `{deviceId, batchId}` index on `intervals`. Change it to `{ batchId }` (non-unique). The batch-level dedup in the `batches` collection (which is correctly specified) handles idempotency. Add an integration test: "a batch of 37 intervals is stored; a retry with the same `batchId` returns `duplicate: true` without re-inserting any of the 37."

### H2 — `employeeId` "never back-filled" contradicts Phase 4 "re-resolves affected intervals"
**File:** `docs/ARCHITECTURE.md` §7 vs `docs/PLAN.md` Phase 4
**Problem:** ARCHITECTURE.md §7 states: "`employeeId` and `osUsername` are **snapshots at ingest/capture** — never back-filled." PLAN.md Phase 4 requires: "device→employee remap writes an `auditLog` entry and **re-resolves affected intervals**." These are directly contradictory. If `employeeId` is never back-filled, a remap only affects future intervals — but Phase 4 says it re-resolves historical ones.

The `osUsername` "never back-filled" rule is correct (it's a capture-time fact). The `employeeId` rule should be different: it's an **administrative mapping**, not a capture-time observation, and admins need to correct misattributed history.

**Recommendation:** Split the rule in ARCHITECTURE.md §7:
- `osUsername`: snapshot at capture, never back-filled.
- `employeeId`: snapshot at ingest from the current device→employee mapping; **re-resolvable** by admin action (device→employee remap updates historical intervals within an admin-specified range, audit-logged). Remove "never back-filled" from `employeeId`.

---

## Medium

### M1 — Post-resume state contradiction: "new idle interval" vs media still playing
**File:** `docs/ARCHITECTURE.md` §3 step 3, `docs/DECISIONS.md` #33
**Problem:** Decision #33 says the open interval is closed on sleep/suspend and "a new `idle` interval starts on resume." But if the user was watching YouTube (`passive_media`) before sleep, the browser tab is still playing on resume. GSMTC still reports Playing. The state machine evaluates on the next sample: no input + media playing → `passive_media`. The "new idle interval" is immediately transitioned to `passive_media`, producing a 3–5 s `idle` sliver that the sub-5 s merge rule (#34) absorbs.

The behavior is ultimately correct, but the specification says the new interval **is** `idle` when it should say the new interval **starts as** `idle` and is immediately re-evaluated by the state machine. An implementer reading the current text might hardcode the post-resume state as `idle` and skip the state-machine evaluation for the first sample.

**Recommendation:** Reword ARCHITECTURE.md §3 step 3: "On sleep/suspend the open interval is closed at the suspend time; on resume, a new interval opens and the state machine evaluates immediately (typically `idle` if no media, `passive_media` if media is still playing)."

### M2 — Hysteresis scope is unclear: does it govern `idle → passive_media`?
**File:** `docs/DECISIONS.md` #34, `docs/ARCHITECTURE.md` §4
**Problem:** Decision #34 says: "enter `idle` at 300 s no-input, **leave only after ~30 s sustained input**; sub-~5 s intervals are merged." The exit condition is input-centric ("30 s sustained input"). But the state machine has two ways to leave `idle`: (a) input arrives → `active`, (b) media starts playing → `passive_media`. The hysteresis text only addresses (a). If (b) is also gated by the 30 s requirement, a user who starts watching a video while in `idle` stays `idle` for 30 s despite media playing — producing a confusing 30 s `idle` sliver before `passive_media`.

The natural reading is that hysteresis applies only to the input-driven `idle ↔ active` boundary, and media-driven transitions (`idle → passive_media`, `passive_media → idle`) are immediate. But this is not stated.

**Recommendation:** Add to Decision #34 or ARCHITECTURE.md §4: "Hysteresis applies only to `idle ↔ active` transitions (input-driven). Transitions involving `passive_media` are immediate on the next sample."

### M3 — `startUtc`, `endUtc`, `durationSec` have no stated precedence after clock jumps
**File:** `docs/API-CONTRACT.md` §2, `docs/DECISIONS.md` #33
**Problem:** Decision #33 says durations come from a monotonic clock and UTC from the wall clock. After a forward clock jump during an open interval, `endUtc − startUtc` can be much larger than `durationSec`. The contract stores all three fields. A dashboard computing "time spent" from `endUtc − startUtc` gets a wrong (inflated) number; one using `durationSec` gets the correct one. No document states which field is canonical.

**Recommendation:** Add to API-CONTRACT.md §5 (Conventions): "`durationSec` (from the monotonic clock) is the canonical interval duration. `endUtc − startUtc` may differ due to clock skew; consumers must use `durationSec` for time calculations, never derive duration from timestamps."

### M4 — GPO cert pre-trust assumes Active Directory; may not exist
**File:** `docs/DECISIONS.md` #18, `docs/PLAN.md` Phase 5
**Problem:** Decision #18 says "IT pushes the self-signed cert to `LocalMachine\Root` via **Group Policy** on every target PC BEFORE the MSI rollout." GPO requires Active Directory domain-joined machines. A 100-person company in Bangladesh may use workgroup (non-domain) machines. If AD is not available, the GPO step fails silently and the MSI install triggers SmartScreen prompts.

**Recommendation:** Note the AD dependency in Decision #18 and the Phase 5 runbook. Provide an alternative for non-domain machines: a PowerShell one-liner (`Import-Certificate -FilePath cert.cer -CertStoreLocation Cert:\LocalMachine\Root`) that IT runs before or during the MSI install. The MSI itself could install the cert as a custom action (already running as admin).

### M5 — Phase 1 is overloaded; risks becoming a multi-week bottleneck
**File:** `docs/PLAN.md` Phase 1
**Problem:** Phase 1 now contains 14 substantial checklist items: helper launch via `WTSQueryUserToken` + `CreateProcessAsUser`, session-change handling, foreground sampling, foreground hijacker filter, `appName` from `FileDescription`, idle detection with hysteresis, media detection (GSMTC + gated WASAPI), three-state machine, interval aggregator with monotonic clock, sleep/suspend handling, graceful shutdown, named pipe transport, capture health in heartbeat, windowless helper verification, and unit tests for the state machine + aggregator. Several of these are among the hardest Windows-interop problems in the entire project (helper launch, GSMTC/WASAPI, UIA-free foreground sampling).

Phase 1 is the critical path: Phase 2 (buffer + sync) cannot start until Phase 1 produces intervals. A bottleneck here delays everything.

**Recommendation:** Consider splitting Phase 1 into two sub-phases:
- **Phase 1a** (helper + transport): helper launch, session-change handling, named pipe, foreground sampling, `appName`, windowless check. Exit gate: "helper launches correctly, samples reach the service over the pipe, basic intervals form in logs."
- **Phase 1b** (state machine + aggregator): idle with hysteresis, media detection, three-state machine, aggregator, sleep/suspend, graceful shutdown, capture health. Exit gate: the current Phase 1 exit gate.

This allows Phase 2 work (SQLite schema, sync loop skeleton) to begin against Phase 1a's basic intervals while Phase 1b is still in progress.

---

## Low

### L1 — `batches` receipt collection has no retention or archival policy
**File:** `docs/ARCHITECTURE.md` §7, `docs/DECISIONS.md` #28
**Problem:** The `batches` collection is retained independently of `intervals` (correct — prevents stale replay resurrection). But it has no TTL, no archival strategy, and no admin tooling. At 100 agents × ~96 batches/day × 365 days, the collection grows to ~3.5M documents/year (~350 MB). Over 5 years of indefinite retention: ~17.5M documents. Not a storage problem for MongoDB, but unbounded growth with no management mechanism.

**Recommendation:** Add a retention note to ARCHITECTURE.md §7: "Batch receipts for archived/revoked devices may be purged by admin action after N days (e.g., 365). Active device receipts are retained indefinitely." This can be a Phase 4 admin tooling item.

### L2 — Rate limiting + 413 split interaction causes slow sub-batch delivery
**File:** `docs/API-CONTRACT.md` §2, §5
**Problem:** Rate limit on `/ingest` is ≤ 1 req per 30 s. A 413 response triggers a split into sub-batches, each with a fresh UUID. If the original batch splits into 3 sub-batches, the 2nd and 3rd hit the rate limit and must wait for `Retry-After`. A large batch that triggers a 3-way split takes ≥ 60 s to deliver fully. This is correct behavior (rate limits protect the server) but could surprise an implementer who expects sub-batches to be delivered immediately.

**Recommendation:** Add a note to API-CONTRACT.md §2 Behavior: "Sub-batches from a 413 split are subject to the same rate limits as any other ingest call. The agent should space them according to the rate limit, not fire them immediately."

### L3 — Order of auth checks at `/enroll` not specified (org key vs device block)
**File:** `docs/API-CONTRACT.md` §1
**Problem:** `/enroll` can fail with `401` (bad org key) or `403` (blocked device). If a blocked device presents an invalid org key, which error wins? The order matters for security (leaking whether a device is blocked to someone with the wrong org key) and for agent behavior (the agent treats 401 and 403 differently).

**Recommendation:** Specify: org key validation happens first (`401` if invalid), then device-block check (`403` if blocked). This prevents an attacker without the org key from probing device-block status.

### L4 — `orgTimeZone` change has no documented effect on existing data
**File:** `docs/ARCHITECTURE.md` §7 (`settings.orgTimeZone`), `docs/DECISIONS.md` #40
**Problem:** Decision #40 fixes day/week bucketing to the org timezone (`Asia/Dhaka`). If an admin changes `orgTimeZone` (e.g., the company opens a remote office), all existing daily/weekly summaries were bucketed under the old zone. Changing the setting doesn't retroactively re-bucket stored aggregates. This is correct behavior (re-bucketing would be expensive and surprising) but should be documented.

**Recommendation:** Add a note to ARCHITECTURE.md §7 or the Phase 4 dashboard spec: "Changing `orgTimeZone` affects only future bucketing. Historical summaries retain their original bucket boundaries. A timezone change is logged in the audit log."

---

## Where v1.1 Is Already Correct (No Action Needed)

The following pressure-test areas were examined and found sound:

- **Atomic ingest on a single-node replica set**: At ~0.3 req/s with 100 agents, MongoDB transaction overhead on a single-node replica set is negligible (no secondary replication lag, minimal lock contention). The `rs.initiate()` requirement is a one-time operational step. Failure modes (transaction rollback on unique-index violation → `duplicate: true`) are correctly specified. No performance or operational concern at this scale.

- **Helper-as-logged-on-user vs DPAPI/secrets**: Correctly specified. The helper holds no secrets and does no networking. All DPAPI/machine-scope secrets (device token, buffer encryption key) are accessed only by the service (LocalSystem). The named pipe security descriptor (LocalSystem + interactive user) is appropriate for the threat model.

- **`batches` collection size at v1 scale**: ~100 bytes per receipt × ~35k receipts/year/device × 100 devices = ~350 MB/year. Trivially within MongoDB's comfort zone. Not a concern for v1.

- **Cloudflare Access in Phase 0**: Correct to wire auth from day one. The dependency on a Cloudflare account and DNS configuration is an external prerequisite, not a design gap.

- **Enrollment idempotency (no token rotation on re-enroll)**: Correctly specified in API-CONTRACT.md §1. Avoids the buffered-data-era problem that iteration 1 warned about.

- **UUID `batchId`**: Eliminates all collision/restart/sequence-persistence concerns from iteration 1. No residual risk.

- **Config validation with clamping + `configSchemaVersion`**: Forward-compatible and backward-compatible. Correctly handles out-of-range values and unknown fields.

- **Phase 7 redistribution**: The moved items (helper/session/sleep/clock → Phase 1, buffer-cap/revocation/org-key → Phase 2, browser variants/DPI → Phase 3, AV/AppLocker → Phase 5) are in the right phases. The residual Phase 7 (multi-monitor, PWAs/Electron, 100-device soak test, revocation E2E) is appropriately scoped.

---

## Summary by Severity

| Severity | Count | Key themes |
|----------|-------|------------|
| High | 2 | Unique index on intervals blocks multi-interval batches; employeeId back-fill contradiction |
| Medium | 5 | Sleep-resume state, hysteresis scope, timestamp/duration precedence, GPO/AD assumption, Phase 1 overload |
| Low | 4 | Batch receipt retention, rate-limit + 413 interaction, enroll auth-check order, timezone change docs |

The two High findings are specification errors that would cause Phase 2 implementation failure if not corrected. Both are one-line fixes in the docs. No architectural redesign is needed for any finding.
