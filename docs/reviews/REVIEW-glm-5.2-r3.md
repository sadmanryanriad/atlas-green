# Atlas Green — Pre-Implementation Design Review (Round 3)

- **Model:** glm-5.2 (opencode-go/glm-5.2)
- **Date:** 2026-06-23
- **Scope:** Stress-test the v1.2 revisions (DECISIONS #43–#48, amended #7/#18/#28/#29/#33/#34/#37/#38/#39/#40/#42, API-CONTRACT v1.2, Phase 1a/1b split) for new problems introduced by the round-2 fixes. Rounds 1–2 findings are not repeated.

## Executive summary

1. v1.2 absorbed round 2 thoroughly — token-loss rotation, revoked-device visibility, pipe anti-spoofing, heartbeat-never-426, forget-device atomicity, liveness+flags, the idle-entry-debounce, and the batches index fix are all real improvements. No architectural error was introduced.
2. Three High issues are **new regressions/contradictions from the v1.2 fixes**: the `batches` TTL (~180 d) is shorter than `bufferCapDays`'s max (365 d), so an agent offline >180 d with an un-ACKed batch re-inserts committed data as "new" (duplicate); `/enroll`'s "presents a valid existing token → no-op" has **no wire mechanism** (§1 headers carry no `Authorization`), so rotation-vs-no-op can't be distinguished; and "any input exits idle / resets the 300 s timer" (the r2 fix for bursty-input calls) is defeated by keep-awake/mouse-jiggler tools — a single phantom jiggle every <300 s now keeps an empty desk perpetually `active`, the opposite failure of the old 30 s-sustained rule.
3. Mediums cluster around the liveness+flags model: the `devices` doc still carries `blocked: bool` + `archived: bool` **alongside** the new `flags` (dual source of truth, no sync rule); flag transitions/co-existence rules are still absent; `archived` ingest behavior is still undefined; `reauthorized` is "true once" but a missed heartbeat strands the agent revoked, and the re-enroll-after-reauthorize path depends on a possibly-rotated org key.
4. The Phase 1a/1b split is sound (capture-health correctly sits in 1a; 1b doesn't leak back), **except** the pipe peer code-signature verification (#46, placed in 1a) requires the signing cert that isn't created until Phase 5/6 — 1a blocks in dev unless a dev-mode bypass is specified.
5. Phase 0 is unambiguous to start. The only Phase-0-affecting gap is the `/enroll` token-present mechanism (H2), which the Phase 0 `/enroll` implementer will need to guess at.

---

## High

### H1 — `batches` TTL (~180 d) is shorter than `bufferCapDays` max (365 d) → duplicate data when a long-offline agent replays an un-ACKed batch
**Where:** `docs/DECISIONS.md #28` ("TTL ~180 d, past the buffer cap"), `docs/API-CONTRACT.md §2` (line 160), `docs/ARCHITECTURE.md §7` (batches TTL), vs `docs/API-CONTRACT.md §1` (`bufferCapDays` valid `[7,365]`), `docs/ARCHITECTURE.md §7` (`retention.batchReceiptDays: 180`).

**Problem:** #28's stated rationale is "a receipt can't outlive the agent's max buffer hold anyway," so the TTL was set to ~180 d. But `bufferCapDays` is configurable up to **365** — exactly the "long offline machine" case the buffer cap exists for. The duplicate path: agent syncs batch X → server commits the transaction (receipt + intervals written) → the network drops before the agent receives the 200 → the agent keeps batch X un-ACKed → the agent goes offline for 200 d (within a 365-d cap) → the server's receipt for batch X TTL-expires at 180 d → the agent returns at 200 d, retries batch X → server finds no receipt → treats it as new → **re-inserts intervals that were already stored 200 d ago → duplicate data**. The `intervals` collection has no dedup (non-unique index, by design per the r2 fix), so nothing stops the double-insert. At defaults (cap=75, TTL=180) this is latent (75 < 180), but any admin who raises `bufferCapDays` above 180 — the explicit purpose of the 365 max — silently turns the dedup collection into a duplicate source. The same TTL window also re-opens the stale-replay-resurrection hole #28 was meant to close: after 180 d, an ordinary (non-"forget") interval delete leaves no receipt, so a stale replay resurrects the deleted data — contradicting #28's "rejected as a duplicate, not resurrected" guarantee, which now only holds for 180 d.

**Fix:** Tie the TTL to the configured cap, not a hardcoded 180. Concretely: `batches` TTL = `max(bufferCapDays + 45, 180)` days, recomputed when `bufferCapDays` changes (or simply set `expireAfterSeconds` from `settings.retention.batchReceiptDays` and document that it must exceed the fleet's max `bufferCapDays`). Cheapst path: cap `bufferCapDays` at `batchReceiptDays − 30` in the config-validation ranges. Either way, state in #28 that the anti-resurrection guarantee holds **within the TTL window** and that "forget device" (#48) is the only path that deletes receipts explicitly for full erasure.

### H2 — `/enroll` "presents a valid existing token → no-op" has no wire mechanism; rotation-vs-no-op is indistinguishable
**Where:** `docs/API-CONTRACT.md §1` (Behavior, lines 81–88: "a re-enroll that presents a valid existing token is a no-op… a re-enroll without a valid token rotates") vs `§1` Request headers (lines 42–45: only `X-Atlas-Org-Key` + `Content-Type` — **no `Authorization`**), `docs/DECISIONS.md #43`.

**Problem:** §1 Behavior distinguishes two re-enroll cases by whether the agent "presents a valid existing token," but the documented `/enroll` request headers carry **no `Authorization`** — only the org key. So the server cannot tell a re-enroll-from-a-device-that-still-has-its-token from a token-loss re-enroll; both arrive with just `X-Atlas-Org-Key` + a `deviceId`. An implementer must guess: (a) always rotate → every re-enroll churns the token (the "no-op" case never happens, and a spurious re-enroll triggered by a transient `/ingest` 401 rotates a perfectly good token, superseding the row the agent is still using); or (b) never rotate → token-loss recovery is broken (the agent can't get a new token because the server returns "existing" — which it can't, tokens are hashed, #43). Either guess breaks one of the two cases #43 was meant to support. This is a Phase 0 blocker for the `/enroll` implementer.

**Fix:** Add an **optional `Authorization: Bearer <existing token>` header to `/enroll`** (alongside `X-Atlas-Org-Key`). Server logic: org-key validated first (401 if bad) → device-block check (403) → **if `Authorization` is present and the token hash matches a non-revoked row bound to this `deviceId` → no-op (return the device record; the agent keeps its token)** → **if `Authorization` is absent or the token is invalid/unknown → rotate** (supersede old row, mint new token). State this explicitly in §1 Request headers and Behavior, and in the agent skill §4 (the agent sends `Authorization` on `/enroll` iff it still has a stored token).

### H3 — "Any input exits idle / resets the 300 s timer" is defeated by keep-awake / mouse-jiggler tools → perpetual false `active` (regression vs the old 30 s-sustained rule)
**Where:** `docs/DECISIONS.md #34` ("enter idle after 300 s continuous no-input — any input resets the 300 s timer… leave idle on any input → active"), `docs/ARCHITECTURE.md §4`, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md §3`.

**Problem:** #34's v1.2 form (adopting glm-r2 H4's recommendation) fixed the bursty-input-call problem by making **any** input reset the 300 s entry timer and exit idle. But this is the opposite failure on the other axis: a single input event every <300 s now prevents idle entry forever. The most common source is **keep-awake / mouse-jiggler software** (Caffeine, PowerToys Awake, a $2 USB jiggle stick, or a PowerShell `SendKeys` loop) — which exists *precisely* to defeat idle/screen-lock detection and is a known weapon in any monitored office. One jiggle every 60 s → the 300 s timer never completes → the device records **24×7 `active`** while nobody is there. The old "30 s sustained input to leave idle" rule (which v1.2 replaced) actually *resisted* this (a single 1-s jiggle couldn't exit idle); the v1.2 fix for call participants reintroduced the keep-awake vulnerability. For a system whose pitch is "an honest picture of how work time is spent," a trivially-installable $0 tool turning the whole fleet into "100% active" is a core-integrity defeat, and the spec doesn't acknowledge it. The sub-5 s merge and the 300 s entry-debounce don't help (jiggles are 60 s apart, not sub-5 s; they reset the timer before it completes).

**Fix:** Require a **small but non-trivial input quantum** to reset the idle timer / exit idle — enough to filter a single phantom jiggle but not so much that a call participant typing 5–10 s bursts stays idle. Concretely: "reset the 300 s idle timer only after ≥ ~2 s of cumulative input within a 5 s window" (or: ignore input events spaced > N s apart from prior activity when the prior activity was itself a single event). Pair with a known-keep-awoke-process check (Caffeine/Awake/`SendKeys` loops) flagged as a `health_warning`/`tamper` indicator. Add a Phase 1b unit test: one synthetic input event every 60 s for 30 min → asserted `idle` for the bulk (not `active`). Reference both #7 and #34 in the test. This is the call-vs-idle tradeoff reconciled at a third point rather than at either extreme.

---

## Medium

### M1 — `devices` doc still carries `blocked: bool` + `archived: bool` alongside `liveness` + `flags` → dual source of truth, no sync rule
**Where:** `docs/ARCHITECTURE.md §7` (devices shape, line 238–241: `… liveness, flags, … blocked: bool, archived: bool …`), `docs/DECISIONS.md #39`, `atlas-console/.agents/skills/atlas-api-contract/SKILL.md §5`/`§7`.

**Problem:** #39 moved device state to `liveness` + `flags` (`revoked`, `archived`, …) and says the dashboard reads those, never re-derives from `blocked`/`revokedAt`. But the `devices` doc still lists `blocked: bool` and `archived: bool` as separate fields. So there are now **two** representations of "revoked" (`devices.blocked` + `devices.flags.revoked` + `deviceTokens.revokedAt`) and "archived" (`devices.archived` + `devices.flags.archived`). The spec never says which is authoritative, which drives which, or that they're set atomically. An implementer will either set `blocked=true` and forget the flag (dashboard shows no revoked chip) or set the flag and forget `blocked` (the `/enroll` block check, which most implementers will key off `blocked`, doesn't fire). This is exactly the multi-source-of-truth problem #39 was meant to kill, just relocated.

**Fix:** Pick one. Recommended: keep `blocked` and `archived` booleans as the **stored inputs** and document that `flags.revoked`/`flags.archived` are **derived from them** at write time and stored denormalized for dashboard reads; a revoke action atomically sets `blocked=true` + pushes `revoked` into `flags` + sets `deviceTokens.revokedAt` in one update. Or remove the booleans and make `flags` the sole store (the `/enroll` block check then tests `"revoked" in flags`). State the relationship and the atomic-set requirement explicitly in `ARCHITECTURE.md §7` and the console skill §5.

### M2 — Flag transitions and co-existence rules are still absent (liveness+flags model has no state machine)
**Where:** `docs/DECISIONS.md #39`, `docs/ARCHITECTURE.md §7`, `atlas-console/.agents/skills/atlas-api-contract/SKILL.md §5`.

**Problem:** #39 split state into `liveness` + orthogonal `flags`, but **no transition or co-existence rules** are specified. Open questions an implementer will guess at: can `revoked` and `archived` co-exist (admin revokes then archives — both flags, or one supersedes)? Is `revoked` terminal (clearable only by admin un-revoke → `reauthorized`)? Is `archived` terminal? Does `outdated` clear when the agent updates? Does `clock_skewed` clear when the clock corrects? Does `health_warning` clear when cpu/mem normalize, or stick? Does `possible_clone` ever clear? Can `archived` be set on a `pending` device? minimax-r2 H4 flagged the missing transition table for the old enum; the enum became liveness+flags but the transition-table gap moved with it.

**Fix:** Add a short transition/co-existence table to `ARCHITECTURE.md §7`: which flags are terminal vs clearable and by what event; which may co-exist (`outdated`+`clock_skewed`+`health_warning`+`possible_clone` are all non-terminal badges and may stack; `revoked`+`archived` may co-exist, with `revoked` taking dashboard precedence); liveness is independent of all flags. ~10 lines of doc; prevents Phase 4 rework.

### M3 — `archived` flag's ingest/agent behavior is still undefined
**Where:** `docs/ARCHITECTURE.md §7` (`archived` flag), `docs/API-CONTRACT.md §2`/`§5` (errors cover `revoked` but not `archived`), `docs/DECISIONS.md #21`/`#48`.

**Problem:** `revoked` is fully specified (r3 #45: `/ingest` 401, `/enroll` 403, `/heartbeat` 200 + `flags:[revoked]`, agent stops capturing). `archived` is not. Does `/ingest` accept an archived device (it keeps appending data, defeating "archive") or reject it (401 → agent enters revoked-terminal state, but §5 only documents that flow for `revoked`)? Does `/heartbeat` return `flags:[archived]`? Does the agent stop capturing on `archived`? `#48 "forget device"` sets blocked + deletes data — is that the same as `archived`, or different? An implementer can't tell. (glm-r2 M7 raised this; it was not absorbed into v1.2 and now interacts with the new flags model.)

**Fix:** Define `archived` as: ingest-**blocking** (401, same agent flow as revoked — stop capturing, keep buffer, heartbeat-only); `/heartbeat` returns `flags:[archived]` (and `[revoked]` if both); dashboard shows `archived` as "hidden but recoverable" vs `revoked` as "blocked." A plain archive (no token revoke) keeps the token valid but blocked at `/ingest`. Un-archive clears the flag and the agent resumes on the next `reauthorized:true`. Add to `API-CONTRACT.md §5` and `ARCHITECTURE.md §7`.

### M4 — `reauthorized: true` "once" strands the agent if that heartbeat is missed; and re-enroll-after-reauthorize depends on a possibly-rotated org key
**Where:** `docs/API-CONTRACT.md §3` (line 228: "`reauthorized: false` // true once after an admin un-revoke → agent resumes capture"; line 263: "On `reauthorized:true` the agent exits its revoked state and resumes (re-enrolls for a fresh token/config)"), `docs/DECISIONS.md #45`.

**Problem:** Two issues. (a) "True once" — if the agent misses that single heartbeat (network blip, service restart), the flag is gone on the next heartbeat, and the agent stays in its self-imposed revoked state forever (until a manual service restart). A one-shot signal over a lossy channel is fragile. (b) "Re-enrolls for a fresh token/config" — re-enroll needs the org key. If the org key has been **rotated** since the agent was deployed (the transition window closed and the agent's baked-in key was revoked), the agent's `/enroll` gets 401 → it can't re-enroll → but its **existing token is now valid again** (admin un-revoked it) → it could just resume `/ingest`. Requiring re-enroll introduces a stale-org-key dependency that un-revoke doesn't actually need.

**Fix:** (a) `reauthorized: true` is returned on **every** heartbeat while the device is un-revoked **until the server observes the agent acting on it** (a successful `/ingest` or a `/enroll` with the new token), not "once." (b) On `reauthorized:true`, the agent should first **resume `/ingest` directly** (its existing token is valid again); only if `/ingest` still 401s should it fall back to re-enroll. Drop "re-enrolls for a fresh token" as the primary path — it's the recovery-for-token-loss path, not the un-revoke path. State both in §3 Behavior and the agent skill §4.

### M5 — Double-rotation race on concurrent `/enroll` retries (no `Authorization`) → agent may store a superseded token
**Where:** `docs/API-CONTRACT.md §1` (rotation on token loss), `§5` (Auth failures: "a single re-enroll"), `docs/DECISIONS.md #43`.

**Problem:** Agent loses its token, calls `/enroll` (no `Authorization`) → server rotates, mints tokenB. The response is delayed (slow network). The agent's retry timer fires (or the HTTP client times out and retries) **before** the first response arrives → a second `/enroll` (still no `Authorization`) → server rotates again, supersedes tokenB, mints tokenC. If the responses arrive out of order, the agent stores tokenB (superseded) after tokenC (current) → subsequent `/ingest` with tokenB → 401 → the agent thinks it's still revoked, loops. Even in-order, the agent burns two rotation cycles and the server accumulates a superseded-then-superseding chain. §5 says "a single re-enroll" but doesn't forbid concurrent in-flight `/enroll` calls.

**Fix:** State in §1/§5: `/enroll` retries must be **strictly sequential** — the agent must not have two `/enroll` requests in flight; wait for the response or HTTP-client timeout before retrying. Server-side defense: if a no-`Authorization` `/enroll` for a `deviceId` arrives within N seconds of a prior rotation whose new token has **not yet been used** (no successful `/ingest`/`/heartbeat` with it), return the same new token instead of rotating again. Add a Phase 2 integration test: two rapid token-loss re-enrolls → same token returned, one `deviceTokens` row superseded.

### M6 — Monotonic sanity force-close (#33) doesn't specify the state of the force-closed interval → could record phantom `active`
**Where:** `docs/DECISIONS.md #33` ("force-close an open interval whose elapsed time > idle-threshold ×3"), `docs/ARCHITECTURE.md §3` step 3, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md §3`.

**Problem:** #33's safety net force-closes an open interval when its monotonic elapsed time exceeds `idleThreshold × 3` (catching missed power events / Modern Standby). But it doesn't say **what state** to assign the force-closed interval. If the user was `active` and the machine silently hung for 3 h (missed suspend), the open interval was `active`; an implementer who keeps the last-known state records a **3 h `active` interval for an empty desk**. The spec gives no guidance, so the worst natural implementation is likely. (The r2 reviews that motivated #33 — deepseek M5, mimo M4 — flagged the missed-event case but not the state-assignment.)

**Fix:** Specify: the force-close splits the interval — the portion up to the **last real sample** keeps the last sampled state; the portion from the last sample to the force-close time is recorded as `idle` (no input/media observed for >idleThreshold). Log a `missed_power_event` warning. Add a Phase 1b unit test: 3 h gap with no samples → asserted `active` up to the last sample, `idle` for the excess, no 3 h `active` blob.

### M7 — `/heartbeat` accepts a revoked token → a leaked revoked token can spoof the heartbeat panel (liveness + capture health) for a revoked device
**Where:** `docs/DECISIONS.md #45`, `docs/API-CONTRACT.md §3` (heartbeat accepts revoked → 200), `docs/ARCHITECTURE.md §6`.

**Problem:** #45's "revoked stays visible" is a good UX goal, but accepting a revoked token on `/heartbeat` means anyone who **extracted the token before revocation** (a standard user who grabbed the DPAPI blob, or a leaked token) can keep sending heartbeats with `helperRunning: true, lastSampleUtc: now, queuedRows: 0` → the device shows `liveness: online, flags:[revoked]` indefinitely, and the capture-health panel looks healthy. `/ingest` is blocked (no fake activity data lands), so the integrity of the activity record is safe — but the **liveness/tamper backstop** (#17: "an agent that goes silent is itself a flag") is defeated for revoked devices: a revoked machine that's actually gone dark can be masked as "online + revoked" by the attacker who caused the revocation. The spec doesn't restrict what a revoked-token heartbeat may update.

**Fix:** A `/heartbeat` from a revoked token should: update `lastSeenAt` (so the device shows online+revoked, which is the intent) **but** should **not** update `capture` health fields, `helperRunning`, or clear `health_warning`/`possible_clone` flags — those are frozen at the revoke timestamp. Better: the dashboard should render `revoked` as the **primary badge** (overriding `liveness`) so "online + revoked" displays as "revoked," not "online." State both in `ARCHITECTURE.md §6`/`§7` and the dashboard rule.

---

## Low

### L1 — `X-Atlas-Contract` header says `1.1` but the contract is v1.2
**Where:** `docs/API-CONTRACT.md` line 32 ("echoed by the server … `X-Atlas-Contract: 1.1`").

**Problem:** The doc is v1.2 (header banner + §1–§5 revisions), but the wire header value is still `1.1`. An implementer will either bump it to `1.2` (a "major mismatch" → 426 logic may fire against v1.1 agents, which is wrong) or leave `1.1` and wonder if the doc version means anything. The `426` rule keys off "major mismatch," so the header is presumably the wire major.minor, but the doc never says that — and `1.1` ≠ `1.2` under any reading.

**Fix:** Clarify in §5 that the `X-Atlas-Contract` value is the **wire protocol version** (major.minor, independent of the doc revision number) and bump it to `1.2` to match the doc, OR state that v1.2 is a doc-only revision with no wire change and the header stays `1.1`. Pick one and make the doc internally consistent.

### L2 — `liveness` in the `/heartbeat` response is redundant for the agent
**Where:** `docs/API-CONTRACT.md §3` (response includes `liveness: "online"`).

**Problem:** `liveness` is the server's view of the agent, which the server owns and the dashboard reads. The agent already knows it's online (it just sent a heartbeat). The agent only needs `flags` (for `revoked`/`reauthorized` behavior) and `update`/`config`. Sending `liveness` to the agent is harmless but invites an implementer to make the agent act on its own liveness, which it shouldn't.

**Fix:** Either drop `liveness` from the response (the agent doesn't need it), or document that it's informational-only and the agent must not branch on it. Minor.

### L3 — `possible_clone` flag fires on `null` vs a real `machineGuid` (intermittent read failures)
**Where:** `docs/DECISIONS.md #42` ("multiple distinct `machineGuid`s → `possible_clone`"), `docs/API-CONTRACT.md §3` (attributes), `docs/ARCHITECTURE.md §7`.

**Problem:** #42 reads `machineGuid` from `HKLM\…\Cryptography`; if unavailable, sends `null`. The clone-detection rule is "multiple distinct `machineGuid`s on one `deviceId`." A device that intermittently fails the registry read sends `null` on one heartbeat and a real value on the next → two distinct values → false `possible_clone`. This is realistic (the LocalSystem service read can fail transiently under registry pressure).

**Fix:** State in #42/`ARCHITECTURE.md §7`: the `possible_clone` comparison **ignores `null`** values; only two or more distinct **non-null** `machineGuid`s trigger the flag. One line.

### L4 — `§5` "Auth failures (401): on a revoked/invalid token the agent attempts a single re-enroll" should now be scoped to `/ingest` 401
**Where:** `docs/API-CONTRACT.md §5` (Auth failures, line 342).

**Problem:** After #45, `/heartbeat` accepts a revoked token (200), so the 401 that triggers the re-enroll path only comes from `/ingest`. §5 still says "on a revoked/invalid device token" without naming the endpoint, which an implementer could read as "if `/heartbeat` ever returns 401…" — but `/heartbeat` 401 now means token↔deviceId mismatch or a deleted device doc (§3), a different recovery path (re-enroll as new device, not the revoked flow). The two 401 cases need different agent responses.

**Fix:** Reword §5: "On a `401` from **`/ingest`** (revoked/invalid token), the agent attempts a single re-enroll… On a `401` from `/heartbeat` (token↔deviceId mismatch) or `404` (device doc deleted), the agent re-enrolls as a new device." Minor but prevents the two flows being conflated.

### L5 — "Forget device" naming implies full erasure but the device doc survives (blocked)
**Where:** `docs/DECISIONS.md #48`, `docs/ARCHITECTURE.md §7`.

**Problem:** #48 "forget device" revokes the token, sets `blocked`, deletes intervals + receipts — but the `devices` doc survives (it must, so `/enroll` returns 403 for the blocked `deviceId` and the agent doesn't re-create the device). So the employee's `pcName`/`hostname`/`osUsername`/`enrolledAt` remain on the blocked doc. That's correct mechanics (a fully-deleted doc would let the agent re-enroll and resurrect the device), but the name "forget" implies the record is gone. For a privacy/erasure action this naming will mislead admins and any future GDPR-adjacent review.

**Fix:** Rename to **"block and wipe"** (or "revoke and wipe") in the dashboard + docs, and document that the device doc is intentionally retained (minimal, `blocked`) to prevent re-enrollment, while all activity data (intervals + receipts) is deleted. Cosmetic but accuracy matters for a monitoring system's erasure semantics.

---

## Where v1.2 is already correct (needs nothing)

- **#43 rotation safety for buffered rows.** The claim "rotation is safe because batches are keyed by `deviceId`+`batchId`, not token era" is correct. Buffered rows sync under the new token; a superseded token doesn't orphan or duplicate batches. The only gap is the *mechanism* for detecting token-present (H2), not the rotation safety itself.

- **#45 revoked-device-visibility, conceptually.** A revoked device showing as `revoked` (not silently "offline") is the right UX, and `/heartbeat` returning 200 + `flags:[revoked]` is the right wire design. The gaps are the display precedence (M7) and the `reauthorized` durability (M4), not the core idea.

- **#46 pipe anti-spoofing design.** Per-instance UUID handshake + `(instanceId, seq)` dedup + peer verification (PID → image path → signature) is the correct fix for both the r2 seq-reset data-loss (deepseek H2) and the standard-user spoofing (kimi C3). The only issue is the Phase-1a-vs-cert-timeline collision (H4), not the design.

- **#47 `/heartbeat` never 426 + compiled-in fallback `manifestUrl`.** Correctly preserves the auto-update recovery path across a major contract bump. The `contractDeprecated` flag is a clean signal. No gap.

- **#48 "forget device" atomicity (delete intervals + receipts together).** Correctly closes the r2 "ordinary delete leaves receipts → partial erasure / replay path" hole. The naming is misleading (L5) but the mechanics are right.

- **#7 concurrent media+call override.** "An active comms call always overrides even when a non-comms source plays concurrently" plus the ≤60 s recency gate cleanly resolves the Spotify+Teams case (glm-r2 M5) and the stale-GSMTC-source case (kimi-r2 H4). The media-transitions-bypass-hysteresis scoping in #34 (the `(idle↔passive_media)` parenthetical) correctly leaves the `active↔passive_media` boundary on the 300 s idle timer. No contradiction.

- **#33 resume state re-derivation.** "On resume the new interval's state is re-derived from the next sample (not hardcoded `idle`)" + the `SERVICE_CONTROL_POWEREVENT` (not `SystemEvents.PowerModeChanged`) + the sub-30 s blip debounce are all correct fixes for the r2 power-handling findings. Only the force-close state-assignment is unspecified (M6).

- **#34 idle-entry debounce (reset-on-input).** "Any input resets the 300 s timer" is the correct fix for the boundary-oscillation problem (a jiggle at 299 s resets instead of toggling). The regression it introduces on the keep-awake axis (H3) is real but is a tradeoff to tune, not a reason to revert to the 30 s-sustained rule.

- **#39 liveness + orthogonal flags (the split itself).** Modeling co-occurring conditions (online + outdated + clock_skewed) as liveness + flags is correct — a flat enum couldn't represent that. The gaps are the missing transition/co-existence rules (M2) and the dual-source-of-truth with the booleans (M1), not the model.

- **#40 device-tz capture + `employees.tzOverride` + clock-skew-before-bucketing.** This is the right v1 posture: fixed org tz for the 99%, a per-employee override for the rare non-Dhaka hire, device tz captured now so history can be re-bucketed later, and the skew offset applied before the day boundary. The `bucketDate`+`bucketTimeZone` (not client-converted) rule correctly prevents the summary/timeline mismatch (kimi-r2 M2). No gap.

- **#42 `machineGuid` in heartbeat `attributes` for clone detection.** Sending it on the heartbeat (not just enroll) makes the clone signal realizable, and the `possible_clone` flag is the right severity (suspicious, not definitive). The `null`-comparison nit (L3) aside, the design is correct.

- **Phase 1a/1b split.** The split is well-chosen: capture-health (`helperRunning`, `lastSampleUtc`, `helperStartedAtUtc`, `helperRestartCount`) is correctly in 1a — none of it needs the state machine (it's pipe/process state, not activity state). `appName`/`sessionId` tagging at sample time (1a) carries cleanly into intervals at 1b. Phase 2 **can** start after 1a against synthetic/seed intervals (the console side `/ingest` + atomic transaction + `batches` is fully buildable without real intervals; the agent-side sync loop can be built and unit-tested with stub intervals). The only 1a blocker is the pipe-signature check (H4).

- **Phase 0 readiness.** Phase 0 is unambiguous to start: the dev auth bypass (kimi-r2 H5), dev org-key seed (kimi-r2 M5), fixed-seed dev script (mimo-r2 M6), one-real-smoke-test (deepseek-r2 M1), ACL-locked prod logs (glm-r2 L10), `auditLog` infra (minimax-r2 H3), and single-node-RS `127.0.0.1` bootstrap (kimi-r2 H6) are all now in the checklist. The only Phase-0-affecting gap is H2 (the `/enroll` token-present mechanism), which the Phase 0 `/enroll` implementer will need resolved to avoid guessing.

---

*End of round-3 review. No code or existing documents were modified; only this file was created.*
