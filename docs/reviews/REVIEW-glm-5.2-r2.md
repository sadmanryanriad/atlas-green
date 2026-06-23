# Atlas Green — Pre-Implementation Design Review (Round 2)

- **Model:** glm-5.2 (opencode-go/glm-5.2)
- **Date:** 2026-06-23
- **Scope:** Stress-test the v1.1 revisions (DECISIONS #28–#42, API-CONTRACT v1.1, redistributed PLAN) and surface what six iteration-1 reviews missed. Iteration-1 findings are not repeated.

## Executive summary

1. v1.1 absorbed iteration 1 well, but one Critical spec bug slipped through: `ARCHITECTURE.md §7` declares `{deviceId, batchId}` **unique** on the `intervals` collection — a batch's rows share that key, so a unique index makes every batch of >1 interval fail. The unique constraint belongs only on `batches`.
2. Two locked v1.1 decisions contradict each other and the rollout model: §1 "re-enroll returns the existing token" is impossible because §5 stores tokens **hashed** (unrecoverable); and `/enroll ≤ 1/hour/IP` (§5) against an office NAT makes the 100-PC manual rollout take ~100 hours.
3. Decision #40 (Dhaka day-bucketing) is wrong for the non-Dhaka "remote staff" the plan itself acknowledges, and the agent's local timezone is never captured — so the deferred per-employee fix can't be back-filled. Decision #34's "30 s sustained input to leave idle" silently overrides Decision #7's "input within threshold = active," re-classifying entire calls as idle.
4. New edge cases: clock-skew offset is applied at display but not before Dhaka bucketing; `batches` has no TTL (unbounded growth); the device `status` single-enum can't show online+outdated+clock_skewed; 426-on-heartbeat bricks the auto-update recovery path; concurrent media+call mis-gates WASAPI.
5. The redistributed Phase 7 is largely right (tar-pit resolved). Phase 0 is well-scoped. The helper-as-user / DPAPI-in-LocalSystem split is clean — the only caveat is an undocumented "service must stay LocalSystem" constraint.

---

## Critical

### C1 — `{deviceId, batchId}` declared **unique** on `intervals` makes any batch of >1 interval impossible to store
**Where:** `docs/ARCHITECTURE.md §7` (intervals index list, line 231: "`{ deviceId, batchId }` unique"), mirrored by `docs/DECISIONS.md #28`.

**Problem:** A batch contains N intervals that **all share the same `deviceId` and `batchId`**. A unique index on `{deviceId, batchId}` over the `intervals` collection allows exactly **one** interval per (deviceId, batchId) pair. So the first interval of a 37-row batch inserts; the second hits a duplicate-key error and the atomic transaction (Decision #29) aborts. `/ingest` can only ever accept batches of size 1. This is a leftover from the pre-`batches` dedup design (iteration 1 suggested a unique index on `batchId` for dedup *before* the receipt collection existed); v1.1 added the `batches` receipt collection with its own unique index **and** kept the unique index on `intervals`, doubling the constraint and breaking the multi-interval case. It will surface the moment a Phase 2 integration test sends a real batch.

**Recommendation:** On `intervals`, make `{deviceId, batchId}` a **non-unique** index (used only for "find intervals of this batch" / admin-delete-by-batch queries). Keep the **unique** constraint solely on `batches` (`{deviceId, batchId}`), which is the one-receipt-per-batch serialization point. Edit `docs/ARCHITECTURE.md §7` and re-state in `docs/DECISIONS.md #28` that dedup uniqueness lives on `batches`, not `intervals`.

---

## High

### H1 — "Re-enroll returns the existing token" contradicts "tokens stored hashed server-side" → DPAPI-loss recovery is impossible
**Where:** `docs/API-CONTRACT.md §1` (line 75: "returns the **existing** token (no rotation)") vs `§5` (line 294: "device tokens are stored hashed server-side"). Also `docs/PLAN.md` Phase 2 line 124.

**Problem:** A hashed token is one-way; the server cannot return it. So when an agent loses its DPAPI-encrypted token (corrupted blob, deleted ProgramData file, DPAPI master-key invalidation after a domain join / service-account change) and calls `/enroll` with its `deviceId`, §1 promises the existing token — which the server does not have. The agent is bricked with no documented recovery: §1 says "no rotation," §5 says "rotation only via explicit admin revocation," but the only way to issue a usable token is to rotate. The contradiction was introduced by combining mimo-v2-5-pro's "return existing token, no rotation" (C2) with the standard "store tokens hashed" rule without reconciling them. (Note: rotation itself is safe here — batches are keyed by `deviceId`+`batchId`, not by token era, so buffered rows sync fine with a new token. The mimo concern about a "buffered-data era" does not apply to this design.)

**Recommendation:** Rewrite §1 Behavior: "A re-enroll that **presents a valid existing token** is a no-op (already enrolled). A re-enroll **without** a valid token (token lost) **rotates**: the server marks the prior `deviceTokens` row revoked (`supersededAt`) and issues a new token. A blocked/revoked device still gets `403`." Keep tokens hashed; rotation just means "revoke old + mint new." Add a Phase 2 integration test: agent loses token → re-enroll → new token → buffered rows sync.

### H2 — Org-timezone (Asia/Dhaka) day-bucketing is wrong for the remote employees the plan itself acknowledges, and the device timezone is never captured
**Where:** `docs/DECISIONS.md #40` ("per-employee timezone bucketing is over-engineering for v1"), `docs/AGENTS.md §1` / `docs/ARCHITECTURE.md §8` ("the team spans multiple time zones"), `docs/DECISIONS.md #22` ("remote employees/admins span multiple time zones"), `docs/API-CONTRACT.md §1`/`§3` (no timezone field anywhere).

**Problem:** #40 fixes the UTC-bucketing bug iteration 1 flagged (glm-5.2 M4) by bucketing on Dhaka — correct for the 99%, but it **relocates** the same bug to the 1%: a remote employee in UTC−5 working 09:00–17:00 local maps to ~20:00–04:00 Dhaka, so their whole workday is split across two Dhaka days and the bulk lands on the *next* day's summary. Weekly/weekend detection by Dhaka-Friday/Saturday mislabels their weekend. The plan repeatedly says remote/multi-tz staff exist (R2 CDN "for remote staff" #20, #22, §8), so this isn't hypothetical. Worse: the agent **knows** its OS timezone at capture but the contract never sends it up (no `timezone` in enroll or heartbeat `attributes`). So when per-employee bucketing is later un-deferred, historical intervals can't be re-bucketed correctly — the information was never recorded. The deferral is much more expensive than #40 claims.

**Recommendation:** Add `timezone` (IANA name, e.g. `Asia/Dhaka`, `America/New_York`) as a mutable device attribute sent in `/enroll` `machineHints`-adjacent block and in heartbeat `attributes` (when changed); store on the `devices` doc. v1 dashboard **still** buckets by org tz per #40, but (a) shows the device's local tz as a badge next to the Dhaka-bucketed summary so the misalignment is visible, and (b) flags devices whose `timezone ≠ orgTimeZone` in the heartbeat panel. This is one field, free at capture, and it preserves the option to re-bucket historically without a data loss. Reconcile the "team spans multiple time zones" wording with #40's "99% in Dhaka" framing.

### H3 — `/enroll` rate limit `≤ 1/hour/IP` vs the office NAT breaks the stated 100-PC manual rollout
**Where:** `docs/API-CONTRACT.md §5` (rate limits: "`/enroll` ≤ 1 / hour / IP"), `docs/AGENTS.md §4` / `docs/DECISIONS.md #16` ("MSI installed manually PC-to-PC by IT").

**Problem:** All ~100 office desktops share one public IP (office NAT). At 1 enrollment per hour per IP, IT can enroll 1 machine per hour → **100 hours of walking around** to roll out. This directly defeats the locked deployment model. The limit is right for rogue-enrollment defense (glm-5.2 M15 / minimax M13), but the per-IP dimension is wrong for a single-corporate-LAN fleet.

**Recommendation:** Keep per-device enroll throttling, but drop the per-IP enroll limit for the corporate network. Concretely: (a) raise `/enroll` to `≤ 10/min/IP` (still caps a scripted rogue burst while letting IT roll out 100 PCs in ~10 min), **and/or** (b) allow an `enrollAllowlistCidrs` entry in `settings` (the office CIDR) exempt from the per-IP limit — this dovetails with the corp-IP enroll gate iteration 1 suggested. State in §5 that the per-IP limit is a rogue-defense backstop, not a rollout throttle, and that the corporate LAN is allowlisted.

### H4 — Decision #34 "leave idle only after ~30 s sustained input" contradicts Decision #7 "input within threshold = active" for call participants and periodic-input users
**Where:** `docs/DECISIONS.md #34` (hysteresis) vs `docs/DECISIONS.md #7` ("a user actively on a call (input within threshold) is `active`"), `docs/ARCHITECTURE.md §4`.

**Problem:** #7 says any input within the 5-min idle threshold = `active`. #34 says leave `idle` only after **~30 s of sustained input**. Take a user on a 1-hour Teams call (gated out of media → not `passive_media`), typing 5–10 s every 2–3 min to take notes / answer in chat. At 5 min no-input they enter `idle`. Each 5–10 s burst is well under 30 s *sustained*, so under #34 they **never leave idle** — the entire call is recorded as `idle`, directly contradicting #7's "actively on a call = active." The same swallow applies to a reader who scrolls/mouse-moves for 10 s every few minutes. #34 was meant to stop boundary oscillation (299 s vs 301 s), but "30 s contiguous" is the wrong exit condition — it gates on *continuity* of input rather than *presence* of input, and most real work is bursty. The two locked decisions are not simultaneously satisfiable.

**Recommendation:** Reconcile in #34: keep the 300 s **enter**-idle debounce, but redefine the **exit** condition as a sliding window, not contiguous input — e.g. "leave idle when ≥ X seconds of input occur within any Y-second window" (say ≥8 s of input within 30 s), or simply "any input event exits idle and restarts the 300 s active timer" with the oscillation fix applied on the **enter** side (require 300 s of *continuous* no-input-plus-no-media to enter idle, so a single jiggle at 299 s resets the 300 s clock instead of toggling). Drop "30 s sustained." Add a Phase 1 unit test: call participant with 5-s input bursts every 3 min over 60 min → asserted `active` for the input windows, not `idle` for the whole hour. Reference both #7 and #34 in the test so the reconciliation is enforced.

### H5 — Decision #37 "employeeId never back-filled" contradicts Phase 4 "device→employee remap re-resolves affected intervals"
**Where:** `docs/DECISIONS.md #37` ("employeeId … never back-filled") vs `docs/PLAN.md` Phase 4 ("device→employee remap writes an `auditLog` entry and **re-resolves affected intervals**").

**Problem:** #37 says the interval's `employeeId` is a snapshot, never rewritten. Phase 4 says a remap re-resolves affected intervals (implying a historical rewrite). These are incompatible. The likely intent: a remap changes `employeeId` for **future** intervals only; historical intervals keep their snapshot; a **separate, explicit** bulk-reassign admin action (audit-logged, as minimax H8/L10 suggested) handles the rare case where the past must be reattributed. But Phase 4's wording bundles them, so an implementer will either (a) back-fill on every remap (violating #37, rewriting history silently) or (b) never re-resolve (violating Phase 4, leaving the dashboard action inert). Category re-resolve (rules change) **is** meant to back-fill; `employeeId` re-resolve is not — and Phase 4 doesn't distinguish them.

**Recommendation:** Split Phase 4's wording: "device→employee remap updates the `devices.employeeId` mapping; **new** intervals pick up the new `employeeId` at ingest (per #37, historical intervals are snapshots and are not touched). A separate **`bulkReassignIntervals` admin action** (audit-logged, optional date range) reattributes historical intervals and is the only path that rewrites them." Keep category bulk-re-resolve as its own item (it does rewrite history, by design).

---

## Medium

### M1 — Clock-skew offset is applied at display (#33) but not before Dhaka day-bucketing (#40) → skewed devices' daily summaries land on the wrong day
**Where:** `docs/DECISIONS.md #33` + `docs/ARCHITECTURE.md §8` ("the dashboard can apply the stored offset at display time") vs `docs/DECISIONS.md #40` (day/week bucketing uses `Asia/Dhaka`).

**Problem:** Day-bucketing uses the (possibly wrong) raw `startUtc`. The stored `clockOffsetMs` is applied only at **timeline rendering**, not before the Dhaka-day boundary computation. A device 1 h fast: activity at 23:30 Dhaka (true 17:30 UTC, which is < 18:00 UTC → Dhaka day N−1) is stamped 18:30 UTC → bucketed into Dhaka day N. The timeline shows it at the right wall-clock position (offset applied), but the daily/weekly summary attributes it to the **next** day. The `clock_skewed` flag surfaces the device but doesn't fix the bucket. This is a new interaction between two v1.1 decisions that iteration 1 couldn't have flagged (#40 didn't exist).

**Recommendation:** When bucketing a `clock_skewed` device by Dhaka day, apply `startUtc + clockOffsetMs` before computing the day boundary (the offset is stable across two heartbeats per #33, so it's safe to use). State in `docs/ARCHITECTURE.md §8` that the offset correction applies to **both** rendering and bucketing, while raw `startUtc` in storage stays untouched (per #33's "never rewrite buffered timestamps"). Add a Phase 4 unit test: a skewed device's late-evening activity lands in the correct Dhaka day after offset correction.

### M2 — `batches` receipt collection has no retention/TTL → unbounded growth (~10M docs/year)
**Where:** `docs/ARCHITECTURE.md §7` ("retained independently of `intervals`"), `docs/DECISIONS.md #28`, `docs/DECISIONS.md #21` (server retention indefinite).

**Problem:** 100 devices × ~1 batch / 5 min × 288/day × 365 = ~10.5M receipt docs/year (~1 GB/year, plus the growing unique index). "Retained independently" was the right anti-resurrection property, but it was specified with **no retirement**. The protective value of a receipt is bounded: a receipt only blocks a replay that an agent could actually send, and an agent can only hold an un-ACKed batch for ~`bufferCapDays` (default 75) before the buffer cap drops it. After ~75–120 days a receipt can never be legitimately replayed; and a *rogue* replay (extracted token + old batchId) is harmless because (a) revoked/archived devices are rejected at auth before dedup, and (b) a rogue with a valid token can already submit fresh-UUID fabricated batches — dedup isn't the control there, revocation is. So retaining receipts beyond the buffer window is pure waste.

**Recommendation:** Add a **TTL index** on `batches.receivedAt`, `expireAfterSeconds = (bufferCapDays + 45) * 86400` (≈120 days default), so receipts self-expire just past the agent's max buffer hold. State in `docs/ARCHITECTURE.md §7` and add a `settings.retention.batchReceiptDays` entry. This preserves the anti-resurrection guarantee for the only window it matters while bounding the collection.

### M3 — Device `status` single-enum (#39) cannot represent online + outdated + clock_skewed simultaneously
**Where:** `docs/DECISIONS.md #39` (`status: pending → online/offline, plus outdated/revoked/archived`), `docs/ARCHITECTURE.md §7` ("the dashboard reads **only** `status`"), `docs/API-CONTRACT.md §4` (below `minSupportedVersion` → `outdated`).

**Problem:** `status` is one enum. A device that is alive (`online` by `lastSeenAt`) but also below `minSupportedVersion` (`outdated`) and clock-skewed (`clock_skewed`) can only show **one** of those. The dashboard "reads only status," so liveness is hidden behind `outdated`/`clock_skewed` — IT can't tell whether an outdated device is still phoning home or has gone dark, which is the exact question that matters for an outdated fleet. `revoked`/`archived` are terminal and reasonably mutually exclusive with online, but `outdated` and `clock_skewed` are **badges**, not liveness states.

**Recommendation:** Split into two fields: `liveness: online|offline` (driven by `lastSeenAt` vs `heartbeatIntervalSec×3`) and `flags: [outdated?, clock_skewed?, revoked?, archived?]` (admin/system-set badges). The dashboard reads `liveness` for the online/offline badge and renders `flags` as chips on top. Keep #39's intent (one authoritative place, not three ad-hoc derived fields) but model it as liveness+flags rather than a flat enum. Update `docs/ARCHITECTURE.md §7` and `docs/DECISIONS.md #39`.

### M4 — "Reopen idle on resume" is wrong when media auto-resumes on wake
**Where:** `docs/DECISIONS.md #33` ("a new `idle` interval starts on resume"), `docs/ARCHITECTURE.md §3` step 3, `docs/PLAN.md` Phase 1 ("reopen idle on resume").

**Problem:** The spec hardcodes the post-resume state as `idle`. But on wake, a browser tab that was playing YouTube often **auto-resumes**, GSMTC may immediately report `Playing`, and the user may already be typing. The correct post-resume state is whatever a fresh sample says (`active` / `passive_media` / `idle`), not a presumption. Hardcoding `idle` creates a spurious idle interval that the 3–5 s sample then flips — and with #34's sub-5s merge, the merge precedence between "idle (presumed)" and "passive_media (sampled 3 s later)" is undefined, so the merged interval could be recorded as idle when the user was actually watching media.

**Recommendation:** Change the prescription to: **close** the pre-sleep interval at suspend time (this part is correct and should stay), then **start a new open interval on resume whose state is determined by the next fresh sample** (input + GSMTC/WASAPI), exactly as on a normal sample. Do not presume `idle`. Update `docs/DECISIONS.md #33`, `docs/ARCHITECTURE.md §3`, and the Phase 1 checklist. Specify that the sub-5s merge (#34) resolves a "presumed-then-sampled" conflict in favor of the **sampled** state.

### M5 — Concurrent media + call: the WASAPI gate miscounts the call as `passive_media`
**Where:** `docs/DECISIONS.md #7` (WASAPI counts only when the active GSMTC source is **non-communications**), `docs/ARCHITECTURE.md §4`.

**Problem:** The gate checks the **active/most-recent GSMTC source's** app category. A user running Spotify (non-comm) **while on a Teams call** (communications) has Spotify as the active/most-recent GSMTC media source → the gate says "non-comm source → audio counts as `passive_media`" → the call's audio is folded into `passive_media`. The user is on a call, not "watching media." The single-active-source assumption breaks when media and a call run concurrently (common: music during a call, a YouTube tab open while on a meeting). The spec has no rule for the concurrent case.

**Recommendation:** Refine the gate (#7): WASAPI audio yields `passive_media` only when (a) the active GSMTC source is non-communications **and** (b) **no** communications app currently has an active call/session (detect via GSMTC `SourceAppUserModelId` for known communications apps, or via the Windows `PhoneCallTransport`/call-state APIs if available). If a call is active, audio alone does **not** yield `passive_media` — the user is `active` if inputting, else `idle` (per #7's "silent call = idle"). Document the known limit: background music during a call is then counted as `idle`/`active` (not `passive_media`), which is the lesser evil vs mislabeling a call as media. Add a Phase 1 manual test: Spotify + Teams call concurrent → asserted not `passive_media` for the call duration.

### M6 — `426` on a major contract mismatch applies to `/heartbeat` too → the agent can't reach the update block and is bricked
**Where:** `docs/API-CONTRACT.md §5` ("the server returns `426` on a major mismatch (the agent then surfaces it via heartbeat and relies on auto-update to recover)"), `§3` (heartbeat `update` block carries `manifestUrl`).

**Problem:** If the server is upgraded to a new major contract version and returns `426` on **every** agent request, the agent's `/heartbeat` also 426s — so it never receives the `update` block with `manifestUrl`, and "relies on auto-update to recover" has no source for the manifest URL. The agent is stuck with no recovery path. `/ingest` 426 is fine (keep buffering), but `/heartbeat` is the lifeline and must always return the update pointer.

**Recommendation:** Specify that `426 Upgrade Required` applies to `/enroll` and `/ingest` only; **`/heartbeat` always returns 200** with the `update` block (and a `contractDeprecated: true` flag) so any agent, however old, can fetch `manifestUrl` and self-update. Add a fallback: a compiled-in **default `manifestUrl`** (R2 endpoint) so the agent can still find the manifest even if the heartbeat response is malformed/unreachable. State both in `docs/API-CONTRACT.md §5` and the Phase 6 update-client spec.

### M7 — `archived` status vs ingest acceptance is unspecified; the agent has no terminal state for archive
**Where:** `docs/ARCHITECTURE.md §7` (`status: … archived`), `docs/API-CONTRACT.md §5` (the revoked-terminal-state flow covers **revoked** only), `docs/DECISIONS.md #21` (manual per-PC archive/delete).

**Problem:** If `archived` blocks ingest (the intuitive reading — archiving should freeze the device), then an archived device's agent gets 401, attempts re-enroll, and per §5 enters the revoked-terminal state — but §5 only specifies that flow for **revoked** devices, and `/enroll` returns 403 for **blocked/revoked**, not explicitly for `archived`. If `archived` does **not** block ingest, the archived device keeps appending data, defeating the archive. Either way it's undefined, and the agent's behavior on archive (stop generating? keep buffering?) is unspecified.

**Recommendation:** Define `archived` as ingest-blocking (401), and state that the §5 revoked-terminal-state flow (stop generating new intervals, keep buffer, heartbeat only) applies to **both** `revoked` and `archived`. Distinguish them in the dashboard (archived = hidden but recoverable; revoked = blocked). Add to `docs/API-CONTRACT.md §5` and `docs/ARCHITECTURE.md §7`.

### M8 — `machineGuid` clone-detection intent (#42) is unrealizable: it's sent at enroll only, and idempotent re-enroll doesn't update it
**Where:** `docs/DECISIONS.md #42` ("`machineGuid` … the one cheap hint that could later detect a disk-clone"), `docs/API-CONTRACT.md §1` (`machineHints` at enroll only), `§1` Behavior (idempotent re-enroll returns existing token, no hint update).

**Problem:** #42 keeps `machineGuid` specifically to detect disk-clones. The realistic clone path is sysprep-imaging: Windows regenerates `MachineGuid` per clone, but the agent's `DeviceId` file (cloned pre-sysprep) is **not** regenerated — so N clones share one `deviceId` but have N distinct `MachineGuid`s. The first enroll records `machineGuid=A`; the other N−1 re-enrolls are idempotent (no hint update per §1), so their `machineGuid`s (B, C, …) are never stored. The server sees one `deviceId` with one `machineGuid` and N heartbeats — no clone signal. The hint's stated value is unreachable with the current contract.

**Recommendation:** Send `machineGuid` in the **heartbeat** `attributes` block (when changed), not just at enroll. The server flags a `deviceId` that reports **multiple distinct `machineGuid`s** across heartbeats as `possible_clone` (a new flag/badge, see M3). This realizes #42's intent with one extra field on the heartbeat. Also add an IT-runbook rule (Phase 5): **never image a machine with the agent already enrolled** — install the MSI *after* imaging so each PC generates its own `DeviceId`.

### M9 — Multi-monitor media attribution: a `passive_media` interval attributes the media to the **foreground** app, not the media source
**Where:** `docs/ARCHITECTURE.md §3`/`§4`, `docs/PLAN.md` Phase 7 ("Multi-monitor" as residual).

**Problem:** `GetForegroundWindow` returns the focused window. A user watching YouTube on monitor 2 while VS Code is focused on monitor 1, not typing: state = `passive_media` (media playing, no input), `appName` = "Visual Studio Code", `domain` = null. The interval says "passive_media on VS Code" — the media source (YouTube) is invisible. A dashboard "time on YouTube" by `appName` undercounts this; only a domain query would catch it, and only if the YouTube tab was foreground at some point. The model has no way to represent "foreground=X, media=Y." Phase 7 lists "multi-monitor" as a polish item, but the **attribution gap** is a data-model issue, not a capture polish.

**Recommendation:** Acknowledge the limit explicitly in `docs/ARCHITECTURE.md §4`: `passive_media` intervals attribute the media to the foreground app; when media is playing from a **non-foreground** source, the interval's `appName`/`domain` reflect the foreground, not the media source. Consider (v1.1, cheap) adding an optional `mediaSource: { appName?, domain? }` field on intervals when `state == passive_media` and the GSMTC media source differs from the foreground app — populated from GSMTC's `SourceAppUserModelId` → app name. This lets the dashboard surface "media from YouTube while foreground was VS Code." If deferred, at least document the undercount in the Phase 3/4 README so reports don't over-claim.

---

## Low

### L1 — `dropped` block is transient; the server should persist a data-gap record so the gap is visible historically
**Where:** `docs/API-CONTRACT.md §3` ("present only when the buffer cap forced a drop"), `docs/PLAN.md` Phase 2.

**Problem:** The `dropped {count, oldestUtc, newestUtc}` block appears on the **first heartbeat after the drop** and is absent thereafter. If the dashboard only renders the current heartbeat's `dropped`, the signal vanishes after one cycle — a manager looking at the timeline a week later sees a gap with no explanation. Nothing says the server persists it.

**Recommendation:** On receiving a `dropped` block, the server writes a `dataGaps` entry on the device (or a `gaps` collection): `{deviceId, count, oldestUtc, newestUtc, detectedAt}`. The dashboard renders gap markers on the timeline from this persistent record, not from the ephemeral heartbeat field. Add to `docs/ARCHITECTURE.md §7` and Phase 2.

### L2 — `dropped` block has no state breakdown; dashboard can't distinguish "5 active lost" from "95 idle dropped"
**Where:** `docs/API-CONTRACT.md §3`, `docs/PLAN.md` Phase 2 ("prefer dropping `idle`").

**Problem:** Phase 2 says prefer dropping `idle` over active/passive. The `dropped` block reports a single `count` — the dashboard can't tell whether the drop was harmless (idle) or a real loss (active/passive_media). "100 intervals dropped" reads alarming even if all 100 were idle.

**Recommendation:** Extend `dropped` to `byState: {active, passive_media, idle}` counts. The dashboard can then show "5 active / 0 media / 95 idle dropped" and triage accordingly. Trivial agent-side change.

### L3 — Service account MUST stay LocalSystem; changing it makes DPAPI-encrypted secrets unreadable and loses the buffer
**Where:** `docs/DECISIONS.md #32` (service as LocalSystem), `#30` (buffer key via DPAPI machine scope), `#15` (token via DPAPI machine scope), `docs/ARCHITECTURE.md §6`.

**Problem:** DPAPI machine-scope blobs encrypted by LocalSystem are decryptable by LocalSystem (and admins). If IT later hardens the deployment by switching the service to a named least-privilege service account, that account **cannot** decrypt the LocalSystem-encrypted token or buffer key → the device token is lost (forces re-enroll) **and** the buffer encryption key is lost (all buffered unsynced intervals are unrecoverable). The plan locks LocalSystem for v1 (#32) so this isn't a bug today, but the constraint — and the fact that there's no DPAPI-migration/re-encrypt procedure — is undocumented. An IT admin "tightening security" would silently brick agents.

**Recommendation:** Document in `docs/ARCHITECTURE.md §6` and the Phase 5 runbook: **the service must remain LocalSystem**; changing the service account invalidates DPAPI and loses the buffer. If least-privilege is ever required, a re-encrypt-on-account-change procedure must be built first (decrypt under LocalSystem → re-encrypt under new account). Flag as a v1 constraint.

### L4 — MSI upgrade/repair must preserve ProgramData or the agent loses identity + buffer
**Where:** `docs/PLAN.md` Phase 5 (WiX MSI), `docs/ARCHITECTURE.md §5`/`§6` (DeviceId file + DPAPI blobs + SQLite buffer under `C:\ProgramData\…`).

**Problem:** An MSI **major upgrade** or "repair" that removes/uninstalls the previous version's files can, depending on WiX config, delete `ProgramData\AtlasGreen` — wiping the `DeviceId` (→ re-enrolls as a new device, splits history), the DPAPI-encrypted token (→ re-enroll), and the SQLite buffer (→ loses unsynced intervals). The Phase 5 MSI item doesn't specify data preservation across upgrade.

**Recommendation:** Phase 5 checklist: "MSI upgrade/repair **preserves** `C:\ProgramData\AtlasGreen` (DeviceId file, DPAPI blobs, buffer DB); only a full uninstall removes it, and even then prompts/flags." Use WiX's `RemoveFolder` carefully — never on the data folder during upgrade. Add to the Phase 5 exit gate.

### L5 — GSMTC may report stale `Playing` immediately after resume → false `passive_media`
**Where:** `docs/DECISIONS.md #33`, `#7`, `docs/ARCHITECTURE.md §4`.

**Problem:** On wake, a GSMTC session that was Playing before sleep may briefly read `Playing` before the OS/app updates it to `Paused` (the audio engine resumes, the app hasn't caught up). If the agent trusts GSMTC on the first post-resume sample, it opens a `passive_media` interval that may persist until the next real state change — inflating media time while the user is away.

**Recommendation:** On resume, treat GSMTC as **untrusted for ~10 s** (force a re-read / require two consecutive `Playing` samples before asserting `passive_media`), or clamp the first post-resume interval to a short duration that self-corrects on the next sample. Note in `docs/ARCHITECTURE.md §4` and the Phase 1 sleep/resume test.

### L6 — `passive_media`↔`active` boundary has no hysteresis spec (only `idle`↔`active` does)
**Where:** `docs/DECISIONS.md #34` (hysteresis specified only for the idle boundary), `docs/ARCHITECTURE.md §4`.

**Problem:** A user watching YouTube and scrolling every ~20 s: each scroll flips `passive_media`→`active` (input), then input ages out → `active`→`passive_media`. With no hysteresis on this boundary, the state oscillates every few samples, flushing frequently — the exact problem #34 solved for idle but left unaddressed for media. The sub-5s merge (#34) partially absorbs it, but 20-s bursts aren't sub-5s.

**Recommendation:** Extend #34's hysteresis to the `passive_media`↔`active` boundary: stay `active` for a short debounce (e.g., 10–15 s) after the last input before dropping back to `passive_media`, mirroring the enter-idle debounce. One rule, applied to both non-idle↔idle-adjacent boundaries.

### L7 — `title-fallback` URL capture can yield **wrong** domains from window-title parsing
**Where:** `docs/DECISIONS.md #35` (fallback chain UIA → title-parse → `domain: null`), `docs/API-CONTRACT.md §2` (`urlCaptureMethod: "title-fallback"` with a non-null `domain`).

**Problem:** Parsing a domain from a window title is heuristic ("Inbox (23) - user@gmail.com - Gmail - Mozilla Firefox"). A wrong domain (e.g., extracting `gmail.com` when the user is elsewhere, or a title containing an unrelated URL) is **worse** than `null` — it silently misattributes time in domain-based reports. The contract allows `title-fallback` to produce a non-null `domain`, so an implementer will naively regex-extract one.

**Recommendation:** Define a high-confidence bar for `title-fallback` domain extraction: only emit a non-null `domain` when the title matches a strict pattern (e.g., a clear `" - <eTLD+1> - <Browser>"` suffix where the eTLD+1 is on the public-suffix list); otherwise emit `domain: null` with `urlCaptureMethod: "title-fallback"` and keep the `pageTitle`. Document in `docs/DECISIONS.md #35` and add a Phase 3 unit test: ambiguous titles → `domain: null`, not a guessed domain.

### L8 — `mandatory` field in the update block has undefined semantics
**Where:** `docs/API-CONTRACT.md §4` (`update.mandatory: false`).

**Problem:** The field exists but the contract never says what `mandatory: true` makes the agent do. Does it update immediately? Block `/ingest` after a deadline? Ignore `Retry-After`? An implementer will guess.

**Recommendation:** Define: `mandatory: true` means the agent updates **as soon as possible** (next heartbeat cycle, not waiting for the regular poll) and, if the update fails N times, reports `updateStatus: "failed"` and continues running the current version (no ingest shutoff — never lose data). State in `docs/API-CONTRACT.md §4`.

### L9 — `sessionId` is reused across logoff/logon; document it as a scoping hint, not a stable user id
**Where:** `docs/API-CONTRACT.md §2` (`sessionId`), `docs/ARCHITECTURE.md §5` ("every interval carries the Windows `sessionId`").

**Problem:** Windows session IDs are recycled: user A logs in → session 1, logs out → session 1 freed, user B logs in → session 1 again. So `sessionId: 1` is ambiguous across a logoff boundary. The dashboard's multi-user/mis-map detection (§5) relies on `osUsername`+timestamp, which is correct, but `sessionId` alone is not a stable identifier and the spec doesn't say so.

**Recommendation:** Note in `docs/ARCHITECTURE.md §5`: `sessionId` is a **contiguous-session scoping hint** (valid only within one login window, useful to separate simultaneous sessions), not a stable user identifier; attribute by `osUsername`+timestamp across windows. Keep `sessionId` on intervals (it's free), just document its limits.

### L10 — Prod log file location is unspecified; it must be ACL-locked, not user-readable
**Where:** `docs/PLAN.md` Phase 0 ("Structured logging to a rotating local file"), `atlas-agent/AGENTS.md` §Conventions, `docs/ARCHITECTURE.md §6`.

**Problem:** Phase 0 says "rotating local file (dev: also console)" but never says **where** the prod log lives. If it lands beside the exe or in `%TEMP%` (both user-readable), a standard user can read window titles / process names from the log — defeating the privacy posture (#36 title caps, §6 ACL-locked data). The logging rules say "never log secrets" but window titles (sensitive, per minimax M2 / glm L5) aren't classified as secrets.

**Recommendation:** Specify: prod logs go to `C:\ProgramData\AtlasGreen\logs\` (ACL-locked, same folder as the buffer), not user-readable. Add a Phase 0 checklist item: "prod log path is ACL-locked; verify a standard user cannot read it." Cross-reference the title-redaction rule (window titles at `Debug` only in prod).

### L11 — Agent should self-split batches **before** send when over the 5000/5MB cap, not only react to `413`
**Where:** `docs/API-CONTRACT.md §2`/`§5` (413 → split; hard cap ≤5000 intervals / ≤5 MB decompressed).

**Problem:** After a long offline period an agent may have 8000 unsynced rows. The contract says "keep batches under the hard cap" (§5) but the only explicit mechanism is "on 413, split." An implementer could send 8000, get 413, then split — wasting a round-trip and a rejected transaction. The proactive self-split isn't stated.

**Recommendation:** State in `§2`/`§5`: when the unsynced row count or estimated decompressed size exceeds the cap, the agent **self-splits into fresh-UUID sub-batches before sending** (no 413 needed). 413 remains the fallback for server-side size limits the agent mis-estimated. Add to the Phase 2 sync-loop checklist.

### L12 — Mis-mapped-machine detection (§5) is "visible" but no dashboard view flags it
**Where:** `docs/ARCHITECTURE.md §5` ("a mis-mapped machine … is *visible* in the dashboard rather than silently merged").

**Problem:** §5 says a device showing two distinct `osUsername`s in a short window is "visible" — but no dashboard view, flag, or alert implements the detection. It's only visible if an admin manually scrolls the timeline. The data is there; the signal isn't built.

**Recommendation:** Phase 4 checklist: dashboard flags a device whose intervals show ≥2 distinct `osUsername`s within a 24-h window (a "possible multi-user / mis-map" badge). Cheap server-side query at render time. Pairs with M3's flags model.

---

## Where the v1.1 plan is already correct (needs nothing)

- **Helper-as-the-logged-on-user vs DPAPI/secrets in LocalSystem (the §32 split).** This is **clean** and the iteration-1 concern (would the helper need secrets it can't reach?) is resolved by design: the helper holds **no secrets** and does **no networking** — it only samples and streams over the named pipe. The device token, org key, and buffer-encryption key all live in the service (LocalSystem, DPAPI machine scope). The helper doesn't need the DeviceId either (the service tags samples). There is no DPAPI split-brain. The only caveat is the operational constraint that the service must stay LocalSystem (L3) — a documentation gap, not a design flaw.

- **Atomic ingest via a single-node replica-set transaction (#29).** Correctly chosen and correctly scaled. At ~0.33 ingest req/s, batches of ~10–20 intervals, transactions add ~1–2 ms/commit — negligible. The single-node RS has no fault tolerance, but that's inherent to single-VPS and is covered by the heartbeat backstop + daily mongodump. The 413→fresh-UUID-sub-batch rule is sound. The only implementation note (not a spec gap): the `batches` receipt insert must be the **serialization point** inside the transaction (attempt receipt insert; on duplicate-key error, abort and return `duplicate: true` with the stored `accepted`); a check-then-insert pattern would race — but this is an implementation detail for the skill, not a plan defect. (The **index placement** bug in C1 is separate and must be fixed.)

- **Closing the open interval on sleep/suspend (#33).** The **close-at-suspend** half is correct and important; monotonic-clock durations + wall-clock UTC stamps are the right pair. Only the **"reopen idle"** prescription is wrong (M4) — the fix is to re-derive state from a fresh sample, not to hardcode idle.

- **The redistributed Phase 7.** The tar-pit risk (minimax L9) is genuinely resolved: helper/session/sleep/clock → Ph1, buffer-cap/revocation/org-key → Ph2, browser variants/DPI → Ph3, AV/AppLocker → Ph5. Phase 7 is now a short, defensible residual (multi-monitor, PWAs/Electron, 100-device soak, revocation E2E, edge-case sweep) with concrete soak pass criteria (zero dropped batches, queries <2 s, stable VPS). Two minor sequencing notes (not blocking): (a) the multi-monitor **media-attribution** gap (M9) is a Phase 1/4 data-model concern, not a Phase 7 polish — acknowledge it earlier; (b) a light 10-agent ingest-concurrency check in Phase 2 would de-risk the Phase 7 100-device soak (the soak is currently the first load test).

- **Phase 0 scope.** Well-calibrated: the single-node-RS setup in Phase 0 correctly de-risks Phase 2 transactions; the self-contained publish smoke test and SQLite skeleton (iteration-1 asks) are placed early; admin auth + settings seed + dev seed are all present. Nothing in Phase 0 is over- or under-specified to block starting. Minor: the agent could compute + log `clockOffset` in the Phase 0 heartbeat (the flagging is Phase 2's job, but the measurement is free now).

---

## Sequencing

The redistributed phase order is **right**. Specifics:

- **Phase 0 → startable as-is.** No blockers. (C1, H1, H3 are doc fixes that should land before Phase 2, not Phase 0.)
- **Before Phase 2 (contract/data fixes):** C1 (drop `unique` from intervals index), H1 (token-rotation-on-loss), H3 (enroll rate limit / corp-IP allowlist), H5 (remap vs back-fill wording), M2 (batches TTL), M3 (status → liveness+flags). These are all doc edits; Phase 2 implements against the corrected spec.
- **Before Phase 1 (capture fixes):** H4 (hysteresis↔#7 reconciliation), M4 (resume state re-derivation), M5 (concurrent media+call gate), L5 (GSMTC resume grace), L6 (passive_media↔active hysteresis). Phase 1 builds the state machine against the reconciled rules.
- **Before Phase 4 (dashboard fixes):** H2 (capture device timezone now, even if bucketing stays Dhaka), M1 (clock-skew offset before bucketing), M9 (media-source attribution), L1/L2 (persistent + stateful gap record), L12 (multi-user flag).
- **Phase 5 (packaging):** L3 (LocalSystem constraint doc), L4 (MSI data preservation), M8 (clone runbook rule + heartbeat machineGuid).
- **Phase 6 (updates):** M6 (heartbeat never 426s; fallback manifest URL), L8 (`mandatory` semantics).

---

## Still-open from iteration 1 (not re-argued; tracking only)

These iteration-1 items were not absorbed into v1.1 and remain worth picking up. Listed once for tracking, not re-argued:

- MongoDB collection-level JSON Schema validators on `intervals`/`devices` (deepseek-v4-pro M2) — a Phase 2 hardening item.
- MongoDB migration strategy (deepseek L8 / mimo-v2-5-pro L4) — needed before schema drift starts; Phase 0 seed is fine, but Phase 2+ schema changes need a `migrations/` runner.
- Windows Event Log registration for IT visibility (minimax N3 / mimo N6) — Phase 5 runbook.
- `User-Agent: AtlasAgent/<version>` header on agent HTTP requests (qwen L2) — trivial, Phase 0/2.
- Log rotation limits (qwen L1) — Phase 0 detail.
- Client-generated interval id for helper-resend dedup on pipe reconnect (glm-5.2 L15) — the pipe-seq handles sample dedup, but a closed interval flushed at a pipe break can still double-store; Phase 1.

---

*End of round-2 review. No code or existing documents were modified; only this file was created.*
