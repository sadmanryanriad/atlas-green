# Atlas Green — Pre-Implementation Design Review

**Model:** mimo-v2-5-pro
**Date:** 2026-06-23

## Executive Summary

Atlas Green's architecture is pragmatic and well-matched to its constraints: ~100 company-owned Windows desktops, a small team, and a clear legal mandate. The Service+Helper split, interval aggregation model, ACK-before-purge sync, and per-device DPAPI tokens are all correct choices. The API contract is clean and the phased plan is sensible. However, several gaps will cause real pain if left unresolved before coding: (1) the named-pipe IPC protocol between helper and service is unspecified — this is Phase 1's core data path; (2) the `/enroll` idempotency semantics around token reissue are ambiguous, creating a security/reliability gap; (3) the dashboard has zero authentication, relying entirely on Cloudflare — a misconfiguration exposes the entire monitoring system; (4) `batchId` format lacks `deviceId` scoping, making cross-device collision a real data-loss risk at scale. Four High-severity items (UIA fragility on modern browsers, session-change handling, config validation on the agent, and the missing `batchId` confirmation in ingest response) should also be resolved before their respective phases.

---

## Critical

### C1 — Named-pipe IPC protocol between helper and service is completely unspecified
**File:** `docs/ARCHITECTURE.md` §2, `docs/PLAN.md` Phase 1
**Problem:** The architecture says the helper "streams samples back to the service (e.g., via a local named pipe)." The skill says "reports samples to the service over a local named pipe." Neither document specifies the protocol: framing (length-prefixed? newline-delimited?), serialization (JSON? MessagePack? protobuf?), direction (one-way helper→service or bidirectional for control?), buffer sizing, or what happens on pipe disconnect/reconnect. Phase 1's checklist item "Helper → service transport (e.g. named pipe)" leaves the entire transport design to the implementer. This is the core data path — every captured interval flows through it. Getting the framing wrong (e.g., using raw JSON without length-prefix) creates partial-read bugs that are extremely hard to diagnose in a silent background process.
**Recommendation:** Add a §2.1 to `docs/ARCHITECTURE.md` specifying: length-prefixed JSON messages (4-byte little-endian length header + UTF-8 JSON body), one-way helper→service direction, service creates the pipe with `PipeDirection.In`, helper connects with `PipeDirection.Out`, automatic reconnection on disconnect with 1-second backoff, and a max message size (e.g., 64KB per sample batch). This is a 10-line specification that eliminates an entire class of IPC bugs.

### C2 — `/enroll` idempotency semantics are ambiguous: does re-enrollment rotate the token?
**File:** `docs/API-CONTRACT.md` §1, `docs/ARCHITECTURE.md` §6
**Problem:** The contract says: "If `deviceId` is already enrolled, the server may return the existing record and (re)issue a token — enrollment is idempotent on `deviceId`." The word "may" and "(re)issue" create two incompatible readings: (a) the server returns the *existing* token (true idempotency — same input, same output), or (b) the server generates a *new* token and invalidates the old one (token rotation). This matters critically: interpretation (b) means every re-enrollment (e.g., after DPAPI corruption) silently revokes the previous token. If the agent had buffered intervals with the old token, those intervals are now unsyncable — the agent will get 401 on ingest, attempt re-enroll, get a new token, but the old buffered batches are tagged with the old token's era. The contract doesn't specify whether the agent should re-tag buffered rows after token rotation.
**Recommendation:** Specify explicitly: "On re-enrollment of an already-enrolled `deviceId`, the server returns the *existing* token (no rotation). Token rotation happens only via explicit admin revocation." This makes enrollment truly idempotent and avoids the buffered-data-era problem. If token rotation on re-enroll is desired, add a `tokenRotated: true` field to the response and specify that the agent must re-tag all buffered `synced=0` rows with the new token before the next sync.

### C3 — Dashboard has no authentication; a Cloudflare misconfiguration exposes the entire monitoring system
**File:** `docs/ARCHITECTURE.md` §8, `docs/PLAN.md` Phase 4
**Problem:** The dashboard is described as "admin dashboard UI for admins (employees never see it)" with Cloudflare providing "TLS, WAF, and DDoS protection." But there is no mention of any authentication on the dashboard itself — no login, no session, no IP allowlist, no Cloudflare Access. If the VPS's Next.js app is accessible directly (bypassing Cloudflare via IP), or if Cloudflare's DNS is misconfigured, anyone on the internet can see all employee activity data. The API endpoints use Bearer tokens, but the dashboard pages and their server-side data queries have zero auth.
**Recommendation:** Add Cloudflare Access (zero-trust) in front of the dashboard as the primary auth layer — it requires no code changes and supports email OTP or Google/GitHub SSO. Document in `docs/ARCHITECTURE.md` §8: "Dashboard is behind Cloudflare Access; unauthenticated requests get a 403." As a secondary layer, Next.js middleware should verify a session cookie for all `/dashboard/*` routes. This is not optional — employee activity data is sensitive.

### C4 — `batchId` format lacks `deviceId` scope; cross-device collision causes silent data loss
**File:** `docs/API-CONTRACT.md` §2
**Problem:** The `batchId` format `b-2026-06-22T09:10:00Z-001` is unique per agent but not globally. If the server's dedup index is a single unique index on `batchId` (as suggested by `atlas-console` skill §3 and `ARCHITECTURE.md` §7), two devices syncing at the same second with the same sequence number will collide. After a power outage, 100 machines boot simultaneously, all start at seq `001`, and the timestamp is identical. At 5-minute sync intervals, the collision window is the entire first batch. Even if the unique index is compound `{deviceId, batchId}`, the contract doesn't specify this — implementors will naturally assume `batchId` alone is the key.
**Recommendation:** Prepend `deviceId` to `batchId`: `b-7f3a9c2e-2026-06-22T09:10:00Z-001`. This makes the idempotency key self-scoping and the invariant obvious. Update the example in `docs/API-CONTRACT.md` §2. Alternatively, specify a compound unique index `{deviceId, batchId}` explicitly in `docs/ARCHITECTURE.md` §7 — but embedding the deviceId in the key is cheaper and self-documenting.

---

## High

### H1 — UI Automation for browser URLs will silently fail on modern browser configurations; fallback chain needed
**File:** `docs/PLAN.md` Phase 3, `docs/ARCHITECTURE.md` §2
**Problem:** The URL capture strategy relies entirely on UI Automation reading the browser address bar. Chrome's accessibility tree has historically been unreliable for address-bar extraction — the "Omnibox" control's UIA patterns vary by version and by whether the browser is running at elevated integrity. Edge's "enhanced security mode" runs in a stricter sandbox that may limit UIA access. Firefox has minimal UIA support — its address bar is not reliably exposed via the Windows accessibility tree. Opera and Brave inherit Chromium's UIA behavior but may have their own quirks. In incognito mode, some browsers change their UIA tree structure. The plan treats UIA as the sole mechanism with no fallback.
**Recommendation:** Add a 3-tier fallback chain to Phase 3: (1) UIA address-bar extraction (primary), (2) window-title regex parsing (Chrome/Edge/Brave include the page title and often the domain in the window title — `"Page Title - Google Chrome"`), (3) if both fail, store `domain: null, pageTitle: <window-title-as-is>` and flag the interval. Also: add browser name+version detection to the heartbeat health payload so the dashboard can warn "Chrome 130+ may block UIA on this device." Test specifically against Firefox and Edge enhanced-security mode in Phase 3's exit gate.

### H2 — Session-change handling (fast user switching, RDP) is deferred to Phase 7 but is core correctness for Phase 1
**File:** `docs/PLAN.md` Phase 1, Phase 7
**Problem:** Phase 1 says the service "launches + supervises" the helper with "restart on death / on user login." Phase 7 lists "fast user switching; RDP/remote sessions" as edge cases. But on Windows, a fast user switch doesn't kill the helper — it leaves it running in the *old* session while a new session becomes active. The helper continues sampling the old (now invisible) desktop, producing intervals for a session nobody is using. The service needs `WTSRegisterSessionNotification` to detect session switches and relaunch the helper into the new active session. If this isn't designed into the supervisor loop from Phase 1, Phase 7 requires rewriting the supervisor's core logic.
**Recommendation:** Move session-change detection (`WTS_SESSION_CHANGE` notifications) into Phase 1's supervisor design. Define the policy: "the helper always runs in the console/physical session; if an RDP session connects, ignore it (one employee, one desktop)." Add "helper relaunches correctly on fast user switch" to Phase 1's exit gate. The RDP-ignore policy is an explicit design decision that should be documented in `docs/ARCHITECTURE.md` §2.

### H3 — Agent does not validate remote config values; a bad config push can silently corrupt all data
**File:** `docs/API-CONTRACT.md` §1, §3
**Problem:** The heartbeat and enrollment responses include a `config` block (`sampleIntervalSec`, `idleThresholdSec`, `syncIntervalSec`, `heartbeatIntervalSec`, `bufferCapDays`). The agent "applies any returned config changes (remote tuning)." There is no mention of bounds validation. If an admin accidentally sets `idleThresholdSec: 5` (5 seconds), every employee becomes idle between mouse clicks. If `sampleIntervalSec: 0`, the agent busy-loops. If `syncIntervalSec: 1`, the agent hammers the server 100 times per minute. A config typo can silently corrupt the entire dataset.
**Recommendation:** Add config validation to the agent conventions skill: `sampleIntervalSec` clamped to [2, 60], `idleThresholdSec` to [60, 3600], `syncIntervalSec` to [60, 3600], `heartbeatIntervalSec` to [60, 3600], `bufferCapDays` to [7, 365]. Out-of-range values are ignored (keep the previous value) and logged as a warning. Also: document the valid ranges in `docs/API-CONTRACT.md` §1 alongside the config block so the server can validate before sending.

### H4 — Ingest response doesn't echo `batchId`; agent cannot confirm which batch was ACK'd
**File:** `docs/API-CONTRACT.md` §2
**Problem:** The ingest response returns `{ batchId, accepted, duplicate, serverTimeUtc }`. This is correct. However, the `batchId` in the response is the one the agent sent — this seems redundant but is actually critical for the agent's ACK logic. If the agent has multiple in-flight batches (future optimization) or if a network proxy mangles the response body, the agent needs to confirm which batch was ACK'd. The current contract is fine for single-batch-in-flight, but it doesn't specify that the response `batchId` MUST match the request `batchId`. An implementor might return the server's internal batch ID instead.
**Recommendation:** Add to `docs/API-CONTRACT.md` §2 behavior: "The response `batchId` MUST be identical to the request `batchId`. The agent should verify this match before purging." This costs nothing and prevents a subtle class of purge-the-wrong-batch bugs.

### H5 — `batchId` sequence counter persistence is unspecified; restart resets to 001, risking collision
**File:** `docs/API-CONTRACT.md` §2, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §4
**Problem:** The `batchId` format includes a sequence number (`-001`). The contract and skill say nothing about where this counter is persisted. If the agent crashes and restarts, the counter resets to `001`. Combined with a timestamp at minute granularity, the agent will generate the same `batchId` for its first post-restart batch as it did for its first-ever batch — potentially hours or days earlier. If the server has already stored the old batch, the new batch is silently dropped as a duplicate (data loss until the counter catches up).
**Recommendation:** Specify in `atlas-agent-conventions` skill §4: "Persist the last-used `batchId` sequence counter in the SQLite buffer (a single-row metadata table). On startup, read the last counter and continue from there." Also: add millisecond precision to the timestamp component (`b-<deviceId>-2026-06-22T09:10:00.123Z-001`) as a secondary safeguard.

### H6 — No sync jitter; 100 agents booting simultaneously will thundering-herd the server
**File:** `docs/ARCHITECTURE.md` §3, `docs/PLAN.md` Phase 2
**Problem:** The sync interval is a fixed 5 minutes with no jitter. After a power outage at 9 AM, all 100 agents boot, enroll, and start their sync timers simultaneously. Every 5 minutes, 100 `POST /ingest` requests arrive at exactly the same instant. Next.js on a VPS handles this at 100 agents, but the exponential backoff also lacks jitter — if the server returns a 500, all 100 agents retry in lockstep. This is a well-understood anti-pattern that becomes a real problem if the fleet grows or if MongoDB has a brief hiccup.
**Recommendation:** Add ±30s random jitter to the sync timer on each cycle. Add jitter to the exponential backoff (randomize within the backoff window, e.g., first retry at 5-10s instead of exactly 5s). Document in `docs/ARCHITECTURE.md` §3.

---

## Medium

### M1 — SQLite encryption method is unspecified; Phase 2 cannot begin without it
**File:** `docs/DECISIONS.md` #8, `docs/PLAN.md` Phase 0, Phase 2
**Problem:** Every document says "SQLite, encrypted" but no encryption mechanism is chosen. SQLite has no built-in encryption. The options have significant implications: SQLCipher requires a native/custom SQLite build (complicating self-contained publish), `Microsoft.Data.Sqlite` with SEE extension is paid, and application-layer encryption (encrypt payloads before insert) is fully managed but changes the buffer schema. Phase 2 is where the buffer is built, but the encryption decision affects the NuGet packages, the publish configuration, and the schema design. It should be decided before Phase 2 starts.
**Recommendation:** Add the encryption decision to `docs/DECISIONS.md`. For this threat model (standard users only, ACL-locked folder), application-layer encryption using .NET's `AesGcm` with a key stored via DPAPI machine scope is the simplest path — zero native dependencies, fully managed, and the ACL on ProgramData already blocks file-level read. If SQLCipher is chosen for defense-in-depth, document the native build implications for self-contained publish.

### M2 — `GET /api/v1/update/manifest` in the API contract conflicts with R2 hosting
**File:** `docs/API-CONTRACT.md` §4, `docs/ARCHITECTURE.md` §9
**Problem:** The API contract lists `GET /api/v1/update/manifest` as a console endpoint. The architecture diagram and update flow describe the manifest as hosted on Cloudflare R2. The heartbeat response already provides a `manifestUrl` pointing to R2. Having the manifest also served from the console creates two sources of truth — the console could serve a different version than R2 if the deploy is out of sync.
**Recommendation:** Remove `GET /api/v1/update/manifest` from the API contract. The heartbeat response's `update.manifestUrl` is the sole source for the manifest location. If the console needs to serve the manifest during early development (before R2 is set up), that's a deployment detail, not a contract term.

### M3 — Clock-skew detection is mentioned but no correction strategy is defined
**File:** `docs/API-CONTRACT.md` §5, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §6
**Problem:** The design says the agent reads `serverTimeUtc` "to detect/sync drift" but "never rewrites already-buffered timestamps." If an agent's clock is off by hours (CMOS battery failure, manual misconfiguration), all intervals have wrong timestamps permanently. The dashboard shows activity on the wrong day. Indefinite retention carries bad data forever. "Detect" without "correct" is operationally useless — someone has to notice the warning and manually fix it.
**Recommendation:** Add a clock-skew policy: if `|serverTimeUtc - localUtc| > 5 minutes`, log a warning and flag the device as "clock-skewed" in the heartbeat. If `> 1 hour`, store a `clockOffsetMs` on the device record (computed from two consecutive heartbeat round-trips to confirm stability). The dashboard applies the offset at display time and shows a "clock skew" badge on the device. This preserves raw data integrity while giving admins visibility.

### M4 — Agent binary is ~60-80 MB; update download over Bangladesh internet may never complete
**File:** `docs/ARCHITECTURE.md` §9, `docs/DECISIONS.md` #20
**Problem:** A self-contained .NET 8 single-file publish with Win32/UIA/GSMTC dependencies will be 60-80 MB. Over typical Bangladesh office internet (5-20 Mbps shared), downloading 80 MB takes 30-130 seconds per machine. With 100 machines and no range-request resume, a flaky connection means the agent retries from byte 0 each time. On a bad day, the update never completes.
**Recommendation:** Document the update download strategy: (1) download to a temp file with HTTP Range request support for resume, (2) verify SHA-256 of the complete file before swap, (3) if download fails 3 times, report failure on heartbeat and wait for next cycle. Also consider: serve the manifest with a `minSupportedVersion` check — agents older than this must update before doing anything else (force-update gate). This is already in the manifest schema but not in the agent's behavior spec.

### M5 — `machineHints` in enrollment collects hardware fingerprints that v1 doesn't use
**File:** `docs/API-CONTRACT.md` §1, `docs/DECISIONS.md` #14
**Problem:** The enrollment payload includes `machineHints.machineGuid` and `machineHints.mac`. Decision #14 explicitly says "no hardware dedup in v1." This data is collected, transmitted, and stored with no v1 purpose. It's a privacy liability (the server now has hardware fingerprints it doesn't need) and adds implementation complexity (MAC address retrieval across multiple NICs, error handling for missing values). If the argument is "collect now for future dedup," the history won't include machines enrolled before dedup is built.
**Recommendation:** Remove `machineHints` from the v1 enrollment schema. Re-add it in the phase where hardware dedup is actually implemented. This reduces v1 attack surface and implementation scope.

### M6 — Interval duration has no sanity bounds; a clock bug or aggregator defect can produce 86,400-second intervals
**File:** `docs/API-CONTRACT.md` §2
**Problem:** The aggregator flushes on app/domain/state change. If a user plays a YouTube video for 8 hours and the media detection works perfectly, the interval is 28,800 seconds — correct. But if the idle detection is broken and the user's input is never detected, a single interval could span an entire workday. The contract doesn't specify maximum `durationSec`. The ingest endpoint should have a ceiling to catch aggregator defects.
**Recommendation:** Add to `docs/API-CONTRACT.md` §2: intervals with `durationSec > 86400` (24h) should be rejected with `400`. The agent should also sanity-check intervals before buffering — if `durationSec > 43200` (12h), log a warning (the aggregator may be stuck). This catches bugs early rather than storing nonsensical data.

### M7 — No heartbeat retry on transient failure; a single network blip shows device offline for 15 minutes
**File:** `docs/API-CONTRACT.md` §3
**Problem:** The contract specifies no retry logic for heartbeats. If a heartbeat fails (DNS timeout, TCP reset), the agent waits 5 minutes for the next cycle. The server marks devices offline after ~3× the heartbeat interval (15 minutes). Two consecutive failures = 15 minutes of false "offline" status. For a monitoring system, "offline" is a signal that something is wrong — false positives erode trust.
**Recommendation:** Specify: heartbeats retry once after 30 seconds on failure, then wait for the next scheduled cycle. This is lightweight and prevents brief network hiccups from creating false offline flags.

### M8 — `osUsername` is denormalized on intervals with no rule for historical consistency
**File:** `docs/API-CONTRACT.md` §2, `docs/ARCHITECTURE.md` §7
**Problem:** `osUsername` appears on both the `devices` collection and every `intervals` document. If the OS username changes (new user logs in on a shared machine, or the user renames their account), historical intervals still carry the old username. New intervals carry the new one. The dashboard has no guidance on which to display or how to handle the transition.
**Recommendation:** Document: interval `osUsername` is a snapshot at capture time and is **never backfilled**. Dashboard queries that need the current username should read from `devices`. The interval's `osUsername` is kept for auditing (who was logged in when). Add a note to `docs/ARCHITECTURE.md` §7.

### M9 — Buffer-cap hit produces silent data gaps with no server-side visibility
**File:** `docs/ARCHITECTURE.md` §3, `docs/PLAN.md` Phase 2
**Problem:** When the buffer cap (~2-3 months) is hit, the agent drops oldest rows. There's no mention of reporting this to the server. If an agent is offline for 3 months and then reconnects, it silently drops the oldest data. The dashboard shows a gap with no explanation. Admins have no way to know data was lost vs. the machine was simply off.
**Recommendation:** When the agent drops rows due to cap pressure, include `droppedIntervals: { count, oldestUtc, newestUtc }` in the next heartbeat request. The dashboard heartbeat panel shows "Data gap: N intervals dropped due to buffer overflow" on that device. Add these fields to the heartbeat request schema in `docs/API-CONTRACT.md` §3.

---

## Low

### L1 — Phase 0 doesn't include SQLite skeleton; Phase 2 starts the buffer from scratch
**File:** `docs/PLAN.md` Phase 0, Phase 2
**Problem:** Phase 0 includes enrollment and heartbeat (the wire plumbing) but no SQLite buffer. Phase 2 adds the full buffer + sync loop. This means the SQLite library choice, connection management, schema migration approach, and encryption integration are all discovered in Phase 2. If there's a problem with the SQLite NuGet package or encryption approach, it's found late.
**Recommendation:** Add to Phase 0: "SQLite database creation with schema migration, insert/read test row." This is a 30-minute task that validates the library choice and migration approach early. No encryption or sync loop needed — just the plumbing.

### L2 — Self-contained publish is first tested in Phase 5; trimming/linking issues discovered late
**File:** `docs/PLAN.md` Phase 5
**Problem:** Phase 5 is the first time `dotnet publish --self-contained` is tested. If .NET trimming removes types needed by UIA or GSMTC at runtime, or if a native dependency is missing from the publish output, it's discovered after all capture/sync/dashboard logic is built. This is a high-cost failure.
**Recommendation:** Add a "Phase 0.5" or include in Phase 0: `dotnet publish --self-contained -r win-x64` and run the resulting `.exe` on a clean Windows VM (no .NET SDK). Verify it starts, generates a DeviceId, and sends a heartbeat. This is a 1-hour task that validates the publish pipeline before any real logic is built.

### L3 — Token revocation has no terminal state; agent buffers forever on a revoked device
**File:** `docs/API-CONTRACT.md` §5, `docs/ARCHITECTURE.md` §6
**Problem:** When a device token is revoked, the agent gets `401`, attempts re-enroll, fails (the device is revoked), and "backs off and keeps buffering." The buffer grows until the cap drops data. There's no terminal state — the agent never stops generating intervals it can never sync.
**Recommendation:** After 5 consecutive re-enroll failures over several hours, the agent enters a "revoked" state: stops generating new intervals, keeps existing buffer intact, heartbeats with `status: "revoked"` so the dashboard shows the device as revoked. Document in `docs/ARCHITECTURE.md` §6.

### L4 — MongoDB has no migration strategy; schema evolution will be ad-hoc
**File:** `docs/ARCHITECTURE.md` §7, `atlas-console/AGENTS.md` §Data model
**Problem:** As the product evolves, the interval document shape will change (new fields, renamed fields, type changes). There's no mention of a migration tool or strategy. Without it, ad-hoc changes accumulate and the database drifts from what the code expects.
**Recommendation:** Choose a lightweight migration approach: a `migrations/` folder with numbered scripts and a `migrations` collection to track applied migrations. For a project this size, a custom script runner is sufficient — no need for a heavy framework. Document in `atlas-console/AGENTS.md`.

### L5 — Logging guidance is incomplete; window titles may contain sensitive data
**File:** `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §1
**Problem:** The skill says "Never log secrets (org key, device token) or full page contents." But window titles can contain sensitive information (file paths, document names, personal search queries). Logging full window titles at `Information` level in production creates a privacy issue.
**Recommendation:** Add to the skill: "Log window titles at `Debug` level only in production. At `Information` level, log only `processName` and `appName`. Never log full page contents or URL query parameters."

### L6 — `dotnet-best-practices` skill contains irrelevant guidance (Semantic Kernel, ResourceManager)
**File:** `atlas-agent/.agents/skills/dotnet-best-practices/SKILL.md`
**Problem:** The community skill includes sections on "Semantic Kernel & AI Integration" and "ResourceManager for localized messages" that are completely irrelevant to a silent monitoring agent. An AI coding agent following this skill might add unnecessary dependencies or over-engineer localization.
**Recommendation:** The community skill is a template — it's fine to have it. But the `atlas-agent-conventions` skill should explicitly override irrelevant sections: "Ignore Semantic Kernel, ResourceManager, and localization sections from dotnet-best-practices. This agent has no AI features, no localized UI, and no user-facing strings."

### L7 — No `os` field in heartbeat; server can't track OS version across the fleet
**File:** `docs/API-CONTRACT.md` §3
**Problem:** The enrollment request includes `os: "Windows 11 Pro 23H2"`, but the heartbeat request doesn't. OS version can change after a Windows Update. The server has no way to track which agents are on which OS build — useful for debugging UIA issues or OS-specific bugs.
**Recommendation:** Add `os` to the heartbeat request body. Update it only when it changes (not every heartbeat).

---

## Nice-to-Have

### N1 — Interval `endUtc` is redundant with `startUtc` + `durationSec`
The interval schema includes `startUtc`, `endUtc`, and `durationSec`. Any two of these derive the third. Storing all three is a mild denormalization that can lead to inconsistency (what if `endUtc - startUtc != durationSec`?). Keeping `startUtc` + `durationSec` and deriving `endUtc` at query time is cleaner. However, storing `endUtc` makes range queries simpler — this is a style preference, not a bug.

### N2 — The manifest schema could include a changelog URL
The manifest has `latestVersion`, `package`, `minSupportedVersion`, and `releasedUtc`. Adding a `changelogUrl` field would let IT staff see what changed before approving an update. Low cost, moderate value.

### N3 — Named-pipe security descriptor should be documented
The named pipe between helper and service should have a security descriptor that restricts access to LocalSystem and the current user. Without it, any process on the machine can connect to the pipe and inject fake samples. This is outside the "standard user" threat model but good hygiene.

### N4 — Auto-update should support downgrade for emergency rollbacks
The manifest's `latestVersion` could be set to a prior version to trigger a rollback. The agent's update logic should handle `latestVersion < currentVersion` as a valid downgrade signal, not just an "already up to date" case.

### N5 — Consider MongoDB time-series collections for `intervals`
MongoDB 5.0+ supports time-series collections, which are optimized for exactly this workload (time-stamped, append-heavy reads). They offer better compression and faster range queries. Worth evaluating for the `intervals` collection, though the regular collection approach works fine at 100 agents.

### N6 — Agent should register with Windows Event Log for IT visibility
While the agent is silent to users, IT staff should be able to verify installation success/failure via Windows Event Log entries. The MSI custom action can write install/uninstall events to the Application log.

---

## Where the Design Is Solid

- **Three-state activity model.** Correctly distinguishes "watching media" from "away" — more honest than binary idle detection. The passive_media state captures the most common ambiguous scenario (YouTube/Spotify) without over-counting idle time.
- **ACK-before-purge sync with batchId idempotency.** The core reliability primitive is correctly designed. No data loss across failures/retries. The recommendations above are hardening, not redesign.
- **Per-device tokens via DPAPI machine scope.** Correct for this threat model. Single-device revocation without fleet-wide key rotation. DPAPI stops a standard user from reading the token.
- **DeviceId as opaque GUID with mutable attributes.** Identity never changes, labels can. Reformatted machine = new device is honest and simple. No premature dedup complexity.
- **Interval aggregation (Model B).** ~100-300 rows/day per device vs ~17k raw samples is a massive reduction with no information loss. Queries become trivial. Payloads are tiny.
- **Service + Helper split.** Session 0 isolation for persistence/secrets/network + user-session helper for observation. Helper holds no secrets, does no networking — correct separation of concerns.
- **Self-contained single-file .NET 8 publish.** No runtime install on 100 machines. The right operational posture for a fleet this size.
- **Cloudflare R2 for updates.** Zero egress cost, CDN-fast, decoupled from the VPS. At 80 MB × 100 agents, this is a real cost savings.
- **Phase 0 as thin vertical slice.** Proving enrollment → heartbeat → dashboard "online" before building capture is the right risk-reduction strategy. The exit gate is concrete and falsifiable.
- **Test policy: unit-test pure logic, manually verify OS-interop.** Pragmatic. UIA/GSMTC reads are inherently integration-level; mocking them produces tests that pass CI but fail on real hardware.
- **Console as single Next.js app.** At ~0.3 req/s, separating API from dashboard adds complexity for zero benefit. Cloudflare handles TLS/WAF/DDoS.

---

## Summary of Recommendations by Phase Impact

| Phase | Issues to resolve before starting |
|-------|----------------------------------|
| **Phase 0** | C1 (named-pipe protocol spec), L1 (SQLite skeleton), L2 (validate self-contained publish) |
| **Phase 1** | H2 (session-change detection in supervisor), H3 (config validation bounds) |
| **Phase 2** | C4 (batchId deviceId scoping), H5 (batchId seq persistence), H6 (sync jitter), M1 (SQLite encryption decision), M9 (cap-hit reporting) |
| **Phase 3** | H1 (UIA fallback chain), L5 (window-title log redaction) |
| **Phase 4** | C3 (dashboard authentication), M3 (timezone aggregation semantics) |
| **Phase 5** | L3 (revoked-device terminal state) |
| **Phase 6** | M2 (remove redundant manifest endpoint), M4 (download resume), M7 (heartbeat retry) |
| **Phase 7** | M5 (remove v1 machineHints), M6 (interval duration bounds), M8 (osUsername snapshot rule), L4 (DB migration strategy), L6 (override irrelevant skill sections), L7 (os in heartbeat) |
