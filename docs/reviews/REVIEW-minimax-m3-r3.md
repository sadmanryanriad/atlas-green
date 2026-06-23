# Atlas Green — Pre-Implementation Design Review (Iteration 3)

- **Model:** minimax-m3 (opencode-go/minimax-m3)
- **Date:** 2026-06-23
- **Scope:** Stress-test the v1.2 fixes (decisions #43–#48 and the amended #7/#28/#29/#33/#34/#37/#38/#39/#40/#42) against the seven pressure points the prompt called out, plus new edge cases and Phase 0 readiness.

## Executive Summary

The v1.2 round decisively closed the r2 spec bugs and the r1 criticals: the unique-index bug on `intervals` is gone, token-loss re-enroll now correctly rotates (#43), the corp-CIDR allowlist unblocks the 100-PC rollout (#44), revoked-token heartbeat keeps the device visible without re-issuing ingest (#45), the named pipe is anti-spoofed (#46), the 426-on-heartbeat brick path is fixed (#47), and "forget device" is atomic (#48). Idle hysteresis was rebuilt to keep a bursty-input call `active` (#34 amended), the post-resume state is now re-derived from the next sample rather than hardcoded `idle` (#33 amended), the clock-skew offset is applied before day-bucketing (#40 amended), and the `machineGuid` clone-detection path is real (#42 amended). Phase 1a/1b is a defensible split. What v1.2 still leaves open: the **opposite over-correction in idle hysteresis** — "any input resets the 300 s timer" lets USB jitter / wireless-mouse keepalive / RDP touch events keep a device `active` indefinitely, the mirror-image failure of v1.1's bug; the **`liveness + flags` model is a derived view of fields that are still stored on the device doc** (`blocked`, `archived`, `lastSeenAt`), so the "dashboard reads only liveness+flags" rule can silently drift from source data; the **Phase 1a exit gate "Phase 2 may now begin in parallel"** overstates the parallelism (only the server-side Phase 2 work can, not the agent-side buffer/sync); the **`configSchemaVersion` mismatch is still silently dropped** (deepseek r2 H4 — the agent logs locally, the server never knows); the **batches TTL (~180 d) combined with a corrupted local buffer** can cause a legitimate-batch re-insert with the dedup receipt evicted, producing real duplicates; the **contract body still says `X-Atlas-Contract: 1.1` in the header field** while the doc title is v1.2; the contract is **missing `pending` from `liveness` and `possible_clone` from `flags`** despite both being in Decision #39 and ARCHITECTURE.md §7; the **`reauthorized` lifecycle is ambiguous** (does it reset on read? on re-enroll? on success?); the **"forget device" admin action has no API endpoint yet**; and the **`capture` block doesn't indicate "capturing disabled" for revoked/archived devices** so the dashboard sees a healthy capture-health panel for a device that is not actually capturing. None block Phase 0; one (version-number inconsistency) is a 30-second doc fix; the others are spec clarifications best resolved before the phase that owns them.

---

## Critical

### C1 — `X-Atlas-Contract: 1.1` header field in the contract body is stale; the document is labelled v1.2 throughout the headers but the body still sends/receives the v1.1 header
**File:** `docs/API-CONTRACT.md` (header line 1 "v1.2"; body lines 32, 33, 220, 270, 323)
**Problem:** The document title says "(v1.2)" and the changelog block in lines 8–17 documents the v1.2 changes (rotation, revoked heartbeat, 426 scope, etc.). But the wire header field is still **v1.1**:

```
- **Contract version** is echoed by the server in every response (header
  `X-Atlas-Contract: 1.1`) and sent by the agent as `X-Atlas-Contract: 1.1`. On a
  major mismatch the server returns `426 Upgrade Required`.
```

And §5 Conventions repeats it. The "v1.2" in the title is the *doc* version; the wire header is the *contract* version. The two are meant to be the same (per the v1.1 changelog: "v1.1 (2026-06) folds in the multi-model review"). v1.2 introduces new response fields (`liveness`, `flags`, `reauthorized`, `contractDeprecated`, `attributes.timezone`, `attributes.machineGuid`, `dropped.byState`, `capture.helperStartedAtUtc`, `configSchemaVersion` in the heartbeat body) and new error semantics (`/heartbeat` never 426) — that's a contract change. A v1.2 doc with `X-Atlas-Contract: 1.1` is internally inconsistent: implementers will copy whichever string they see first. The mismatch also breaks the "bump the version, then update both sides" workflow (Decision #5 / API contract header): the contract is bumped, the header isn't. The two repos implementing against v1.2 will set the header to 1.1 (copying the body) and the server's mismatch check at 1.2 will incorrectly fire 426 on v1.2-compliant agents.
**Recommendation:** Change every occurrence of `1.1` in the body of `docs/API-CONTRACT.md` to `1.2` (lines 32, 33, 220, 270, 323 — the four places `X-Atlas-Contract: 1.1` appears in the running text and the §1/§3 response examples). Add a line to the changelog: "The wire contract header `X-Atlas-Contract` is bumped to **1.2** alongside the v1.2 doc." Optionally, add a CI lint that asserts the title's `vN` and every `X-Atlas-Contract: N` in the doc are the same number. This is a 5-minute fix; without it, the very first Phase 0 /enroll will be ambiguous.

---

## High

### H1 — Idle hysteresis "any input resets the 300 s timer" is the opposite over-correction: USB jitter / wireless-mouse keepalive / isolated input events can keep a device in `active` indefinitely
**File:** `docs/DECISIONS.md` #34, `docs/ARCHITECTURE.md` §4, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §3
**Problem:** v1.1's "leave `idle` only after ~30 s sustained input" was wrong (would record a 3-hour call as `idle` because call participants type in short bursts, not 30 s contiguous — glm-5.2 r2 H4). v1.2 inverted it: "**any input** resets the 300 s timer, **leave `idle` on any input**." The v1.2 design correctly fixes the call-as-idle case, but it introduces the opposite failure: a user who steps away from their desk for 10 minutes can be kept in `active` by a single isolated input event at the 295-second mark. Real-world input noise that triggers this:

- **Wireless mouse / keyboard keepalive polls** — many wireless peripherals send a tiny input event every 30–120 s for battery-state synchronization, link-quality re-checks, or DPI-preset re-assertion. The agent sees these as input.
- **USB hub polling** — some hubs emit a phantom input event on port-enumeration or when a device is re-plugged (sleep → wake on a docked laptop).
- **RDP / VNC keepalive** — the Microsoft RDP client sends a synthetic input event every ~60 s to keep the session alive, even when the user is genuinely AFK.
- **Bluetooth re-connection** — a Bluetooth mouse that re-pairs after the laptop wakes from sleep sends a flurry of input events for 1–2 s.

Any of these is enough to reset the 300 s timer. Over a 5-minute window, the user is "active" despite typing nothing. Worse, with the **no-sustained-input requirement to leave `idle`**, the same noise keeps the state in `active` indefinitely — there's no path back to `idle` if input events arrive every 4–5 minutes (well under the 300 s threshold).

This is the *mirror image* of the v1.1 bug, and just as damaging. v1.1 made bursty-input users look idle; v1.2 makes input-noise users look active. A 5-min idle threshold that never fires is a meaningless threshold.

**Recommendation:** The right rule sits in the middle: the 300 s idle-entry timer is reset **only by a "meaningful" input event**, where meaningful = (a) a contiguous input event of ≥ 2 s duration, OR (b) a burst of ≥ 3 input events within 10 s (typing/scrolling). A single isolated mouse-poll event at second 295 of the timer does not reset. A typing burst at second 295 does. This costs one `inputEventCount: int` (resets on timer) and one `inputEventTimestamps: Queue<DateTime>` (for the burst test) on the state machine. Add the rule to skill §3: "The 300 s timer is reset by (a) an input event sustained for ≥ 2 s, or (b) ≥ 3 input events within a 10 s rolling window. Isolated single events (e.g. mouse-poll jitter) do not reset. The timer is also reset on any explicit input gesture the aggregator recognizes (key down, mouse click, scroll wheel — but **not** mouse move < 4 px, which most jitter-class events satisfy)." Add a Phase 1b unit test: "isolated mouse-move events at seconds 100, 200, 295, 400 of a 600 s window → state goes `idle` at second 600; typing burst at second 295 → state stays `active`." Without this fix, the state machine never enters `idle` on any machine with a wireless mouse — the 5-min idle threshold is silently neutered on half the fleet.

### H2 — `liveness + flags` is the authoritative dashboard view, but the source fields (`blocked`, `archived`, `lastSeenAt`, `revokedAt`) are still on the device doc and the server must keep them in sync
**File:** `docs/DECISIONS.md` #39, `docs/ARCHITECTURE.md` §7, `docs/API-CONTRACT.md` §3, `atlas-console/.agents/skills/atlas-api-contract/SKILL.md` §7
**Problem:** Decision #39 collapsed the three ad-hoc derived fields into `liveness + flags`, with the explicit invariant "the dashboard reads `liveness` for the badge and renders `flags` as chips — it never re-derives them from `lastSeenAt`/`revokedAt` separately." Good design. But the device doc **still stores** the source fields:

```
devices: { _id, pcName, hostname, osUsername, timezone, employeeId?, agentVersion, os,
  liveness, flags, lastSeenAt, lastSampleUtc, helperRunning, helperStartedAtUtc,
  clockOffsetMs?, lastDropped?, enrolledAt, blocked: bool, archived: bool, ... }
```

Plus `deviceTokens.revokedAt` and `deviceTokens.supersededAt`. So there are now **two parallel sources of truth for the same state**: the `liveness`/`flags` fields and the underlying `lastSeenAt`/`blocked`/`archived`/`revokedAt` fields. The server has to keep them in sync, and the contract explicitly forbids the dashboard from re-deriving. The risk: an admin or a server bug updates `blocked: true` but forgets to add `revoked` to `flags`. The dashboard shows the device as `online` (not `revoked`). Or: an admin archives a device (`archived: true`) and the server forgets to add `archived` to `flags` — the device is hidden from default views but the badge still says `online` without the `archived` chip. The opposite direction is worse: a developer sets `flags: [revoked]` directly without updating `deviceTokens.revokedAt` — the next /ingest call (which checks `revokedAt`) gets a 401 unexpectedly, and the agent falls into the re-enroll loop.

The cleanest design is to **make `liveness` and `flags` purely derived, never stored**. The device doc keeps the source fields (`lastSeenAt`, `blocked`, `archived`, `agentVersion`, `clockOffsetMs`, `machineGuid` history). The server computes `liveness` (from `lastSeenAt` vs `heartbeatIntervalSec × 3`) and `flags` (from the source fields: `outdated` if `agentVersion < minSupportedVersion`, `clock_skewed` if `|clockOffsetMs| > 1h stable 2 heartbeats`, `revoked` if `deviceTokens.revokedAt`, `archived` if `archived: true`, `health_warning` if health thresholds, `possible_clone` if machineGuid history diverges). The dashboard and the API response use the computed values. The device doc schema drops `liveness` and `flags` entirely.

Alternatively: store `liveness`/`flags` as a single denormalized cache field and **forbid any code path that updates the source fields without updating the cache** (enforced by a single `updateDeviceState(deviceId, sourceChange)` function in the console). This is the "MongoDB updates are atomic per document" approach — one update, both fields move together. Document the rule: "Every server-side mutation of `blocked`, `archived`, `lastSeenAt`, or `deviceTokens.revokedAt` MUST go through `updateDeviceState`; direct field updates are a bug." Add a Phase 2 integration test: "admin sets `blocked: true` → on the next heartbeat, `flags: [revoked]` is in the response and `deviceTokens.revokedAt` is set; admin unblocks → on the next heartbeat, `flags: [revoked]` is absent and ingest returns 200."

Without one of these two designs, the v1.2 "liveness+flags is authoritative" invariant is enforced by convention, not by the schema. Convention erodes under pressure (multiple implementers, multiple phases, multiple admin tools).

### H3 — `configSchemaVersion` mismatch is still silently dropped on the agent and never reaches the server (deepseek r2 H4, unaddressed in v1.2)
**File:** `docs/API-CONTRACT.md` §1/§3 (config block), `docs/DECISIONS.md` #39 (`outdated` flag), `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §4
**Problem:** v1.1 added `configSchemaVersion` to the config block. v1.2 carries it forward: `/enroll` and `/heartbeat` responses include `config.configSchemaVersion`, and the agent's heartbeat body includes `configSchemaVersion: 1` (the agent's supported version). The contract says: "a `configSchemaVersion` newer than it supports → ignore-unknown + log." Fine for forward-compat. But the server has no way to know the agent ignored fields. The `outdated` flag in Decision #39 is triggered by `agentVersion < minSupportedVersion` — a different signal. A v1.0 agent (configSchemaVersion 1) running against a v1.2 server (configSchemaVersion 2) is silently using partial config: the server sends a v1.2-shaped config (e.g., a new `mediaDetectionMode: "gsmtc+wasapi"` field), the agent ignores it, the agent uses its hardcoded `gsmtc`-only behavior, the dashboard shows fewer `passive_media` intervals for those 100 v1.0 agents — a silent behavioral mismatch that no one notices until a v2.0 update lands and the dashboard's "media time per device" chart suddenly shifts.

The same is true in reverse: a v1.2 agent (configSchemaVersion 2) running against a v1.0 server (no `configSchemaVersion` in the response) — the agent sees the missing field and falls back to its defaults. That's correct, but again, no signal to either side.

**Recommendation:** Add a `config_mismatch: bool` flag to the `flags` array (Decision #39's list, alongside `outdated`/`clock_skewed`/etc.). The server sets it on the device when (a) the agent's `configSchemaVersion` (heartbeat body) < the server's current `configSchemaVersion`, OR (b) the server's config response was missing `configSchemaVersion` (downgrade). The dashboard surfaces `config_mismatch` as a chip on the device card. The agent can also log locally — the contract already says "log a warning" — but the server needs to know too. This is the same pattern as `outdated` (a server-computed flag based on a heartbeat field), and the same fix. Add to ARCHITECTURE.md §7 flags list and to the heartbeat panel rendering. Without this, every config change in v1.x ships with a silent-fallback window that the fleet will hit without warning.

### H4 — Phase 1a exit gate "Phase 2 may now begin in parallel" overstates what's parallelizable; agent-side buffer + sync need Phase 1b's intervals to be testable
**File:** `docs/PLAN.md` Phase 1a (exit gate, line 91-93)
**Problem:** The Phase 1a exit gate is: "the helper launches into a real user session (verified non-admin), relaunches on a fast user switch, and foreground samples reach the service over the verified pipe — visible in logs. (Phase 2 may now begin in parallel.)" The split is good (Phase 1 is the critical path and was overloaded per qwen r2 M5 / mimo r2 sequencing). But "Phase 2 may now begin in parallel" is misleading. Phase 2 is "Buffer + sync" and has two halves:

- **Console/server-side** (the new bits): `/api/v1/ingest` endpoint, `batches` collection, atomic transaction, `deviceTokens` schema, `dataGaps` persistence, `status` state-machine wiring, rate limiting, org-key rotation, "forget device." All of this is testable against synthetic /ingest calls from a test harness (curl, a tiny throwaway agent) — independent of the helper/aggregator.
- **Agent-side** (the new bits): SQLite buffer schema, AES-GCM encryption, sync loop, 413 self-split, dropped-byState, exponential backoff, config validation, token-loss recovery. **This is testable only against real intervals produced by the aggregator** (Phase 1b). Without 1b, the agent's buffer sits empty, the sync loop never fires, the dropped-byState accounting is never exercised, the 413 self-split is never triggered, and the end-to-end "intervals captured offline flush to MongoDB" exit gate (Phase 2 line 157) cannot be reached.

So the parallelism is real but partial. "Phase 2 may now begin in parallel" is true for the server-side half and false for the agent-side half. As written, an implementer might read the gate and start the agent-side Phase 2 work against 1a, find it untestable, and either block on 1b (waste) or skip the tests (regress later).

**Recommendation:** Rewrite the Phase 1a exit-gate parenthetical to: "(Phase 2's **server-side** work — `/ingest` endpoint, `batches` collection, atomic transaction, rate limits, org-key rotation, status state machine — may now begin in parallel. Phase 2's **agent-side** work — SQLite buffer, sync loop, dropped-byState, config validation, token-loss recovery — starts **after** Phase 1b's exit gate, because it is testable only against the aggregator's intervals.)" Two-line clarification, prevents a Phase 2 split that has to be undone. Alternatively, split Phase 2 itself into 2a (server-side ingest) and 2b (agent-side buffer+sync) the same way 1 was split — the symmetric structure is easier to plan around.

### H5 — `batches` TTL (~180 d) combined with a corrupted local buffer (stale `synced = 0` on actually-sent rows) can cause a legitimate-batch re-insert with the dedup receipt evicted → real duplicates
**File:** `docs/DECISIONS.md` #28, `docs/ARCHITECTURE.md` §7, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §4
**Problem:** Decision #28 added a `batches` TTL of ~180 d. The justification (correct) is that a receipt older than the agent's max buffer hold (~75 d default) cannot be legitimately replayed. The TTL is safe for **normal flow** — an agent that never crashes and never corrupts its local DB will never re-send an already-sent batch.

But: the agent's local buffer is encrypted SQLite, and SQLite databases are not perfectly crash-safe. A power loss mid-write, a disk-full event, or a buggy migration script can leave the local buffer in a state where rows are marked `synced = 0` (the agent thinks it hasn't sent them) but were actually sent and ACK'd. Without the TTL, the server's `batches` collection has the receipt, the agent's retry with the same `batchId` is deduped, no duplicate. **With** the TTL of 180 d, a re-send of rows from 200 d ago (an unlikely-but-real corruption scenario on a long-lived machine) has no matching receipt on the server — the new `batchId` is fresh, the server inserts the rows, and the dashboard shows the activity twice.

This is a corruption-only failure mode — the prompt asks specifically "can a legitimately-buffered batch now be re-inserted as 'new' and duplicate data?" In normal flow, **no**: the legitimately-buffered batch is a batch that was never sent (the agent is offline / was disconnected); the server's receipt for that batch never existed; the new `batchId` is fresh; no duplicate. The TTL is not in the path. So the prompt's specific concern does not apply to normal flow.

**But** the corruption scenario is real (low-probability but high-impact), and the TTL **creates** the window where corruption becomes duplicates. The fix is one line in the skill: "The agent MUST drop local rows older than `serverBatchesTtl` (default 180 d) even if `synced = 0`. The reasoning: if a row is older than the server's batches TTL, either (a) the server's receipt for the original batch was evicted, so a re-send is a duplicate the server can't dedupe, or (b) the row was never sent (the agent went offline for >180 d) and is past the buffer cap anyway (`bufferCapDays` max 365 d, default 75 d, so this is rare). The agent enforces this by limiting its unsynced-row retention to `min(bufferCapDays, serverBatchesTtl)`. A row older than that is purged with a `dropped` report (so the dashboard sees a data gap, not silent truncation)." This is one paragraph; without it, the TTL is an unbounded duplicate window for corrupted buffers.

Add a Phase 2 checklist item: "agent enforces `unsyncedRowRetention = min(bufferCapDays, serverBatchesTtl)`; older unsynced rows are dropped with a `dropped` report." Add a chaos test in Phase 7: "simulate a local DB corruption on a 200-d-old machine, verify the agent re-bootstraps cleanly (new device, new batchIds, no server-side duplicate of the evicted-receipt interval ranges)."

---

## Medium

### M1 — "Forget device" admin action has no API endpoint in v1.2; Phase 4 owns it but the spec is empty
**File:** `docs/DECISIONS.md` #48, `docs/ARCHITECTURE.md` §7, `atlas-console/.agents/skills/atlas-api-contract/SKILL.md` §3
**Problem:** Decision #48 specifies the action: "atomically revokes the token, sets `blocked`, deletes the device's `intervals` and its `batches` receipts. Outside that action, receipts follow the TTL (#28)." This is the right behavior, but the contract never defines an API endpoint for it, the console skill never lists it, and the Phase 4 checklist (which owns the dashboard implementation) doesn't enumerate it. An implementer in Phase 4 will have to invent the shape: HTTP method, auth (admin Cloudflare-Access email + role?), response, audit-log entry shape, "are you sure?" confirmation, what happens to the dashboard's heart-beat panel view, what happens to a `dataGaps` entry that referenced this device, etc. The action's correctness is irrelevant if no one can call it.
**Recommendation:** Add to `docs/API-CONTRACT.md` (new §6) or to the Phase 4 checklist explicitly: `POST /api/v1/admin/devices/:deviceId/forget` with the documented behavior. Specify: (a) auth = Cloudflare-Access + admin role (Phase 4 NextAuth); (b) response = 200 with `{ deviceId, deletedIntervals, deletedBatches, revokedAt }`; (c) audit log entry `{ adminId, action: "forget_device", target: deviceId, before: { intervals: N, batches: M }, after: { intervals: 0, batches: 0 }, atUtc }`; (d) the device doc is **not** deleted — `archived: true, blocked: true, liveness: offline, flags: [archived, revoked]` is set so the dashboard can show a tombstone ("forgotten on date X by admin Y") rather than a 404; (e) `dataGaps` entries for the device are also deleted (they reference the device); (f) `auditLog` entry is created **before** the deletion so the action is auditable even after the data is gone. Pair with the Phase 4 dashboard action button.

### M2 — `capture` block doesn't indicate "capturing disabled" for revoked / archived devices; the dashboard sees `helperRunning: true, lastSampleUtc: recent` and thinks capture is fine
**File:** `docs/API-CONTRACT.md` §3 (`capture` block), `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §4
**Problem:** Per Decision #45 / skill §4, a revoked device's agent "stops generating new intervals, keeps its buffer, and only heartbeats (`status` reflects `revoked`)." The helper is **still running** (the service keeps supervising it), the helper is **still sending samples** (the helper doesn't know it's "revoked" — only the service / agent knows the token's revoked status), and the service's `capture.helperRunning: true, lastSampleUtc: recent` are correct (the helper IS running, samples ARE flowing). But the agent has stopped writing those samples to the buffer. The dashboard's capture-health panel sees a healthy helper and a recent sample and infers "capturing is fine" — but the device is revoked and the agent is not actually storing intervals. The "online" badge is correct (the service is alive) but the "capturing" implication is wrong.

This is a UX bug, not a correctness bug. An admin looking at a revoked device's row sees a green capture-health indicator next to a red "revoked" chip and may think "this device is being captured correctly" when it's actually not.
**Recommendation:** Add to the `capture` block a `enabled: bool` field (or `captureDisabled: bool` — pick one). The agent sets `enabled: false` when it's in the revoked/archived state and has stopped writing to the buffer; `enabled: true` otherwise. The dashboard renders the capture-health panel as muted/disabled when `enabled: false`, regardless of the `helperRunning` / `lastSampleUtc` values. Document the field in contract §3. This is a 1-line field on the agent's heartbeat-builder and a 1-line render branch on the dashboard. Without it, the capture-health panel is misleading for any non-capturing device.

### M3 — Heartbeat response `liveness` enum in the contract is missing `pending` and `possible_clone` flag; the Decision has both, the contract doesn't
**File:** `docs/API-CONTRACT.md` §3 (response example), `docs/DECISIONS.md` #39, `docs/ARCHITECTURE.md` §7
**Problem:** The contract §3 example shows:
```json
"liveness": "online",
"flags": []
```
But Decision #39 and ARCHITECTURE.md §7 define `liveness: pending | online | offline` and `flags: [outdated, clock_skewed, revoked, archived, health_warning, possible_clone]`. The contract example is the only place a Phase 0/2 implementer will copy from, and it under-specifies the enum:

- **Missing `pending`**: a freshly-enrolled device that has never heartbeated should be `liveness: "pending"`, not `online`. The contract example says "online" with no alternative shown, so an implementer might hardcode the response to `online` and a pending device looks "online" before it's ever been seen.
- **Missing `possible_clone`**: Decision #42 added this flag ("Multiple distinct `machineGuid`s on one `deviceId` → a `possible_clone` flag"). The contract flags example shows `[]` and never demonstrates a `possible_clone`. An implementer reading the contract alone won't know to set this.

The contract should be the authoritative wire format (per its own header). A wire format that omits two enums is not authoritative.
**Recommendation:** Edit the contract §3 response example to show three cases: (a) just-enrolled → `liveness: "pending", flags: []`; (b) normal → `liveness: "online", flags: []`; (c) a flagged device → `liveness: "online", flags: ["outdated", "clock_skewed", "possible_clone"]`. Or, add a "liveness values" table in §3 documenting `pending`/`online`/`offline` and a "flags values" table documenting all six flags with the trigger condition for each. This is a doc fix; Phase 0/2's correctness depends on it.

### M4 — `reauthorized` flag lifecycle is ambiguous; "true once" is three different things
**File:** `docs/API-CONTRACT.md` §3 (`reauthorized: false`), `docs/DECISIONS.md` #45
**Problem:** Decision #45 / contract §3 says `reauthorized: true` "once after an admin un-revoke → agent resumes capture." Three plausible implementations:

1. **One-shot, reset on read**: server sets `reauthorized: true` after admin un-revoke; the first heartbeat that reads it gets `true`, the server resets to `false` on the next response. If the agent misses the heartbeat (network blip, down at the moment of un-revoke), the next heartbeat says `false` and the agent stays revoked until the admin resets the flag.
2. **Sticky until re-enroll**: server sets `reauthorized: true` after admin un-revoke; the flag stays `true` for every heartbeat until the agent successfully re-enrolls (then the server resets to `false`). Robust against network blips but means the dashboard shows `reauthorized: true` indefinitely for a slow agent.
3. **Sticky until first successful re-enroll, with re-set on failure**: like (2), but if the agent re-enrolls and the new token is immediately rejected (e.g., new admin revoke), the server re-sets `reauthorized: true` to trigger another attempt. Most robust; most complex.

The contract doesn't say which. An implementer picks one and the behavior diverges from the other 100 devices.
**Recommendation:** Specify in contract §3 behavior: "**`reauthorized` is sticky until the agent successfully re-enrolls.** The server sets `reauthorized: true` immediately after the admin un-revoke action. The flag remains `true` for every subsequent heartbeat until (a) the agent calls `/enroll` and receives a 200, or (b) the admin re-revokes the device. If the agent's re-enroll fails (e.g., the new token is revoked before the agent can use it), the server re-sets `reauthorized: true` to trigger another attempt. A missed heartbeat does not consume the signal." This is the most robust of the three and the cheapest to implement (one field on the device doc, two write paths). Document the failure mode: a slow agent that takes 24 h to come back online after an un-revoke will sit at `liveness: offline, flags: [revoked], reauthorized: true` for 24 h, which is correct (the agent hasn't acted yet) and the admin sees the device is in a "waiting to recover" state.

### M5 — `health_warning` threshold is "sustained" in ARCHITECTURE.md but unquantified; minimax r2 M12 still unaddressed
**File:** `docs/ARCHITECTURE.md` §7 (`health_warning (cpu > 5% sustained or mem > 200 MB)`), `docs/DECISIONS.md` #39
**Problem:** The `health_warning` flag exists in Decision #39's list but the trigger is specified in ARCHITECTURE.md §7 as "cpu > 5% sustained or mem > 200 MB" with no definition of "sustained." minimax r2 M12 suggested "cpuPct > 5% across 3 heartbeats, or memMb > 200." v1.2 didn't pick this up. Two implementers will interpret "sustained" differently (3 heartbeats? 10? 1 minute?).
**Recommendation:** Add to ARCHITECTURE.md §7 or contract §3: "**`health_warning` is set when** `cpuPct > 5` for **3 consecutive heartbeats** (each ~5 min apart = 15 min sustained high CPU), **OR** `memMb > 200` for **any single heartbeat** (memory leak is more diagnostic than a CPU spike). The flag is cleared when the condition resolves for **2 consecutive heartbeats**." Match the spec to the r2 recommendation. Without it, the flag is a vibe-based trigger.

### M6 — `contractDeprecated` flag has no defined trigger; who sets it and when?
**File:** `docs/API-CONTRACT.md` §3 (`contractDeprecated: false` in response), `docs/DECISIONS.md` #47
**Problem:** Decision #47 introduces the `contractDeprecated` flag in the heartbeat response. Contract §3 shows `"contractDeprecated": false` in the example. But neither the contract nor the decision specifies *when* the server sets it to `true`. Is it an admin action? A `settings.contractDeprecationMap: { "1.0": "warn", "0.9": "reject" }`? A scheduled rollout? An implementer will guess.
**Recommendation:** Add to contract §5 and Decision #47: "**`contractDeprecated` is server-set** based on `settings.contractDeprecation: { [contractMajor: number]: "warn" | "reject" | null }`. When an admin sets `1.0: "warn"`, every heartbeat from a v1.0 agent gets `contractDeprecated: true` (the v1.0 agent still gets the update block, can self-update). The server **never sets `contractDeprecated: true` for the current major** (v1.2 agents always see `false`). The agent's behavior on `contractDeprecated: true`: log a prominent warning, surface it in the dashboard as a chip, and (if not already on the latest version) prioritize the next self-update. Document the setting shape in ARCHITECTURE.md §7 (`settings.contractDeprecation`).

### M7 — `dataGaps` collection has no dashboard surfacing rule; Phase 4 doesn't list a "data gap" view
**File:** `docs/DECISIONS.md` #29 (new), `docs/ARCHITECTURE.md` §7 (`dataGaps` collection), `docs/PLAN.md` Phase 4
**Problem:** ARCHITECTURE.md §7 defines `dataGaps: { _id, deviceId, count, byState, oldestUtc, newestUtc, detectedAt }` — persisted from the heartbeat `dropped` block so a buffer-overflow gap stays visible historically. Skill §3: "the server persists a non-zero dropped as a dataGap record." Good. But Phase 4's checklist (lines 195-216) doesn't include a "render data gaps on the timeline" item. The data is persisted; nothing surfaces it. An admin looking at a device's timeline a month later sees a blank span with no indication that the device dropped N intervals on date X. The gap is invisible.
**Recommendation:** Add to Phase 4 checklist: "Per-employee daily timeline shows `dataGap` markers (e.g., a greyed-out band labeled 'N intervals dropped' on the day the gap was detected). The dataGaps collection is the source; the timeline renders from it. The company overview shows a count of devices with active dataGaps in the last 7 days as a fleet health indicator." Without this, the `dataGaps` collection is write-only storage.

### M8 — `machineGuid` field path is inconsistent: enroll uses `machineHints.machineGuid`, heartbeat uses `attributes.machineGuid`
**File:** `docs/API-CONTRACT.md` §1 (enroll) vs §3 (heartbeat), `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §5
**Problem:** Enroll request body has `"machineHints": { "machineGuid": "..." }`. Heartbeat `attributes` block has flat `"machineGuid": "..."`. The same field (`machineGuid`) has two different paths. An implementer who serializes the request bodies from a single DTO will either rename one to match the other (breaking one endpoint) or keep two fields (DRY violation). Decision #42 says "send `machineGuid` in the heartbeat too so a disk-clone is detectable" but doesn't fix the path. The skill §5 says the same. The contract is the authoritative wire format; it should have one path.
**Recommendation:** Pick one. Either (a) put `machineGuid` in `attributes.machineGuid` for both (heartbeat's shape), or (b) put it in `machineHints.machineGuid` for both (enroll's shape). Option (a) is cleaner because the heartbeat's `attributes` block already has a flat shape, and the enroll can move `pcName, hostname, osUsername, timezone, machineGuid` into an `attributes` block (renaming `machineHints` → just `attributes`). This requires a v1.2 contract addendum (changing the enroll request shape), but it's a one-time change and the console is not in production yet. Update the contract §1 enroll body to match §3's `attributes` shape, and the `machineHints` wrapper goes away.

### M9 — Token double-rotation race: the contract doesn't cap re-enrolls or define behavior on a second concurrent rotation
**File:** `docs/DECISIONS.md` #43, `docs/API-CONTRACT.md` §5 (auth failures), `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §4
**Problem:** Decision #43 specifies rotation on token loss. The skill §4 says: "On `401`, try a single re-`/enroll` with the org key. A **lost/corrupted token** re-enroll **rotates** (server mints a new token; buffered rows then sync)." Single retry. But the contract §5 says: "If `/enroll` returns `403` (blocked device) or re-enroll fails repeatedly, the agent stops generating new intervals..." — "fails repeatedly" is undefined. The edge case: an agent network-blips mid-/enroll and retries. Two concurrent /enrolls from the same device. Server rotates twice. The agent receives one response and uses the new token; the other response is lost. The agent later gets a 401 (the token it used is now superseded by the third rotation, if a third /enroll happens). The agent re-enrolls again, rotates a fourth time. An infinite loop is possible if the server is buggy (e.g., a stuck idempotency check that always re-rotates).

The contract needs (a) a cap on consecutive re-enrolls, and (b) a definition of "fails repeatedly" in §5.
**Recommendation:** Add to contract §5 and skill §4: "After **5 consecutive re-enroll attempts over 1 hour** with no successful post-enroll /ingest or /heartbeat, the agent enters a **terminal `liveness: offline, flags: [revoked, reauth_exhausted]` state**: the agent stops generating intervals, stops re-enrolling, keeps heartbeating at a reduced rate (once per 15 min) for 24 h, then stops. The dashboard surfaces `reauth_exhausted` as a chip. The agent's buffer is preserved (encrypted, on disk). A human (IT) is required to manually unstick the device (re-enroll on the user's behalf, or restart the service). This prevents an infinite re-enroll loop from draining the device battery / filling the org-key rate-limit budget." Add a Phase 2 integration test: "simulate a 401 storm, verify the agent stops re-enrolling after 5 attempts and the dashboard shows the terminal chip." The "fails repeatedly" wording in §5 should reference this concrete number.

### M10 — `archived` flag's agent-side behavior is unspecified; the skill covers `revoked` but not `archived`
**File:** `docs/API-CONTRACT.md` §5 (revoked terminal state), `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §4, `docs/DECISIONS.md` #21
**Problem:** Decision #21 (server retention indefinite, with manual per-PC archive/delete). ARCHITECTURE.md §7: `archived` is one of the orthogonal flags. But the agent's behavior on archive is not specified:

- Does the agent keep generating intervals on an archived device?
- Does /ingest succeed (200) or fail (401) for an archived device?
- Does the agent's buffer get purged on archive, or kept?
- Is the agent's terminal-state flow (stop generating, keep buffer, heartbeat only) the same as for `revoked`?

Decision #21 implies archive is "hide from default views" — the data is kept, the device is just hidden. The natural reading is that the agent continues capturing (the data is still being collected for compliance), /ingest still succeeds, and the only effect is dashboard visibility. But the spec doesn't say. An implementer might treat `archived` like `revoked` (agent stops capturing) and silently lose data on archived devices.
**Recommendation:** Add to skill §4 and contract §5: "**`archived` is a dashboard-visibility flag, not an agent-state flag.** An archived device's agent continues capturing and ingesting exactly as before. /ingest returns 200 (not 401 — the token is still valid). The dashboard hides the device from default views (`liveness: online` is shown only when an admin explicitly views 'archived devices'). `archived` and `revoked` are independent: a device can be `online + archived` (capturing, hidden from default view), `online + revoked + archived` (capturing data is preserved but the token is dead, agent enters the revoked terminal state). The 'forget device' action (#48) is the only one that deletes data; archive is reversible." Document the dual-meaning of `archived` (dashboard flag, not auth flag) in Decision #39's flags list.

### M11 — Revoked-token /heartbeat returns the full `config` and `update` blocks; a leaked revoked token can probe the agent's full state
**File:** `docs/API-CONTRACT.md` §3 (response with `config` + `update`), `docs/DECISIONS.md` #45
**Problem:** Decision #45 / contract §3: `/heartbeat` accepts a revoked token and returns `200` with the full response, including `liveness`, `flags`, `config: { ... }`, `update: { ... }`. A rogue process that obtained a revoked token (e.g., exfiltrated from a departed employee's machine, or extracted from a DPAPI backup) can:

- Read the agent's current `liveness` (online/offline).
- Read the `flags` (revoked, clock_skewed, etc.), learning when the device was un-revoked.
- Read the full `config` block (sample interval, idle threshold, sync interval, etc.) — the agent's full operating envelope.
- Read the `update` block (latest version, manifest URL, mandatory flag) — the agent's update path.

The threat model is "standard users only" and standard users can't read DPAPI machine-scope secrets (correct). So the practical attack surface is small. But the dashboard's view of a revoked device can be kept "online" by a leaked revoked token (an attacker heartbeats, the server processes, the device's `liveness` flips online; the dashboard sees the device as online despite the real machine being off).
**Recommendation:** Document the attack surface in contract §3 behavior and Decision #45. Specifically: "(a) /heartbeat from a revoked token reveals `liveness` and `flags` (these are visible anyway from any admin's dashboard view — the revoked token grants no extra privilege here). (b) /heartbeat from a revoked token also reveals `config` and `update` — these are arguably sensitive (operating envelope + update path), but the threat model (standard users only) means the token could only have been exfiltrated by an admin or by a local administrator, both of whom have admin-level access to the dashboard anyway. The trade-off (recovery lifeline vs. info disclosure) favors the recovery lifeline. Document this in the security model." Add a future hardening note: a future "stricter" mode could return a stripped response (`{ liveness, flags, reauthorized, serverTimeUtc }` only) for revoked tokens, but v1 ships the full response for simplicity.

### M12 — `liveness + flags` transitions are still not enumerated (minimax r2 H4, still open after the liveness+flags amendment)
**File:** `docs/DECISIONS.md` #39, `docs/ARCHITECTURE.md` §7
**Problem:** v1.2 amended Decision #39 to be `liveness + flags` instead of a single enum — a strict improvement. But the **transitions** are still not enumerated (the same gap minimax r2 H4 flagged for the v1.1 single-enum). The flags are orthogonal so the state-space is larger (1 liveness × 2^6 flags = 128 combinations), and each flag has its own trigger and clear condition. Without an enumeration, an implementer must infer:

- `liveness`: `pending → online` on first heartbeat; `online → offline` after `heartbeatIntervalSec × 3`; `offline → online` on any heartbeat; `pending → offline` if no heartbeat ever arrives (the dashboard's "pending" badge becomes "offline" after 3× the heartbeat interval).
- `outdated`: set when `agentVersion < minSupportedVersion`; cleared when the agent updates (next heartbeat has the new `agentVersion`).
- `clock_skewed`: set when `|clockOffsetMs| > 1h` for 2 consecutive heartbeats; cleared when `|clockOffsetMs| < 30 min` for 2 consecutive heartbeats.
- `revoked`: set on admin revoke; cleared on admin un-revoke (and `reauthorized: true` on the next heartbeat).
- `archived`: set on admin archive; cleared on admin un-archive.
- `health_warning`: per M5 above (sustained thresholds).
- `possible_clone`: set when 2+ distinct `machineGuid`s seen for one `deviceId`; cleared never (the clone suspicion is sticky, even if the second machineGuid is no longer seen — it might come back).

The transitions for `outdated` and `possible_clone` in particular interact: a device that updates to a new agent version clears `outdated`; a device that was cloned in the past retains `possible_clone` forever; the two are independent. An implementer who doesn't know the rules will write a state machine that gets one or more wrong.
**Recommendation:** Add a transition table to ARCHITECTURE.md §7 (next to the liveness+flags definition). Rows are the events; columns are the flag transitions. Same idea as minimax r2 H4's table but expanded for the new flag set. This is a 20-line doc that prevents a Phase 4 dashboard rewrite.

---

## Low

### L1 — v1.1/v1.2 version-number inconsistency across the docs (separate from C1)
**File:** `docs/AGENTS.md` (no version field), `docs/API-CONTRACT.md` v1.2, `docs/DECISIONS.md` "Decisions added from the iteration-2 (v1.2) review" (correct), `atlas-agent/AGENTS.md` (references "v1.0 contract" or just contract), `atlas-console/AGENTS.md` (same)
**Problem:** The DECISIONS doc correctly labels the v1.2 additions. The API-CONTRACT doc title is v1.2 but the body header is v1.1 (see C1). The child AGENTS.md files don't pin a specific contract version (they say "the contract is `../docs/API-CONTRACT.md` (authoritative)"). When the contract is bumped, the child AGENTS.md files should be verified to match (per deepseek r2's "contract-bump checklist"). For v1.2, the changes are mostly additive (new response fields, new error semantics for /heartbeat) — the child skills are still compatible, but a contract-version-pinning line in the child AGENTS.md files would make drift detection easier.
**Recommendation:** Add to each child AGENTS.md header: "This repo implements `../docs/API-CONTRACT.md` at the version current at the start of the phase (currently v1.2). The skill files in `.claude/skills/` mirror the contract — if the contract is bumped, verify the skill's wire-shape examples still match." This is the "contract-bump checklist" deepseek r2 flagged, made into a doc rule.

### L2 — Dev seed script should cover all `flags` combinations so the dashboard is built against the full state matrix
**File:** `docs/PLAN.md` Phase 0 (dev seed script), `docs/ARCHITECTURE.md` §7 (liveness + flags)
**Problem:** Phase 0 specifies "3 sample devices, one with 24 h of intervals — all 3 states, 5+ domains incl. `null`, an interval crossing the Dhaka 18:00-UTC midnight, one >4 h, one <10 s." Good for the **interval** matrix. But the dashboard needs to render the **liveness + flags** matrix, which has 6 flags × 3 liveness values. Without seed data for each combination, the dashboard is built against `online + []` and the first time it sees `online + [revoked, clock_skewed, possible_clone]` it's untested.
**Recommendation:** Extend the Phase 0 seed spec: "Seed devices cover the full state matrix: (1) online + []; (2) online + [outdated]; (3) online + [clock_skewed] (with a `clockOffsetMs` of 7200000); (4) online + [revoked]; (5) online + [archived]; (6) online + [health_warning] (with high `cpuPct`/`memMb`); (7) online + [possible_clone] (with a `machineGuid` history of two values); (8) offline (lastSeenAt > threshold); (9) pending (enrolled, no heartbeat). Devices (4) and (5) should also have intervals (so the revoked/archived-but-capturing UX is testable; per M10). Use a fixed seed for reproducibility." The script grows from 3 to 9 devices — still trivial.

### L3 — First heartbeat after enrollment has `liveness: pending`; the contract example doesn't show this case
**File:** `docs/API-CONTRACT.md` §3 (response example), `docs/DECISIONS.md` #39
**Problem:** Same as M3 above but specifically for the first heartbeat. A freshly-enrolled device sends a heartbeat, the server creates the device doc with `liveness: pending, flags: []`, returns the response. The contract example shows `liveness: "online"` — the example is the post-first-heartbeat case. The first heartbeat case is the most important one to get right (the agent's very first interaction with the server after enrollment) and is the one the example doesn't show.
**Recommendation:** Add to contract §3 a "First heartbeat after enrollment" example with `liveness: "pending"`. Or add a sentence in the response behavior: "On the first heartbeat after enrollment (or after a long absence that the server has treated as pending), the response's `liveness` is `pending` (not `online`); the agent treats this as confirmation of enrollment. Subsequent heartbeats transition to `online`."

### L4 — `configSchemaVersion` semantics for the agent's *heartbeat* value vs the server's response value are asymmetric
**File:** `docs/API-CONTRACT.md` §1, §3, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §4
**Problem:** The contract §1 example shows `configSchemaVersion: 1` in the response. The contract §3 example shows `configSchemaVersion: 1` in the heartbeat **request** body (the agent's version). The contract §3 behavior says: "a `configSchemaVersion` newer than it supports → ignore-unknown + log." The asymmetry: the agent sends its own version (the one it COMPILES against), the server sends the current version (the one it IMPLEMENTS). A v1.0 agent (compiled with configSchemaVersion 1) heart-beating against a v1.2 server (sends configSchemaVersion 2): the agent ignores unknown fields; the server's `config_mismatch` flag (per H3) catches it. A v1.2 agent (compiled with configSchemaVersion 2) heart-beating against a v1.0 server (sends no `configSchemaVersion`): the agent uses its defaults. But the agent's heartbeat body sends `configSchemaVersion: 2` — the v1.0 server ignores it (it's an unknown field). No flag. The agent thinks it's running on a v1.2 server, the server doesn't know. Same as H3.
**Recommendation:** Same as H3 — server flags `config_mismatch` when it sees a heartbeat with a `configSchemaVersion` higher than the server supports. The "lower than" case is implicit (the agent handles it via ignore-unknown). Document both directions in the contract.

### L5 — `agentVersion` comparison semantics: 4-part .NET assembly versions vs 3-part semver
**File:** `docs/API-CONTRACT.md` §3 (`agentVersion: "0.1.0"`), `docs/API-CONTRACT.md` §4 (`minSupportedVersion: "0.1.0"`)
**Problem:** mimo r2 M5 flagged this for v1.1. v1.2 carries it forward. The contract example uses `0.1.0` (3-part). .NET's `AssemblyVersion` is 4-part (e.g., `0.1.0.0`). If the agent uses `AssemblyVersion`, the server-side `minSupportedVersion` comparison is wrong (4-part vs 3-part). If the agent uses `AssemblyInformationalVersion` (3-part + prerelease), the comparison is correct. Decision #47 says "`agentVersion` comparison uses semver on `AssemblyInformationalVersion`" — but this is in Decision #47 (auto-update manifest), not the contract. The heartbeat's `agentVersion` field is separate.
**Recommendation:** Add to contract §3 behavior: "`agentVersion` is a 3-part semver string with optional prerelease (e.g., `0.1.0`, `0.1.0-beta.1`, `1.0.0-rc.2`). The agent uses `AssemblyInformationalVersion` (not `AssemblyVersion`). The server compares with semver rules: prerelease < release, major.minor.patch. Build metadata is ignored." This is one paragraph and prevents a comparison bug when the agent is upgraded to a .NET version that defaults to 4-part.

### L6 — `lastSeenAt` shown alongside `liveness: offline` is a UX confusion waiting to happen
**File:** `docs/ARCHITECTURE.md` §7 (devices schema, `lastSeenAt` + `liveness`)
**Problem:** The device doc has `lastSeenAt` (source data) AND `liveness` (derived). The dashboard reads `liveness` for the badge. But the device card also shows "last seen 2 min ago" (a useful field). A device with `liveness: offline` (last heartbeat > 15 min ago) but `lastSeenAt` showing 16 min ago will display:

- Badge: "offline"
- Last seen: "16 min ago"
- Flags: [...]

The admin reading this sees a device that was seen 16 min ago and is now offline. Correct, but the relationship between "last seen" and "offline" is implicit (the admin has to know the 15-min threshold). The dashboard should make this explicit: "Last seen: 16 min ago (offline threshold: 15 min)".
**Recommendation:** Add to Phase 4: the device card shows "last seen: X ago (offline threshold: Y)" with both values visible. Or: change the badge from "offline" to "offline (seen 16 min ago)". This is a UX nit, not a spec bug.

### L7 — `flags` can be `null` or `undefined` in JSON, but the contract should require an array
**File:** `docs/API-CONTRACT.md` §3 (`flags: []`)
**Problem:** The example shows `flags: []` (an empty array). An implementer might serialize `null` when no flags are set (some JSON serializers do this by default) or omit the field entirely. The client should be required to always send an array, even if empty. Otherwise `flags === undefined` checks on the dashboard can silently pass when the server sends `null`.
**Recommendation:** Add to contract §3: "`flags` MUST be a JSON array (possibly empty). `null` or omission is a contract violation; the server should reject the response on serialization or the client should normalize `null` → `[]` on parse." One sentence.

### L8 — The `attributes` block in /heartbeat has `pcName, hostname, osUsername, timezone, machineGuid` but not `os` (which is a top-level field) — and the spec doesn't say `os` is also subject to the "first heartbeat + when changed" rule
**File:** `docs/API-CONTRACT.md` §3 (request body)
**Problem:** The heartbeat example has `os: "Windows 11 Pro 23H2"` at the top level and `attributes: { pcName, hostname, osUsername, timezone, machineGuid }` in a separate block. The behavior says: "First heartbeat after enrollment or restart sends `attributes`/`os` unconditionally; thereafter only when a value differs from the last sent (persisted in an `agent_state` row so a restart isn't treated as a baseline reset)." So both `os` and `attributes` follow the same rule. Good — but the rule is buried in a `>` blockquote after the example, easy to miss. The implementer might apply the rule only to `attributes` and ship `os` on every heartbeat (or never).
**Recommendation:** Move the "first + when changed" rule from a blockquote into the main §3 behavior text. Add a one-line example of the agent's `agent_state` schema (or the SQLite table): `{ lastSentOs, lastSentAttributes }` — a concrete target. The Phase 1b test (when heartbeat is built) should cover both the first-heartbeat and the changed-attribute paths.

---

## Nice-to-have

### N1 — Add `possible_clone` to the dashboard "device flags" filter so IT can find cloned devices in a fleet view
**File:** `docs/PLAN.md` Phase 4 (Company overview), `docs/ARCHITECTURE.md` §7
**Problem:** The decision is that 2+ distinct `machineGuid`s on one `deviceId` raises a `possible_clone` flag. The dashboard should be able to filter the company overview by this flag ("show me all devices that might be cloned"). The Phase 4 checklist doesn't enumerate the filter UI.
**Recommendation:** Add to Phase 4: "Company overview supports filter chips for each `flags` value (outdated / clock_skewed / revoked / archived / health_warning / possible_clone). Clicking a chip filters the device list." One paragraph, supports IT's clone-investigation workflow.

### N2 — Phase 2 chaos test: double /enroll, agent receives one token, then receives the other; verify self-heal
**File:** `docs/PLAN.md` Phase 2
**Problem:** Decision #43 specifies rotation on token loss. A chaos test for the race (network blip mid-enroll, two concurrent /enrolls) is not in Phase 2's checklist.
**Recommendation:** Add to Phase 2: "Chaos test: agent has valid token T1, fires /enroll, network blip before response, retries with T1. Server: T1 still valid → no-op (returns T1). Agent uses T1. No rotation. Repeat with: agent has no token (lost), fires /enroll, network blip, retries. Server: rotates to T2, then rotates to T3. Agent receives T2 response, uses T2, then receives T3 response, uses T3. Verify agent self-heals on 401 if it persists with T2. Verify the device's `deviceTokens` collection has T1 (superseded), T2 (superseded), T3 (active)."

### N3 — Phase 7 chaos test: long-offline + reconnect with a corrupted local buffer; verify no server-side duplicate
**File:** `docs/PLAN.md` Phase 7
**Problem:** Per H5 above, the `batches` TTL combined with a corrupted local buffer can produce real duplicates. The chaos test is in Phase 7, where 100-device soak and chaos testing is owned.
**Recommendation:** Add to Phase 7: "Chaos test: agent offline for 200 d, local SQLite buffer is corrupted (rows from 200 d ago are marked `synced = 0` but were actually sent). Agent comes back online, sends the corrupted rows with a fresh `batchId`. Server's `batches` receipt for the original batchId is gone (TTL evicted). Server inserts the new batch. Dashboard shows the activity twice. Verify the agent's `unsyncedRowRetention = min(bufferCapDays, serverBatchesTtl)` rule prevents this (the agent drops rows older than the server's TTL before sending)."

### N4 — Add a "first heartbeat after re-enroll" test to Phase 1b
**File:** `docs/PLAN.md` Phase 1b (test checklist)
**Problem:** The Phase 1b test list has: "call-vs-media gating, idle reset-on-input (no oscillation), bursty-input call stays active not idle, sleep-resume with media auto-resume, eTLD+1 normalization stub." Missing: the re-enroll flow's first heartbeat (the agent has a new token, old data in the buffer, must send a heartbeat that doesn't 401, then sync the buffer). The new agent behavior is well-defined by Decisions #43 + #45, but there's no unit test for the sequence.
**Recommendation:** Add to Phase 1b: "Test: agent loses token → re-enroll → receive new token → heartbeat returns 200 (not 401) → /ingest with buffered rows and the new token returns 200. Test: agent's first heartbeat after re-enroll has `liveness: online, flags: []` (re-enroll does not set any flags unless the admin un-revoked concurrently)."

### N5 — Document the agent's behavior on WFH-via-RDP days (no helper, no intervals, dashboard shows "no console session")
**File:** `docs/DECISIONS.md` #32, `docs/ARCHITECTURE.md` §2.1, `docs/PLAN.md` Phase 1b
**Problem:** minimax r2 M6 noted that WFH-via-RDP days produce zero intervals (helper doesn't run in the disconnected console session), and the dashboard shows the employee as "idle" with no flag. This is correct behavior but should be tested and documented at the implementation level. Phase 1b's exit gate mentions calls, idle, sleep, but not WFH.
**Recommendation:** Add to Phase 1b: "Test: simulate WFH-via-RDP — service has no console session with a real user (the console session is locked or the user is RDP'd in). Helper is not running. /heartbeat succeeds (service is alive), `capture.helperRunning: false`. Dashboard renders the device as 'online' (service alive) with the capture-health panel greyed out (helper not running)."

---

## Where v1.2 Is Correct (do not relitigate)

These v1.2 changes correctly address r1/r2 findings. Calling them out so future agents don't re-litigate:

- **Decision #28 (amended) — dedup uniqueness lives only on `batches`, not `intervals`.** The unique-index bug from v1.1 is fixed. The TTL of ~180 d bounds the collection. The r2 glm C1 / qwen H1 / mimo C2 are all resolved.
- **Decision #29 (amended) — atomic ingest with `maxCommitTimeMS` and `TransientTransactionError` retry, single-node RS bound to 127.0.0.1.** The 60-s transaction timeout and the Next.js function timeout can race (deepseek r2 H3), but the data-integrity story is correct. The 127.0.0.1 binding prevents the "not primary" failure mode.
- **Decision #33 (amended) — `SERVICE_CONTROL_POWEREVENT` for suspend (not `PowerModeChanged` which needs a message pump); monotonic sanity force-close for missed events; post-resume state re-derived from next sample.** The "media auto-resumes on wake" path is handled correctly. The "VM / Modern Standby" path is handled by the monotonic sanity check. r2 kimi C2, mimo H1 are resolved.
- **Decision #34 (amended) — entry-debounced, exit-immediate; media transitions bypass hysteresis.** Fixes the v1.1 "call recorded as idle" bug (r2 glm H4). My H1 flags the opposite over-correction, but the underlying state-machine model is right.
- **Decision #37 (amended) — `osUsername` snapshot never back-filled; `employeeId` re-resolvable.** The two fields have different back-fill rules, and the r2 contradiction (minimax H8 vs qwen H2 vs mimo L10) is resolved by splitting the rules.
- **Decision #38 (amended) — org key validated first (`401`), then block check (`403`).** Prevents an unauthenticated caller from probing block status. The r2 qwen L3 finding is resolved.
- **Decision #39 (amended) — `liveness + flags` instead of a single enum.** r2 glm M3 / minimax H4 are addressed. My H2 flags the source-of-truth drift risk, but the model itself is correct.
- **Decision #40 (amended) — device IANA tz captured, `employees.tzOverride` for non-Dhaka hire, clock-skew offset applied before bucketing.** r2 glm H2, minimax H2, kimi C1 are all resolved.
- **Decision #42 (amended) — `machineGuid` also sent in heartbeat (clone detection).** r2 mimo M8 is resolved. My M8 flags a path inconsistency (enroll vs heartbeat) but the clone-detection logic is correct.
- **Decision #43 — token-loss re-enroll rotates.** The v1.1 contradiction (server can't return hashed token) is resolved. r2 glm H1 is closed.
- **Decision #44 — `/enroll` rate limit = 10/min/IP + corp-CIDR allowlist.** r2 glm H3, kimi H7 are resolved.
- **Decision #45 — revoked device stays visible via `/heartbeat`; `reauthorized` self-heal.** r2 kimi C4, deepseek L4 are resolved. My M4 flags the `reauthorized` lifecycle as ambiguous, but the recovery lifeline is correct.
- **Decision #46 — named-pipe anti-spoofing: per-instance UUID handshake + peer verification (PID → image path → code signature).** r2 kimi C3 is resolved.
- **Decision #47 — `426` only for `/enroll` and `/ingest`; `/heartbeat` always 200 with `contractDeprecated` + compiled-in fallback `manifestUrl`.** r2 glm M6 is resolved.
- **Decision #48 — "forget device" atomically: revoke + block + delete intervals + delete batches receipts.** r2 kimi H2, mimo C2 are resolved. My M1 flags the missing API endpoint, but the correctness of the action is right.
- **Phase 1a/1b split.** The split is right (r2 qwen M5, mimo sequencing). My H4 flags the exit-gate wording, but the split itself is correct.
- **Data model additions: `dataGaps` collection, `liveness`+`flags` device fields, `possible_clone` flag, `tzOverride` on `employees`, `health_warning` flag.** All consistent with the v1.1/r2 review consensus.
- **Contract v1.2 adds: `liveness`/`flags`/`reauthorized`/`contractDeprecated` in heartbeat response, `attributes.timezone`/`attributes.machineGuid` in heartbeat, `dropped.byState` in heartbeat, `capture.helperStartedAtUtc` in heartbeat, `configSchemaVersion` in heartbeat body, `timezone` in enroll body.** All consistent with the decisions.
- **`archive` retains data; `revoke` blocks ingest/enroll; `forget` deletes data.** Three distinct admin actions, each with a clear meaning. The triple is well-specified.

---

## Phase 0 Readiness

Phase 0 is **mostly ready** to start. The v1.2 changes resolve the r1/r2 criticals. The remaining Phase 0 items are paperwork:

1. **Fix the v1.1 → v1.2 wire header inconsistency** (C1) — 5-minute doc edit. **Before Phase 0 starts.**
2. **Pin contract version in child AGENTS.md files** (L1) — one-line addition per child repo. **Before Phase 0 starts.**
3. **Add `missing` from `liveness` and `possible_clone` from `flags` examples** in contract §3 (M3) — doc edit. **Before Phase 0 starts.**
4. **Add the full state-matrix to the seed script** (L2) — script edit, 6 more devices. **Before Phase 0's dashboard work begins.**
5. **Specify log-rotation policy, config defaults, dev org-key handling** — already in r2 readiness notes; not changed by v1.2. Carry over.

No blockers. The v1.2 changes make Phase 0 more, not less, unblocked.

---

## Phase Ordering — Phase 1a/1b is the right split

The Phase 1a/1b split is correct. Helper + transport + foreground sampling is genuinely independent of the state machine + aggregator. My H4 flags the exit-gate wording ("Phase 2 may now begin in parallel") as overstated — only the **server-side** Phase 2 work can be parallelized; the agent-side Phase 2 work needs 1b's intervals. A two-line clarification on the 1a exit gate (or, alternatively, splitting Phase 2 into 2a/2b symmetrically) prevents a Phase 2 split that has to be undone.

Otherwise, the sequencing is right. Phase 2 still owns revocation end-to-end (as kimi r2 M6 noted was misplaced in Phase 7). The chaos tests in Phase 7 are well-scoped.

---

## Summary by Severity

| Severity | Count | Key themes |
|---|---|---|
| Critical | 1 | Wire header `X-Atlas-Contract: 1.1` in v1.2 doc body |
| High | 5 | Opposite idle over-correction; liveness+flags vs source fields drift; `configSchemaVersion` mismatch still dropped; Phase 1a exit gate overstates parallelism; batches TTL + corrupted buffer = duplicates |
| Medium | 12 | Forget-device endpoint missing; `capture.enabled` missing; contract missing `pending`/`possible_clone`; `reauthorized` lifecycle ambiguous; `health_warning` threshold unquantified; `contractDeprecated` trigger unspecified; `dataGaps` no dashboard surfacing; `machineGuid` path inconsistent; re-enroll cap missing; `archived` agent-side unspecified; revoked-token probe surface undocumented; liveness+flags transitions not enumerated |
| Low | 8 | Version-pinning in child AGENTS.md; seed-script state matrix; first-heartbeat `pending` example; configSchemaVersion asymmetry; agentVersion comparison; `lastSeenAt` UX; `flags` array requirement; `os` first-heartbeat rule |
| Nice-to-have | 5 | Possible-clone filter; double-rotation chaos; long-offline chaos; re-enroll first-heartbeat test; WFH-via-RDP test |

**Before Phase 0 starts:** C1 (wire header), L1 (child AGENTS pinning), M3 (contract examples), L2 (seed matrix).
**Before the phase that owns them:** H1 (Phase 1b), H2 (Phase 2), H3 (Phase 2), H4 (Phase 1a exit gate, 1-line fix), H5 (Phase 2), M1 (Phase 4), M2 (Phase 2 / Phase 4), M4 (Phase 2), M5 (Phase 2), M6 (Phase 6 / settings), M7 (Phase 4), M8 (Phase 0), M9 (Phase 2), M10 (Phase 0 / Phase 4), M11 (Phase 0 docs), M12 (Phase 2), L3–L8 (their respective phases).
**Not blockers:** Nice-to-haves.

---

*End of review. No code or existing documents were modified; only this file was created.*
