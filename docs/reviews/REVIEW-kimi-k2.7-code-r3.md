# Atlas Green — Pre-Implementation Design Review (Round 3)

**Model:** kimi-k2.7-code (opencode-go/kimi-k2.7-code)  
**Date:** 2026-06-23

## Executive Summary

v1.2 cleanly resolves the contradictions that iteration 2 found: token-loss rotation replaces the hashed-token "return existing" impossibility; dedup uniqueness moves exclusively to `batches`; the enroll rate limit is rollout-safe; idle exit is now input-driven; and device state is split into liveness + orthogonal flags. The remaining risks are mostly new edge cases exposed by those fixes: the contract still advertises version `1.1` in its own header; the `batches` TTL can fall behind a lengthened buffer cap and allow duplicate replay; "forget device" does not purge the agent's buffer; Phase 2 starting after Phase 1a is unrealistic without state/duration-bearing intervals; token rotation can mint orphan valid tokens; a revoked token can still heartbeat and probe config; and "any input exits idle" flips to the opposite false-positive failure mode. Phase 0 is now fully unambiguous and ready to start, but a couple of flag/field redundancies need tightening before coding begins.

---

## Critical

### C1 — "Forget device" deletes server-side data but does not purge the agent's local buffer, allowing "forgotten" data to return
**Files:** `docs/DECISIONS.md` #48; `docs/API-CONTRACT.md` §1 (enrollment rotation), §5 (revoked-terminal flow)
**Problem:** Decision #48 atomically revokes the token, sets `blocked`, and deletes the device's `intervals` and `batches` receipts. The agent, however, still holds unsynced rows in its SQLite buffer. If the device is later unblocked and re-enrolls (or if the admin simply un-revokes it), the agent will resume syncing and re-submit batches whose receipts were deleted. The server accepts them as new batches (no dedup) and the "forgotten" intervals reappear. This undermines a legal-erasure workflow.
**Recommendation:** After a "forget device" action, the device must remain **blocked** (or be assigned a new `deviceId`) so it cannot re-enroll with the same identity. If unblocking is required, the agent must first receive a `purge_buffer` directive (e.g., the next heartbeat from a re-enrolling device returns `purgeBuffer:true` before any ingest is accepted). Document that "forget device" is irreversible for the same `deviceId`.

### C2 — The `batches` TTL (~180 d) can become shorter than a custom `bufferCapDays`, allowing duplicate replay
**Files:** `docs/DECISIONS.md` #28; `docs/ARCHITECTURE.md` §7 (`settings.retention`); `docs/API-CONTRACT.md` §1 (`bufferCapDays` range)
**Problem:** `batchReceiptDays` defaults to 180, while `bufferCapDays` can be set anywhere from 7 to 365. If an admin raises `bufferCapDays` above 180, an agent that is offline for 200 days can still hold a buffered batch whose receipt has TTL-expired. On reconnect, the server has no dedup record and re-inserts the intervals, duplicating any rows that were already synced before the outage.
**Recommendation:** Make `batchReceiptDays` a function of `bufferCapDays`, not a fixed constant: `batchReceiptDays = max(180, bufferCapDays + 45)`. If you want a configurable TTL index, store an `expireAt = receivedAt + batchReceiptDays` field per receipt instead of a fixed TTL on `receivedAt`. Add a Phase 2 integration test that verifies a 200-day-offline agent with `bufferCapDays=365` does not duplicate on reconnect.

---

## High

### H1 — The API contract still advertises version `1.1` in its own `X-Atlas-Contract` header and intro text
**Files:** `docs/API-CONTRACT.md` line 32 ("`X-Atlas-Contract: 1.1`"); lines 8–17 (header still says "v1.1 … folded in" / "v1.2 … corrects")
**Problem:** The document is labeled v1.2, but the header example and the version echoed by the server still say `1.1`. The agent also sends `X-Atlas-Contract: 1.1`. Any version-mismatch logic built against this header will be wrong from day one, and the contract-bump checklist will be impossible to verify.
**Recommendation:** Update every `X-Atlas-Contract` example in the file to `1.2`, and add a release checklist item: "When bumping `API-CONTRACT.md`, search the file for the old version string and update all examples, headers, and `contractDeprecated` copy. Verify both child `AGENTS.md` files no longer reference the old version."

### H2 — Phase 2 cannot realistically begin after Phase 1a because 1a does not produce state/duration-bearing intervals
**Files:** `docs/PLAN.md` Phase 1a, Phase 1b, Phase 2
**Problem:** Phase 1a builds the helper, named pipe, and raw foreground sampling, and its exit gate only requires "foreground samples reach the service." Phase 2 then builds the SQLite buffer and `/ingest` sync. But Phase 2's exit gate requires "intervals captured offline flush to MongoDB" — i.e., closed intervals with `state`, `durationSec`, `startUtc`/`endUtc`. Those are only produced by the aggregator in Phase 1b. Starting Phase 2 in parallel is therefore only scaffolding work; the integration point cannot be exercised until 1b finishes.
**Recommendation:** Either (a) extend Phase 1a's exit gate to require the service to close at least one minimal interval (e.g., `state: active`, `durationSec` from the monotonic clock, no idle/media logic) so Phase 2 has a real row shape to store and sync, or (b) remove the "Phase 2 may now begin in parallel" wording and make 1b a hard prerequisite for 2. Keep the 1a/1b split for risk reduction, but don't pretend 2 can complete before 1b.

### H3 — Token-loss rotation can mint orphan valid tokens when a re-enroll response is lost
**Files:** `docs/API-CONTRACT.md` §1; `docs/DECISIONS.md` #43
**Problem:** A re-enroll without a valid token rotates: the server marks the old token `supersededAt` and mints a new one. If the HTTP response containing the new token is lost, the agent retries and rotates again. The first minted token was valid but never delivered to the agent. An attacker who captured that first response now holds a valid, unrevoked token for the device.
**Recommendation:** Include a client-generated `enrollNonce` (UUID, single-use) in re-enroll requests. The server records the nonce for ~10 minutes and returns the same token for the same `{deviceId, enrollNonce}` pair, so a retry does not mint a fresh token. Or, more simply: keep only the most recently returned token active and expire any token that is not used within 24 hours of minting.

### H4 — `/heartbeat` accepting a revoked token still exposes config, manifest URL, and liveness updates
**Files:** `docs/API-CONTRACT.md` §3; `docs/DECISIONS.md` #45; `docs/ARCHITECTURE.md` §6
**Problem:** A leaked revoked token cannot ingest or re-enroll, but it can still call `/heartbeat` and receive `config`, the `update` block (manifest URL), and update `lastSeenAt`/`liveness`. An attacker can therefore keep a revoked device appearing `online` and learn the manifest URL / current config. This does not violate data integrity, but it weakens the "revoked means dead" intent and gives a leaked token ongoing value.
**Recommendation:** For revoked tokens, return `200` with `flags:["revoked"]` and `liveness: "offline"` (do not update `lastSeenAt`), and omit or minimize the `config`/`update` blocks (server time only). The device remains visible as revoked, but the token cannot influence online/offline status or probe fleet configuration.

### H5 — Idle "any input exits" flips to the opposite failure mode: phantom input events keep the user falsely active
**Files:** `docs/DECISIONS.md` #34; `docs/ARCHITECTURE.md` §4; `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §3
**Problem:** v1.2 correctly fixed the "call recorded as idle" bug by leaving idle on any input. But a single stray USB polling event, RDP keepalive, desk vibration on a mouse, or a Windows input-injection can now reset the 300-second idle timer. If such events recur every few minutes, the user stays `active` indefinitely even though they are away. This is the mirror image of the old problem.
**Recommendation:** Add a small debounce on idle exit: require input in **two consecutive samples** (or at least one input event within a 10-second window) before leaving idle. The 300-second entry debounce still prevents oscillation. Document that this is a trade-off: a brief genuine touch is counted, but isolated phantom events are not.

### H6 — `archived` flag behavior is undefined: does it block ingest, hide the device, or both?
**Files:** `docs/ARCHITECTURE.md` §7 (`archived` flag); `docs/API-CONTRACT.md` §5 (revoked-terminal flow)
**Problem:** The liveness+flags model adds `archived` as a flag, but the plan never says whether an archived device continues to ingest, whether `/enroll` rejects it, or whether the agent should enter the revoked-terminal state. If archived only hides the dashboard row while ingest continues, the admin has no way to stop a lost/archived machine from appending data. If archived blocks ingest, the agent's behavior is unspecified.
**Recommendation:** Define `archived` as ingest-blocking (401 on `/ingest`, 403 on `/enroll`) but heartbeat-accepted (200 with `flags:["archived"]`), exactly like `revoked`. The difference is administrative: archived devices are hidden from default dashboard views but can be un-archived; revoked devices are security incidents. Add this to the contract §5 and the console skill.

---

## Medium

### M1 — `reauthorized:true` is fragile if the agent misses the heartbeat that carries it
**Files:** `docs/API-CONTRACT.md` §3 (heartbeat response)
**Problem:** The server sends `reauthorized:true` "once" after an admin un-revoke. If that heartbeat fails or is rejected by a transient 5xx, the agent stays in its revoked state (stopped helper, heartbeat-only) until it sees the flag. There is no retry or standing signal.
**Recommendation:** Send `reauthorized:true` on every heartbeat where the token is valid and the device is no longer revoked, until the agent demonstrates that it has resumed capture (e.g., by sending a successful `/ingest`). Alternatively, have the agent treat any heartbeat 200 that lacks the `revoked` flag as permission to resume, removing the one-shot dependency.

### M2 — `blocked` boolean in the `devices` schema is redundant with the `revoked` flag
**Files:** `docs/ARCHITECTURE.md` §7 (`devices` schema)
**Problem:** The schema lists both `blocked: bool` and `revoked` in `flags`. The `/enroll` block check can use `deviceTokens.revokedAt` or the `revoked` flag; the extra `blocked` field introduces a second source of truth. An admin could set one without the other, leading to a device that is flagged revoked but can still enroll, or vice versa.
**Recommendation:** Drop `blocked` from the `devices` schema. Treat the `revoked` flag (and the most recent non-superseded `deviceTokens.revokedAt`) as the single authority for enrollment blocking. Keep `archived` as a separate flag.

### M3 — The `batches` TTL index cannot honor a runtime change to `batchReceiptDays`
**Files:** `docs/ARCHITECTURE.md` §7 (`batches` collection, `settings.retention`)
**Problem:** The plan says `batches` has a TTL index on `receivedAt` (~180 d), and `settings.retention.batchReceiptDays` is 180. A MongoDB TTL index has a fixed `expireAfterSeconds`; changing `batchReceiptDays` does not affect already-inserted receipts until the index is rebuilt.
**Recommendation:** Use a per-document `expireAt = receivedAt + batchReceiptDays` field and a TTL index on `expireAt`. When `batchReceiptDays` changes, future receipts get the new `expireAt`; existing receipts keep their original expiry. Document that receipt retention is point-in-time.

### M4 — Phase 0 now depends on Cloudflare Access, which may not be provisioned before coding starts
**Files:** `docs/PLAN.md` Phase 0; `docs/ARCHITECTURE.md` §6
**Problem:** Phase 0 requires "Cloudflare Access + middleware" with a dev bypass. The production Cloudflare Access setup (domain, application, JWT keys) is an external dependency. If it is not ready, the Phase 0 exit gate "an unauthenticated browser cannot view `/dashboard`" can only be verified via the dev bypass, not the real edge.
**Recommendation:** Split the Phase 0 checklist: (a) middleware with dev bypass works, and (b) production Cloudflare Access is wired. Gate Phase 0 completion on (a); gate Phase 0 deployment on (b). Document the external prerequisites so Claude does not block on DNS/cert setup.

### M5 — Code-signature verification of the helper assumes the signing cert never changes in v1
**Files:** `docs/ARCHITECTURE.md` §2.2; `docs/DECISIONS.md` #18
**Problem:** The service verifies the helper by checking that its executable is signed by the agent code-signing certificate. If the self-signed cert is ever rotated (leak, expiry, upgrade to an OV cert), existing agents will reject helpers signed with the new cert, and updated agents will reject old helpers during a staggered rollout.
**Recommendation:** Document that the same certificate must sign both service and helper for the entire v1 fleet. If cert rotation is ever needed, the rotation must be bundled with an agent auto-update that ships the new public key and accepts both old and new signatures during a transition window.

### M6 — `possible_clone` flag can false-positive on legitimate machineGuid changes
**Files:** `docs/DECISIONS.md` #42; `docs/ARCHITECTURE.md` §7
**Problem:** A Windows rearm, Sysprep, or certain disk-cloning repairs can legitimately change `MachineGuid`. The server will flag the device as `possible_clone`, but the dashboard has no workflow to inspect the old/new GUIDs or dismiss the flag.
**Recommendation:** When `possible_clone` is set, the dashboard must show the sequence of reported `machineGuid`s with timestamps and offer an admin action "Dismiss clone warning" (audit-logged). Otherwise the flag becomes noise and will be ignored.

### M7 — The agent still does not send its `configSchemaVersion` to the server in a way the server can act on mismatches
**Files:** `docs/API-CONTRACT.md` §3 (heartbeat request `configSchemaVersion`)
**Problem:** The heartbeat request includes `configSchemaVersion: 1`, but the contract does not say what the server does if it sends config fields the agent does not understand. An agent that ignores unknown config silently deviates from admin intent.
**Recommendation:** If the server sends a `configSchemaVersion` greater than the agent's supported version, the server should set a flag (e.g., `config_mismatch`) in the heartbeat response and dashboard. This is in addition to the existing `outdated` flag driven by `agentVersion`.

---

## Low

### L1 — Console skill and contract still use old `status` terminology in places
**Files:** `docs/API-CONTRACT.md` §5 ("`status` reflects `revoked`"); `atlas-console/.agents/skills/atlas-api-contract/SKILL.md` §1 ("updates `lastSeenAt`/`status`")
**Problem:** v1.2 replaced the single `status` enum with `liveness` + `flags`. A few sentences still refer to the old `status` field, which will mislead implementers.
**Recommendation:** Replace "`status`" with "`liveness` and `flags`" everywhere in the contract and console skill. Add a glossary note: the old `status` enum no longer exists.

### L2 — `health_warning` thresholds exist only in the architecture doc, not the contract or skill
**Files:** `docs/ARCHITECTURE.md` §7; `docs/API-CONTRACT.md` §3
**Problem:** The `health_warning` flag is defined in the data model but never in the heartbeat contract or agent skill. Without explicit thresholds, two implementers will choose different values.
**Recommendation:** Add to the agent skill §2 and the contract §3: `health_warning` is set by the server when `cpuPct > 5` sustained across 3 heartbeats or `memMb > 200`.

### L3 — The `batches` schema mixes `_id = batchId` with a redundant `{deviceId, batchId}` unique index
**Files:** `docs/ARCHITECTURE.md` §7 (`batches` schema)
**Problem:** The schema says `_id: batchId` and also "unique on `{deviceId, batchId}`". Because `_id` is already globally unique, the compound unique index is redundant and confusing.
**Recommendation:** Either (a) keep `_id` as an auto-generated `ObjectId` and make `{deviceId, batchId}` the unique index, or (b) keep `_id = batchId` and replace the compound unique with a `{deviceId, receivedAt}` index for purge queries.

---

## Nice-to-have

### N1 — Add a contract test fixture in the mother repo
A small shared test (e.g., `tests/contract/`) that exercises `/enroll`, `/ingest`, and `/heartbeat` against a running console would catch agent↔console drift before release. Not blocking.

### N2 — Localhost-only agent diagnostics endpoint
A `http://127.0.0.1:<port>/diag` endpoint bound to localhost would help IT debug without reading log files. Keep it silent and unauthenticated only on loopback.

---

## Where v1.2 is already correct and needs nothing

These v1.2 changes correctly address the iteration-2 findings and should not be relitigated:

- **Token-loss rotation (Decision #43).** Fixes the hashed-token "return existing" impossibility without breaking buffered rows.
- **Dedup uniqueness lives only on `batches`; `intervals` `{deviceId,batchId}` is non-unique (Decision #28).** Fixes the multi-interval-batch rejection bug.
- **Enroll rate limit = 10/min/IP + corp-CIDR allowlist (Decision #44).** Prevents the 100-hour rollout problem.
- **Idle entry debounced, exit on any input (Decision #34).** Correctly reconciles idle hysteresis with call participation.
- **Active comms call overrides concurrent non-comms media (Decision #7).** Prevents Teams+Spotify from being labeled `passive_media`.
- **Device state = liveness + flags (Decision #39).** Allows online + outdated + clock_skewed to coexist.
- **Heartbeat never returns 426; fallback manifest URL (Decision #47).** Prevents contract-bump bricking.
- **Service `SERVICE_CONTROL_POWEREVENT` + monotonic sanity + sub-30s power-blip debounce (Decision #33).** Correctly handles Session-0 services and BD power blips.
- **Post-resume state re-derived from next sample (Decision #33).** Fixes the hardcoded-idle-on-resume bug.
- **Device timezone capture + `employees.tzOverride` + clock offset before bucketing (Decision #40).** Fixes non-Dhaka hire bucketing and skewed-device day boundaries.
- **Helper pipe anti-spoofing + `(instanceId, seq)` dedup (Decision #46).** Closes the standard-user injection path and the restart seq-reset bug.
- **Crash-safe update swap with startup marker recovery (API-CONTRACT §4 / ARCHITECTURE §9).** Handles crash mid-update.
- **Audit log infra, dev auth bypass, dev org key, seed spec, and health endpoint in Phase 0.** Phase 0 is now fully unambiguous.

---

*End of review. Only this file was created; no other documents were modified.*
