# Atlas Green — Iteration 3 Design Review

**Model:** mimo-v2.5-pro
**Date:** 2026-06-23
**Round:** 3 (v1.2 stress-test)

## Executive Summary

v1.2 absorbed iteration 2 well — token-loss rotation, dedup-uniqueness-on-batches-only, liveness+flags, idle-exit-on-any-input, pipe anti-spoofing, heartbeat-never-426, and forget-device atomicity are all real improvements. The plan is sound and Phase 0 can start. However, v1.2 introduced new edge cases: (1) a leaked revoked token can still spoof capture health via `/heartbeat`, defeating the "agent goes silent = flag" backstop; (2) the `reauthorized` self-heal path after un-revoke depends on a possibly-rotated org key when the existing token is already valid; (3) which 401 triggers the revoked-terminal flow vs re-enroll-as-new-device is ambiguous between two contract sections; (4) pipe peer code-signature verification in Phase 1a blocks on a cert that doesn't exist until Phase 5/6. Several medium-severity issues cluster around the liveness+flags model: `archived` flag behavior is undefined, `blocked`/`archived` booleans coexist with `flags` without a sync rule, and flag transition/co-existence rules are absent.

---

## High

### H1 — Revoked-token `/heartbeat` can still spoof capture health, defeating the "silent agent = flag" backstop

**Files:** `docs/DECISIONS.md` #45; `docs/API-CONTRACT.md` §3; `docs/ARCHITECTURE.md` §6
**Problem:** Decision #45 correctly accepts a revoked token on `/heartbeat` so the device shows as `revoked` rather than silently "offline." But the contract doesn't restrict *what* a revoked-token heartbeat may update. The server currently updates `lastSeenAt` (liveness → `online`), `capture.helperRunning`, `capture.lastSampleUtc`, and all other fields. An attacker who extracted the token before revocation can send heartbeats with `helperRunning: true, lastSampleUtc: now, helperRestartCount: 0` → the device shows `liveness: online` with healthy capture — indefinitely. This defeats Decision #17's backstop: "an agent that goes silent is itself a flag." The activity record is safe (`/ingest` is blocked), but the liveness/tamper signal is compromised.

**Fix:** For a revoked-token heartbeat: update `lastSeenAt` (so the device shows online+revoked — the intent of #45), but **freeze** `capture` health fields, `helperRunning`, `health_warning`, and `possible_clone` at their last non-revoked values. The server should not accept new capture health from a revoked token. Better: the dashboard renders `revoked` as the **primary badge** overriding `liveness`, so "online + revoked" displays as "revoked" — the admin's question is "is it revoked?" not "is it online?" State both in `ARCHITECTURE.md §6`/`§7` and the dashboard rendering rule.

### H2 — `reauthorized` self-heal recovery path depends on a possibly-rotated org key; the existing token is already valid

**Files:** `docs/API-CONTRACT.md §3` (lines 228, 263); `docs/DECISIONS.md` #45
**Problem:** The contract says on `reauthorized:true`, the agent "exits its revoked state and resumes (re-enrolls for a fresh token/config)." But re-enrollment requires the org key from the MSI. If the org key has been **rotated** since deployment (the transition window closed, the old key was revoked), the agent's re-enroll gets `401` — it can't recover. But its **existing token is already valid again** (the admin un-revoked it). Requiring re-enroll introduces a stale-org-key dependency that un-revoke doesn't actually need. Additionally, `reauthorized` is "true once" — if the agent misses that single heartbeat (network blip, service restart), the flag is gone on the next cycle and the agent stays in its self-imposed revoked state forever.

**Fix:** (a) On `reauthorized:true`, the agent should first **resume `/ingest` directly** with its existing (now valid) token; only if `/ingest` returns `401` should it fall back to re-enroll. (b) Send `reauthorized:true` on **every** heartbeat while the device is un-revoked **until the server observes the agent acting on it** (a successful `/ingest`), not "once." State both in §3 Behavior and the agent skill §4.

### H3 — Which 401 triggers revoked-terminal vs re-enroll-as-new-device is ambiguous between two contract sections

**Files:** `docs/API-CONTRACT.md §3` (line 248–249) vs `§5` (line 342–345)
**Problem:** §3 says a `401` from `/heartbeat` (token↔deviceId mismatch) means "the agent re-enrolls." §5 says "on a revoked/invalid device token the agent attempts a single re-enroll… if `/enroll` returns `403` (blocked device) or re-enroll fails repeatedly, the agent stops generating new intervals, keeps its buffer, and only heartbeats (status reflects `revoked`)." These describe **different** 401 cases with **different** recovery paths: §3's 401 is a mismatch/deleted-device (re-enroll as a new device, new identity), while §5's 401 is a revoked token (enter terminal state, keep identity). An implementer who applies §5's "stop capturing, heartbeat only" to §3's mismatch case will never re-enroll. An implementer who applies §3's "re-enroll" to §5's revoked case will create a new device identity, orphaning the old one.

**Fix:** Split explicitly: "On a `401` from **`/ingest`** (token revoked or invalid), the agent attempts a single re-enroll with the org key; if `403` → revoked-terminal state. On a `401` from **`/heartbeat`** (token↔deviceId mismatch, device doc deleted, or token superseded by a rotation) or `404` from `/heartbeat` (device doc deleted), the agent re-enrolls as a **new** device (new identity). On a `404` from `/ingest` (device doc deleted between heartbeats), the agent re-enrolls once." Add to §3 Behavior and §5 Auth failures.

### H4 — Phase 1a pipe peer code-signature verification (#46) blocks on a signing cert that doesn't exist until Phase 5/6

**Files:** `docs/DECISIONS.md` #46; `docs/ARCHITECTURE.md` §2.2; `docs/PLAN.md` Phase 1a
**Problem:** Decision #46 specifies that the service verifies the helper peer via `GetNamedPipeClientProcessId` → `QueryFullProcessImageName` → confirm it's the installed helper exe and **code-signed by the agent cert**. Phase 1a includes this pipe verification. But the self-signed code-signing cert is created in Phase 5 (#18) and the CI pipeline that signs binaries is Phase 6. During Phase 1a development, the helper runs **unsigned** — the signature check will reject it, and the implementer must either skip the check (defeating #46) or create a throwaway dev cert (unspecified).

**Fix:** Add to Phase 1a or the agent skill: "In **dev mode** (running as a console, not a service), the pipe peer signature check is **skipped** — only the image-path check is enforced. In prod (Windows Service), the full signature check is enforced. A dev-mode flag (`DEV_SKIP_SIGNATURE_CHECK=1` or a build-time `#if DEBUG` guard) controls this." Document that the production signing cert is created in Phase 5 and the full verification is validated in Phase 5/6 testing.

---

## Medium

### M1 — `archived` flag's ingest/agent behavior is undefined

**Files:** `docs/ARCHITECTURE.md §7` (`archived` flag); `docs/API-CONTRACT.md §2`/`§5`; `docs/DECISIONS.md` #21
**Problem:** The liveness+flags model includes `archived` as a flag, and Decision #21 mentions "manual per-PC archive/delete by admin." But the contract never says whether an archived device's `/ingest` is accepted (401) or rejected, whether `/enroll` rejects it (403), whether `/heartbeat` returns `flags:[archived]`, or whether the agent should enter the revoked-terminal state. If archived only hides the dashboard row while ingest continues, the admin has no way to stop a lost/archived machine from appending data.

**Fix:** Define `archived` as ingest-**blocking** (401 on `/ingest`, 403 on `/enroll`), heartbeat-accepted (200 with `flags:[archived]`), same agent flow as revoked — stop capturing, keep buffer, heartbeat-only. The difference from `revoked` is administrative: archived = hidden but recoverable; revoked = security incident. Un-archive clears the flag and the agent resumes on the next `reauthorized:true`. Add to `API-CONTRACT.md §5` and `ARCHITECTURE.md §7`.

### M2 — `devices` doc still carries `blocked: bool` + `archived: bool` alongside `liveness` + `flags` — dual source of truth, no sync rule

**Files:** `docs/ARCHITECTURE.md §7` (devices shape, line 238–241)
**Problem:** Decision #39 moved device state to `liveness` + `flags` (`revoked`, `archived`, etc.) and says "the dashboard reads liveness for the badge, renders flags as chips — never re-derives from `lastSeenAt`/`revokedAt` separately." But the `devices` schema still lists `blocked: bool` and `archived: bool` as separate fields alongside `flags`. So there are two representations of "revoked" (`devices.blocked` + `flags.revoked` + `deviceTokens.revokedAt`) and "archived" (`devices.archived` + `flags.archived`). The spec never says which is authoritative or that they're set atomically. An admin could set one without the other.

**Fix:** Pick one. Recommended: keep `blocked` and `archived` booleans as the **stored inputs** and document that `flags.revoked`/`flags.archived` are **derived from them** at write time and stored denormalized for dashboard reads; a revoke action atomically sets `blocked=true` + pushes `revoked` into `flags` + sets `deviceTokens.revokedAt` in one update. Or remove the booleans and make `flags` the sole store. State the relationship and the atomic-set requirement in `ARCHITECTURE.md §7` and the console skill §5.

### M3 — Flag transition and co-existence rules are absent

**Files:** `docs/DECISIONS.md` #39; `docs/ARCHITECTURE.md §7`
**Problem:** Decision #39 lists flags (`outdated`, `clock_skewed`, `revoked`, `archived`, `health_warning`, `possible_clone`) but specifies no transitions or co-existence rules. Open questions: can `revoked` and `archived` co-exist? Is `revoked` terminal (clearable only by admin un-revoke)? Does `outdated` clear when the agent updates? Does `clock_skewed` clear when the clock corrects? Does `health_warning` clear when cpu/mem normalize? Does `possible_clone` ever clear? Can `archived` be set on a `pending` device?

**Fix:** Add a short transition/co-existence table to `ARCHITECTURE.md §7`: which flags are terminal vs clearable and by what event; which may co-exist (`outdated` + `clock_skewed` + `health_warning` + `possible_clone` are non-terminal badges that may stack; `revoked` + `archived` may co-exist with `revoked` taking dashboard precedence); liveness is independent of all flags. ~10 lines of doc; prevents Phase 4 rework.

### M4 — `pcName` ownership is contradictory between contract sections

**Files:** `docs/API-CONTRACT.md §3` (line 259: "the agent's reported `pcName` is logged, not stored — admins rename from the dashboard") vs `§3` (line 258: "Server applies mutable `attributes` to the device doc")
**Problem:** Line 259 says the server ignores the agent's `pcName` (admin-owned). Line 258 says the server applies `attributes` from the agent to the device doc. The `attributes` block includes `pcName`. These directly contradict: either the server applies `attributes.pcName` (the agent can overwrite admin renames) or it doesn't (the agent's report is ignored). The architecture doc says "pcName is admin-owned" and the agent skill says "don't fight a dashboard rename" — but the contract's "applies attributes" rule would do exactly that.

**Fix:** Explicitly state in §3: "The server treats `attributes.pcName` from the agent as **informational** (logged for debugging, not stored). `hostname`, `osUsername`, `timezone`, and `machineGuid` are **agent-owned** and applied to the device doc." One sentence resolves the contradiction.

### M5 — Monotonic sanity force-close (#33) doesn't specify the state of the force-closed interval

**Files:** `docs/DECISIONS.md` #33; `docs/ARCHITECTURE.md §3` step 3
**Problem:** Decision #33's safety net force-closes an open interval when its monotonic elapsed time exceeds `idleThreshold × 3` (catching missed power events / Modern Standby). But it doesn't specify **what state** to assign the force-closed interval. If the user was `active` and the machine silently hung for 3 hours, the open interval was `active`; an implementer who keeps the last-known state records a 3-hour `active` interval for an empty desk.

**Fix:** Specify: the force-close splits the interval — the portion up to the **last real sample** keeps the last sampled state; the portion from the last sample to the force-close time is recorded as `idle` (no input/media observed for >idleThreshold). Log a `missed_power_event` warning. Add a Phase 1b unit test: 3-hour gap with no samples → asserted `active` up to the last sample, `idle` for the excess.

### M6 — `reauthorized` recovery depends on the agent having the org key, which may have been rotated/revoked since deployment

**Files:** `docs/API-CONTRACT.md §3`; `docs/DECISIONS.md` #45; `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §4
**Problem:** The agent skill §4 says on `401` → re-enroll with the org key. If `/enroll` returns `403` → revoked-terminal. After un-revoke, the contract says the agent re-enrolls. But if the org key embedded in the MSI has been rotated (old key revoked), the re-enroll gets `401` (bad org key) — not `403` (blocked device). The agent's retry loop may exhaust and the agent stays in limbo. The agent skill doesn't distinguish "org key rejected" from "device blocked."

**Fix:** Agent skill §4: "If re-enroll returns `401` (bad org key, distinct from `403` blocked), log a critical warning: 'org key rejected — MSI may be outdated.' Continue heartbeating with the existing token (which is valid again after un-revoke) and attempt `/ingest` directly. Do **not** enter the revoked-terminal state for a `401` on re-enroll — the device is not blocked, the key is." This pairs with H2's "try `/ingest` first" recommendation.

---

## Low

### L1 — `X-Atlas-Contract` header still says `1.1` in the v1.2 document

**Files:** `docs/API-CONTRACT.md` line 32
**Problem:** The document is labeled v1.2, but the wire header example still says `X-Atlas-Contract: 1.1`. An implementer building version-mismatch logic against this header will be wrong from day one.

**Fix:** Update all `X-Atlas-Contract` examples to `1.2`, or explicitly state in §5 that v1.2 is a doc-only revision with no wire change and the header stays `1.1`. Add a release checklist item: "When bumping `API-CONTRACT.md`, search the file for the old version string and update all examples."

### L2 — `batches` schema mixes `_id = batchId` with a redundant `{deviceId, batchId}` unique index

**Files:** `docs/ARCHITECTURE.md §7` (`batches` schema)
**Problem:** The schema says `_id: batchId` and also "unique on `{deviceId, batchId}`." Because `_id` is already globally unique, the compound unique index is redundant and suggests two competing identity concepts.

**Fix:** Either (a) keep `_id` as an auto-generated `ObjectId` and make `{deviceId, batchId}` the unique index, or (b) keep `_id = batchId` and replace the compound unique with a `{deviceId, receivedAt}` index for purge queries. Pick one and update the architecture and contract to match.

### L3 — §5 "Auth failures" wording conflates two 401 recovery paths

**Files:** `docs/API-CONTRACT.md §5` (line 342–345)
**Problem:** After Decision #45, `/heartbeat` accepts a revoked token (200), so the 401 that triggers the re-enroll path only comes from `/ingest`. But §5 says "on a revoked/invalid device token the agent attempts a single re-enroll" without naming the endpoint. An implementer could read this as applying to any 401, including the `/heartbeat` mismatch case (which has a different recovery path).

**Fix:** Reword §5: "On a `401` from **`/ingest`** (revoked/invalid token), the agent attempts a single re-enroll with the org key. On a `401` from `/heartbeat` (token↔deviceId mismatch) or `404` (device doc deleted), the agent re-enrolls as a **new** device."

### L4 — `possible_clone` flag false-positives on `null` vs real `machineGuid` (intermittent read failures)

**Files:** `docs/DECISIONS.md` #42; `docs/ARCHITECTURE.md §7`
**Problem:** The clone-detection rule is "multiple distinct `machineGuid`s on one `deviceId`." A device that intermittently fails the registry read sends `null` on one heartbeat and a real value on the next → two distinct values → false `possible_clone`.

**Fix:** State in #42/`ARCHITECTURE.md §7`: the `possible_clone` comparison **ignores `null`** values; only two or more distinct **non-null** `machineGuid`s trigger the flag.

### L5 — "Forget device" naming implies full erasure but the `devices` doc survives (blocked)

**Files:** `docs/DECISIONS.md` #48; `docs/ARCHITECTURE.md §7`
**Problem:** Decision #48 "forget device" revokes the token, sets `blocked`, deletes intervals + receipts — but the `devices` doc survives (so `/enroll` returns 403). The employee's `pcName`/`hostname`/`osUsername`/`enrolledAt` remain on the blocked doc. The name "forget" implies the record is gone, which will mislead admins and any future privacy review.

**Fix:** Rename to **"block and wipe"** (or "revoke and wipe") in the dashboard + docs, and document that the device doc is intentionally retained to prevent re-enrollment.

---

## Nice-to-have

### N1 — `liveness` in the `/heartbeat` response is redundant for the agent
The agent already knows it's online (it just sent a heartbeat). The agent only needs `flags` (for `revoked`/`reauthorized` behavior). Sending `liveness` to the agent is harmless but invites an implementer to make the agent branch on it. Document that `liveness` is informational-only and the agent must not branch on it.

### N2 — Add a contract test fixture in the mother repo
A small shared test (e.g., `tests/contract/`) that exercises `/enroll`, `/ingest`, and `/heartbeat` against a running console would catch agent↔console drift before release.

---

## Phase 0 Readiness

Phase 0 is **ready to start**. The only Phase-0-affecting gap is the `/enroll` token-present mechanism (no `Authorization` header is documented for `/enroll`, so the Phase 0 implementer must guess whether rotation or no-op applies). A pragmatic Phase 0 can work around this by always rotating (safe at this scale), but the spec should be resolved before Phase 2's real token-loss recovery flow is built.

All other Phase 0 items are unambiguous: dev auth bypass, dev org key seed, fixed-seed dev script, one-real-smoke-test, ACL-locked prod logs, `auditLog` infra, and single-node-RS `127.0.0.1` bootstrap are all in the checklist with clear exit criteria.

---

## Where v1.2 is already correct and needs nothing

- **Token-loss rotation (Decision #43).** Fixes the hashed-token impossibility. Buffered rows sync under the new token. Correct.
- **Dedup uniqueness lives only on `batches` (Decision #28).** Fixes the multi-interval-batch rejection. Correct.
- **Enroll rate limit = 10/min/IP + corp-CIDR allowlist (Decision #44).** Prevents the 100-hour rollout problem. Correct.
- **Idle entry debounced, exit on any input (Decision #34).** Correctly reconciles idle hysteresis with call participation (the tradeoff on keep-awake is acknowledged but the design is right).
- **Active comms call overrides concurrent non-comms media (Decision #7).** Prevents Teams+Spotify from being labeled `passive_media`. Correct.
- **Device state = liveness + flags (Decision #39).** The split is correct; the gaps are in transition rules and boolean coexistence, not the model.
- **Heartbeat never returns 426; fallback manifest URL (Decision #47).** Preserves the auto-update recovery path. Correct.
- **`SERVICE_CONTROL_POWEREVENT` + monotonic sanity + sub-30s debounce (Decision #33).** Correctly handles Session-0 services and BD power blips. Only the force-close state-assignment is unspecified (M5).
- **Post-resume state re-derived from next sample (Decision #33).** Fixes the hardcoded-idle-on-resume bug. Correct.
- **Device timezone capture + `employees.tzOverride` + clock offset before bucketing (Decision #40).** Fixes non-Dhaka hire bucketing and skewed-device day boundaries. Correct.
- **Pipe anti-spoofing + `(instanceId, seq)` dedup (Decision #46).** Correctly closes the standard-user injection path and the restart seq-reset bug. Only issue is Phase-1a-vs-cert timeline (H4).
- **Forget device atomicity (Decision #48).** Correctly closes the partial-erasure hole. Only issue is the naming (L5) and the agent-buffer gap (covered by kimi-r3 C1).
- **Phase 1a/1b split.** Capture-health correctly in 1a, state machine in 1b. Phase 2 can scaffold after 1a (console side is fully buildable). The only 1a blocker is the pipe-signature dev-mode bypass (H4).

---

*End of review. Only this file was created; no other documents were modified.*
