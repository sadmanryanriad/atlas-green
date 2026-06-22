# Atlas Green — Pre-Implementation Design Review

**Model:** minimax-m3
**Date:** 2026-06-23

## Executive Summary

Atlas Green is, in the main, a thoughtful and well-suited design for the problem: ~100 company-owned Windows machines, a small team, AI-agent-built code, and a legitimate monitoring mandate. The Service+Helper split, interval aggregation (Model B), the 3-state model, ACK-before-purge + `batchId` idempotency, DPAPI-scoped per-device tokens, and the contract-first / doc-first discipline are all the right calls. The other three reviews (deepseek-v4-pro, glm-5.2, mimo-v2-5-pro) have already covered the high-profile risks well; this review deliberately does **not** repeat them and instead focuses on what they missed: media-detection false positives on audio calls, URL normalization, elevated-process foreground window bypass, idle-threshold hysteresis, the device `status` state machine, the missing `employeeId` on intervals, the no-rate-limiting gap, and several Windows-specific traps (AppLocker, DPI scaling, MSE audio, portable apps). The most material new finding is that **WASAPI peak audio as the "media" fallback will mark every Teams/Zoom call as `passive_media`** — which is factually wrong and operationally embarrassing — and needs to be disambiguated from true media playback before Phase 1.

---

## Critical

None. Phase 0 can start as planned. The items below should be resolved before the phases that own them.

---

## High

### H1 — WASAPI audio-peak fallback misclassifies calls, system sounds, and notification chimes as `passive_media`
**File:** `docs/ARCHITECTURE.md` §4, `docs/AGENTS.md` §3, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §3, `docs/DECISIONS.md` #7
**Problem:** Decision #7 says WASAPI audio-output is the "fallback signal" when GSMTC reports no Playing session. But WASAPI peak audio fires on **any** non-muted output: Teams/Zoom/Meet calls (the remote party's voice), Windows notification sounds, accessibility screen-reader TTS, music in a backgrounded tab playing through a Bluetooth headset, and even microphone-monitored audio on some USB audio devices. All of these get marked `passive_media`. A Bangladesh support agent on a 3-hour client call appears in the dashboard as "passive_media for 3 hours" — which is operationally wrong (they were working, just not typing) and legally dangerous for a product whose value is "honest picture of how work time is spent." This isn't an edge case; it's the single most common "is the user working?" signal in 2026.
**Recommendation:** Replace the WASAPI audio-peak fallback with a **two-condition check**: (a) GSMTC reports a session with `PlaybackStatus == Playing`, **OR** (b) WASAPI detects audio **AND** GSMTC has at least one *non-call* media session registered (e.g., Spotify, YouTube) within the last N minutes. Concretely, maintain a rolling list of "non-call media sources" populated from GSMTC's `SourceAppUserModelId`; only treat `passive_media` as the fallback audio if the most recent GSMTC source isn't a communications app (Teams, Zoom, Meet, Skype, Discord, Slack-huddle). If neither GSMTC nor a non-call app is the source, fall through to `idle`. This adds 30 lines of code and removes the worst false positive. Also: explicitly document the known limitation that "watching a downloaded movie in a non-GSMTC-registered player (e.g., VLC) while the window is in the background" may be misclassified, and treat that as a Phase 7 known limit rather than a correctness bug.

### H2 — URL normalization is not defined; `www.x.com` / `x.com` / `m.x.com` fragment the dashboards
**File:** `docs/AGENTS.md` §3, `docs/API-CONTRACT.md` §2, `docs/ARCHITECTURE.md` §3
**Problem:** The interval schema has `domain: "youtube.com"`. But raw UIA / window-title extraction can yield `www.youtube.com`, `m.youtube.com`, `youtube.com`, `youtu.be`, and `music.youtube.com`. Without a normalization step, "time on YouTube" requires `WHERE domain IN ('youtube.com', 'www.youtube.com', 'm.youtube.com', 'youtu.be', 'music.youtube.com')` everywhere, and the company-wide overview undercounts by the fragment that happens to be `m.youtube.com`. The same applies to international variants (`google.com.bd` vs `google.com`), subdomains-as-products (`docs.google.com` vs `drive.google.com` vs `mail.google.com`), and the Public Suffix List (a `co.uk` domain is conceptually different from `example.co.uk`).
**Recommendation:** Define a single normalization function used by the agent *before* writing to the buffer (so both `domain` and the aggregator's "domain-change" trigger see normalized values): lowercase, strip leading `www.`, then apply a small **eTLD+1** extraction via a public-suffix-list library (e.g., Nager.PublicSuffix or Nito.AsyncEx.SmartThreadPool — anything that ships the PSL data). The displayed `domain` is the eTLD+1 (`youtube.com`, `google.co.uk`). Store an optional `subdomain` field separately if the dashboard ever wants to distinguish `docs.google.com` from `mail.google.com`. Add a Phase 3 unit-test checklist item: "given a list of 20+ real-world URL variants, normalization yields the expected eTLD+1 for each." Pin the PSL version in the agent so behavior is reproducible.

### H3 — `GetForegroundWindow` is hijacked by any elevated/integrity-higher process; the helper sees the wrong window
**File:** `docs/ARCHITECTURE.md` §2, `docs/PLAN.md` Phase 1
**Problem:** A standard user cannot normally elevate. But the helper itself is launched by the service as a child process inheriting the user's token — fine. However: (a) the **consent.exe / UAC prompt** that appears when an installer elevates is itself a foreground window at a higher integrity; `GetForegroundWindow` returns it for the entire UAC duration, producing a 30-90 second interval on `Consent.exe`. (b) A user who right-clicks Task Manager and chooses "Run as administrator" (allowed on every Windows install) shifts the foreground to elevated Task Manager for as long as it's open. (c) Windows-Sandbox, WSLg windows, and the new Terminal app all have elevated sub-windows that can briefly own the foreground. Each of these is a **legitimate user action** that hijacks the foreground, and the helper has no way to know "the user is just looking at a consent dialog" vs "the user switched to a different app." The result: intervals on `Consent.exe` and elevated `TaskMgr.exe` pollute the report and inflate apparent time on system utilities.
**Recommendation:** Filter the foreground process list against a small **deny-list of known shell/dialog processes** at the helper level: `Consent.exe`, `CredentialUIBroker.exe`, `LogonUI.exe`, `TaskMgr.exe` (when running elevated — detected by checking the process token integrity level via `GetTokenInformation(TokenIntegrityLevel)`), `WerFault.exe`, `Wusa.exe`, and any process whose `SessionId` differs from the helper's. For these, **retain the previously-captured foreground app** for the duration (or close the interval with the real last-known app and mark `state: "system_dialog"` if a fourth state is desired — though I'd argue against the complexity). Add a Phase 1 unit test against a static list of well-known hijacker process names.

### H4 — Idle threshold (5 min) lacks hysteresis; a single mouse jiggle toggles state every sample
**File:** `docs/DECISIONS.md` #6, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §3
**Problem:** The state machine has crisp thresholds: `active` if input within 300s, `idle` if not. Real user behavior is messy: a user on a 5-minute phone call might press the spacebar once at the 4:30 mark to keep Teams "active" (or the OS might register a phantom input from a USB device polling). At sample time 4:59, `idleTimeSec=299` → `active`. At sample time 5:01, `idleTimeSec=301` → `idle`. The single sample of input flipped the state, and the next sample flips it back. This creates an oscillating state machine that flushes an interval every 3-5 seconds instead of every 10+ minutes, **defeating the entire point of interval aggregation** (Model B's value is "rare flushes"). Worse, with `active` being the "honest" state and idle being the "away" state, the bouncing user appears as 50% active / 50% idle in the dashboard.
**Recommendation:** Add **hysteresis**: become `idle` when `idleTimeSec > 300`, but **stay** `idle` until `idleTimeSec < 30` (i.e., 30 seconds of sustained input is required to leave idle). Symmetric hysteresis for `passive_media` (stay in `active` for at least 5s after the last input to avoid bounce). This is one extra `bool _isInStateSince` field in the state machine, fully unit-testable. The existing Phase 1 unit test for the state machine should be extended to cover the oscillating-input scenario and verify only one state transition is produced.

### H5 — `status` field on `devices` is mentioned but its state machine is undefined
**File:** `docs/ARCHITECTURE.md` §7, `docs/API-CONTRACT.md` §3
**Problem:** Devices have a `status` field. The contract says "Devices not seen within ~3× the heartbeat interval are shown `offline`." But the dashboard needs at least: `online`, `offline`, `archived`, `revoked`, `pending_enrollment` (the device sent one heartbeat but no intervals yet), `agent_outdated` (below `minSupportedVersion`), and possibly `clock_skewed` (recommended in mimo M3 / deepseek H3). Without an explicit state machine, the dashboard's "online/offline" badge is computed ad-hoc from `lastSeenAt`, while `archived` is a boolean, and `revoked` is a `revokedAt` on `deviceTokens` — three independent sources of truth that don't compose. When an admin "archives" a device, is the token revoked? What shows in the heartbeat panel — archived, offline, or both?
**Recommendation:** Define the device state machine explicitly in `docs/ARCHITECTURE.md` §7 as a single `status` enum with the documented transitions. Recommended set: `pending | online | offline | outdated | clock_skewed | revoked | archived`. Server is the source of truth (the agent never tells the server "I'm offline"); admin actions (revoke, archive) are atomic state changes. Add to `docs/PLAN.md` Phase 4 as a checklist item: "device status state machine implemented; dashboard reads only `status`, not derived fields." This unblocks several other findings (M3/M9 from mimo, M8 from deepseek).

### H6 — Server has no rate limiting on `/ingest` or `/enroll`; a single misbehaving agent can DoS the fleet endpoint
**File:** `docs/API-CONTRACT.md` §1, §2, §5
**Problem:** The contract mentions `429` as a response code but defines no rate limits. A bug in the agent's sync loop (e.g., backoff forgets the multiplier) could have all 100 agents hammering `/ingest` at 1000 req/s. A rogue `org key` use (see glm-5.2 M15) could create a fake device and submit malformed batches to fill MongoDB. The contract currently treats 429 as a signal the agent backs off — but there's no per-device limit and no global limit. Cloudflare helps at the network edge but doesn't apply per-token throttling.
**Recommendation:** Specify rate limits in the contract. Recommended: per-device, `/ingest` ≤ 1 req per 30s (well above the 5-min sync), `/heartbeat` ≤ 1 req per 60s, `/enroll` ≤ 1 req per hour per IP. Globally: 200 req/s per IP across all endpoints. The server returns `429` with `Retry-After` (already implied). Add a Phase 2 checklist item: "rate limiting per device + per IP implemented; tested with a synthetic burst." This is cheap with `next-rate-limit` or a simple token-bucket middleware.

### H7 — Dashboard has no admin authentication (re-stated as High; not addressed anywhere in the plan)
**File:** `docs/ARCHITECTURE.md` §8, `docs/PLAN.md` (no phase), `atlas-console/AGENTS.md`
**Problem:** All three other reviews flag this; the plan has not absorbed it. The dashboard is the *only* way to read the captured data, and there's no login. The threat model is "standard users," but the dashboard is on a public-internet-routable VPS. Cloudflare Access is free and zero-config; the only reason not to do it is that no one assigned the task. For a product whose stated goal is "an honest picture of how work time is spent" (AGENTS.md §1), shipping the dashboard open is the single most damaging privacy failure possible short of leaking the MongoDB.
**Recommendation:** Add a **Phase 0** checklist item (before any real data lands): "Dashboard is behind **Cloudflare Access** with email-OTP for the two admins; `/api/v1/*` endpoints additionally check Bearer tokens (existing); unauthenticated dashboard requests are 403." Phase 4 can later upgrade to a real NextAuth flow with per-admin roles, but Cloudflare Access is the v1 minimum. The Phase 0 exit gate must include: "an unauthenticated browser session cannot view `/dashboard`."

### H8 — `employeeId` is not denormalized on intervals; every dashboard query requires a join
**File:** `docs/ARCHITECTURE.md` §7, `atlas-console/.agents/skills/atlas-api-contract/SKILL.md` §7
**Problem:** The interval schema is `{ deviceId, employeeId?, startUtc, ... }` with `employeeId` *optional*. The current device→employee mapping can change (admin re-maps a device). But for dashboard queries (per-employee daily summary, company overview), the most common access pattern is "all intervals for employee X in date range Y." If `employeeId` is only on the device record, every interval query joins `intervals → devices → employees`. The whole point of storing `osUsername` on every interval (denormalized for query speed) is undermined if `employeeId` isn't denormalized the same way.
**Recommendation:** Make `employeeId` a **required field at ingest time** — the server resolves it from the current device mapping and stores it. Add an `employeeIdAssignedAt` timestamp on intervals for audit. When an admin re-maps a device, **all new intervals** get the new `employeeId`; historical intervals keep the old one (with the snapshot). Add a `bulkReassignIntervals(employeeIdFrom, employeeIdTo, rangeUtc?)` admin endpoint for the rare case where the past also needs to be reattributed (audit-logged). Update the indexes: `{employeeId, startUtc}` becomes the primary dashboard index. This costs one field at ingest and one optional admin action; it eliminates a join on the hottest query path.

---

## Medium

### M1 — Aggregator's "flush on app change" misses "new window in same app" (e.g., two Chrome windows on different domains)
**File:** `docs/ARCHITECTURE.md` §3, `docs/DECISIONS.md` #5
**Problem:** The aggregator flushes on **app**, **domain**, or **state** change. A user has two Chrome windows open: one on `github.com`, one on `stackoverflow.com`. Switching between them changes the *foreground window* but not the *process* (`chrome.exe`) and not (per the current spec) the *domain* (the helper only samples the foreground). Result: a single interval that spans "github.com" then "stackoverflow.com" with a `domain: "github.com"` snapshot. Time on Stack Overflow is silently attributed to GitHub. The user looks productive on GitHub when they were actually searching for help.
**Recommendation:** Define the aggregator's flush conditions to also fire on **foreground window handle change** (use `GetForegroundWindow`'s HWND, not just process name). When a same-process switch happens, the helper reads the new window's domain via UIA and either (a) flushes the old interval with the old domain and starts a new one with the new domain (preferred), or (b) keeps a per-HWND last-seen-domain map and re-reads on switch. Add a Phase 1 checklist item: "two Chrome windows on different domains produce two intervals." This is a 30-line change but it preserves the integrity of "time on X."

### M2 — Window-title and page-title fields can carry arbitrary-length sensitive content; length cap undefined
**File:** `docs/API-CONTRACT.md` §2, `docs/ARCHITECTURE.md` §7
**Problem:** `windowTitle` and `pageTitle` come from the OS without bounds. A long Stack Overflow question title is 200-300 chars; a long Notepad++ document is 2000+ chars. Storing them unbounded is a MongoDB write-amplification risk (and a UI rendering mess), but more importantly, these fields often contain **sensitive content** (email subject lines from webmail tabs, file paths, customer names in support tickets, search queries). The plan says "domain + page title" (Decision #2) is "low noise" but does not address the sensitivity. With 100 employees' page titles in MongoDB, the dataset is essentially a keylogger for browser activity. Combine with no dashboard auth (H7) and this is a serious privacy surface.
**Recommendation:** Cap `pageTitle` at **256 characters** and `windowTitle` at **512 characters** at the agent (truncate with `…` ellipsis). Add a separate boolean `pageTitleTruncated: true` to the interval so admins know. Document the cap in the contract. Consider (as a v1.1 enhancement) a per-device setting to **disable pageTitle storage entirely** for employees handling highly sensitive content (HR, finance), replacing it with `domain` only. At the very least, add the pageTitle cap now.

### M3 — `appName` mapping from `processName` is fragile (portable apps, renamed exes, fork-bombs of `node.exe`)
**File:** `docs/API-CONTRACT.md` §2, `docs/ARCHITECTURE.md` §7
**Problem:** The interval schema separates `appName: "Visual Studio Code"` (friendly) from `processName: "Code.exe"` (real). The agent must do this mapping, but no `appCatalog` or `processMap` is defined. Several common cases will misbehave: (a) `code.exe` from a portable install is still `code.exe` but lives in `D:\tools\`; (b) a developer with multiple VS Code versions has two `Code.exe`; (c) every Electron app is its own `myapp.exe` with no obvious friendly name unless the FileDescription is read; (d) a user running multiple Node processes has N rows of `node.exe` with no way to distinguish which is "the dev server" vs "the build script." The dashboard's "top apps" chart will show "node.exe (45 min)" as a single bucket, lumping all dev work together.
**Recommendation:** Read `FileDescription` from the process's PE header (via `GetFileVersionInfo` + `VerQueryValue` on the .exe) as the default `appName`. For processes with a generic `FileDescription` (e.g., "Node.js"), fall back to the executable filename without extension. For Electron apps, FileDescription is usually set to the app name. Cache the mapping per process path (don't re-read the PE header every sample). Add a Phase 1 unit test: a small static catalog of `{path → expected appName}` pairs. Document in the agent skill.

### M4 — `Settings` collection is mentioned but never specified
**File:** `docs/ARCHITECTURE.md` §7, `docs/DECISIONS.md` #21
**Problem:** Decision #21 implies per-PC archive/delete; `deviceTokens` has a `revokedAt`; org keys are stored "in settings" per the architecture doc. But `settings: { org key(s), default agent config, retention policy, etc. }` is a one-line placeholder with no schema. Org-key rotation (H6 in deepseek), dashboard defaults, retention policies, and rate-limit thresholds (H6 above) all live here. Without a schema, each of these gets stuffed into `settings` as ad-hoc keys and the next implementer guesses the shape.
**Recommendation:** Define the `settings` schema in `docs/ARCHITECTURE.md` §7 before Phase 0 ends: `{ _id: "global" | string, key, value, updatedBy, updatedAt }` with a few canonical entries: `orgKeys: [{id, valueHash, createdAt, expiresAt}]`, `agentDefaults: {sampleIntervalSec, idleThresholdSec, ...}`, `rateLimits: {ingestPerDevice, heartbeatPerDevice, enrollPerIp}`, `retention: {intervalDays: null, screenshotDays: 7}`. Use document-per-key rather than a single doc so updates are atomic per key. Add to Phase 0 checklist.

### M5 — Aggregator doesn't close the open interval on system suspend; sleep-then-resume produces 8+ hour phantom intervals
**File:** `docs/PLAN.md` Phase 7, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §3
**Problem:** M3 in glm-5.2 partially covers this (sleep events), but the consequence is sharper than stated. If the machine sleeps mid-interval (user is active on Chrome, walks away, laptop sleeps), `GetLastInputInfo` doesn't advance but `DateTime.UtcNow` does. On resume, the helper sees `idleTimeSec = 4 hours`, state goes `idle`, and the previous `active` interval is *closed* at resume time with a 4-hour duration. The dashboard then shows "active on Chrome for 4 hours" while the user was asleep. The clock-skew problem (mimo M3, deepseek H3) compounds: the wall clock is correct, but the *interval duration* is wrong because the open interval spanned a sleep.
**Recommendation:** In Phase 1, subscribe to `SystemEvents.PowerModeChanged` (or `WM_POWERBROADCAST`) and **close the current interval at the suspend timestamp** (use a `SleepUntilUtc` recorded just before sleep, derived from the last sample's `DateTime.UtcNow` + the elapsed-time-from-monotonic-clock delta). On resume, start a new interval with `state: "idle"` and `idleReason: "post_sleep"`. This is the hook M3 in glm-5.2 recommends; the additional requirement is to *close* the pre-sleep interval, not just *start* a post-resume one.

### M6 — MediaSource Extensions (MSE) audio in browsers often doesn't register as a GSMTC session
**File:** `docs/DECISIONS.md` #7, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §3
**Problem:** GSMTC tracks media sessions that the application explicitly registers via the Windows.Media.Control APIs. Browsers register their **main tab audio** as a GSMTC session — but only when the tab is the active media source. **MSE-based players** (Twitch, many news sites with HTML5 players, embedded YouTube via iframe on third-party sites) often don't register reliably. Result: a user watching a Twitch stream for 30 minutes produces a 30-minute `idle` interval even though they were "watching media." This is a silent-undercount of `passive_media` and an over-count of `idle`, which the design explicitly wanted to avoid.
**Recommendation:** As a complement to GSMTC, monitor **process-level audio activity** for known browser processes: a small per-process rolling RMS of WASAPI output, scoped to the browser's session. If a browser process is producing sustained audio (>2s of non-trivial RMS), treat it as `passive_media` regardless of GSMTC. Combine with H1's call-detection to avoid the false-positive. This adds ~50 lines and a per-process audio session handle (the agent must request it from WASAPI). Document as "GSMTC is the primary signal; per-process audio is a browser-specific supplement."

### M7 — No support for distinguishing multiple OS sessions on one machine (RDP + console)
**File:** `docs/ARCHITECTURE.md` §2, §5
**Problem:** glm-5.2 M1 and M11 cover the helper-per-session launch and the employee-attribution ambiguity. The missing piece is the **interval `sessionId` field**: without it, an admin looking at a machine with both a console session (employee) and an RDP session (IT doing maintenance) cannot tell whose activity is whose. The current plan silently attributes everything to "the device's user" — which the design says is one per machine, but the architecture also acknowledges RDP/fast-user-switching as Phase 7 edge cases.
**Recommendation:** Add `sessionId: int` to the interval schema (Windows session ID, 0 for service, 1+ for user sessions). The helper tags each sample with the session it's running in. The dashboard can group intervals by `sessionId` to show "console activity vs RDP activity." This is one field, free at capture time, and it makes M11 from glm-5.2 actually answerable.

### M8 — Page-title length cap, window-title redaction, and "no full URL paths" deserve an explicit logging & storage policy doc
**File:** `atlas-agent/AGENTS.md` §Conventions, `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §1
**Problem:** Deepseek L5 and mimo L5 both note that window titles can contain sensitive content. The fix is scattered: cap on page titles (M2 above), redaction in logs (deepseek L5), no full URL paths in storage (Decision #2). None of these are in one place. AI agents implementing Phase 1+ will need to discover them across four documents.
**Recommendation:** Add a single **"Privacy & Logging Policy"** section to `atlas-agent-conventions` SKILL.md (or a new top-level `docs/PRIVACY.md`) consolidating: (a) what is never logged at any level (secrets, query-string URLs, full email subjects); (b) what is logged only at `Debug` (window titles in production); (c) what is stored truncated (M2 caps); (d) what is **never stored** (URL query strings, file paths from open/save dialogs, clipboard). One doc, four bullets, one place to look.

### M9 — Phase 0 doesn't seed any test data; a fresh clone has an empty dashboard
**File:** `docs/PLAN.md` Phase 0
**Problem:** Phase 0's exit gate is "the device appears online in the dashboard and flips offline when the agent stops." With one device, the dashboard is a single row. There's no seeding of: a sample device that's been offline 30 minutes, a sample device with intervals, a sample device with multiple state transitions. Every Phase 4 view (timeline, summary) will be built against empty data and only exercised against real data after the first 100 machines are installed. This is workable but means the dashboard UI is essentially unverified for 3-4 months.
**Recommendation:** Add a Phase 0 checklist item: "Seed script (`scripts/seed-dev.ts` in console repo) inserts 3 sample devices, one with 24 hours of synthetic intervals covering all 3 states and 5 sample domains. The dev environment loads this on first boot if `intervals` is empty." Cheap insurance; it lets the dashboard developer iterate against realistic data immediately. Mark seed data with `deviceId` starting with `seed-` and a UI badge so it's never confused with real data.

### M10 — Self-update has no config-migration step; v1.0 → v1.1 with a new mandatory config field will silently use defaults
**File:** `docs/ARCHITECTURE.md` §9, `docs/PLAN.md` Phase 6
**Problem:** Auto-update is designed as "swap binary, restart." But the agent's config (`sampleIntervalSec`, `idleThresholdSec`, etc.) is server-tunable via the heartbeat. If v1.1 introduces a new mandatory config field (e.g., `mediaDetectionMode: "gsmtc" | "gsmtc+wasapi"`) and the server is still sending the v1.0 config, the agent either (a) uses a hardcoded default that may not match the server's expectations, or (b) crashes on config deserialization. There's no "schema version" on the config block.
**Recommendation:** Add a `configSchemaVersion: 1` field to the config block returned by `/enroll` and `/heartbeat`. The agent's config-loader checks the version; if newer than the agent supports, it logs and ignores unknown fields (forward-compat) or applies defaults (backward-compat). The agent sends `agentConfigSchemaVersion` in the heartbeat so the server can decide whether the agent is "old" and needs to be nudged to update. Document in the manifest schema and the contract.

### M11 — AppLocker / WDAC policies on company machines can block the unsigned-or-self-signed agent binary
**File:** `docs/PLAN.md` Phase 5, `docs/DECISIONS.md` #18
**Problem:** Many Bangladesh enterprise Windows installs enforce AppLocker or Windows Defender Application Control (WDAC) policies that only allow Microsoft-signed or explicitly allow-listed binaries to run. The agent ships as a self-contained single-file .NET binary that is at best self-signed (Decision #18). On a hardened machine, the service install may succeed but the helper launch will be blocked by WDAC, producing no helper, no data, and a "service online" false positive (M2 in glm-5.2). The decision says "push cert to company machines" but AppLocker policy updates are a separate admin action that IT must perform in parallel with the MSI install.
**Recommendation:** Add to the Phase 5 checklist: "Verify on a WDAC/AppLocker-hardened reference machine: the agent binary is allow-listed, the helper launches, the service can read/write its ProgramData folder, and DPAPI works under the restricted token." Document the Group Policy / AppLocker rule snippet needed to allow-list the agent binary path and the certificate. Consider an MSI option to **add the AppLocker rule during install** (custom action) so IT doesn't need a separate step.

### M12 — High-DPI scaling can break UIA tree traversal
**File:** `docs/PLAN.md` Phase 3
**Problem:** On a 4K display at 200% DPI scaling, UI Automation's coordinate system and control rectangles are reported in "device-independent pixels" but some browser address-bar controls (particularly older Chromium versions) return mismatched bounding boxes. The `BoundingRectangleProperty` and `ClickablePointProperty` can be off by the DPI factor, and the `.NET 8` UIA client sometimes returns null on `ClickablePoint`. This isn't a capture bug (we don't click), but **name/value properties** for some controls in the address-bar path can be empty or wrong at non-100% DPI.
**Recommendation:** In the helper, call `AutomationElement.BoundingRectangleProperty` defensively (catch `ElementNotAvailableException`). For DPI: declare the helper manifest `dpiAware = true/pm` in the app.manifest so Windows returns the right coordinate system to UIA. Add to the Phase 3 exit gate: "verified on a 4K display at 200% scaling." Document the test matrix.

### M13 — `org key` rotation has no rollback if the new key is leaked during distribution
**File:** `docs/DECISIONS.md` #15, deepseek H6
**Problem:** Deepseek's H6 recommends a transition window with both old + new keys valid. The missing piece: what if the new key is **itself** compromised (e.g., the new MSI is built and the build environment leaks the key)? Currently there's no way to "un-rotate" or to invalidate a not-yet-shipped key. The settings model (M4 above) supports multiple active keys with `expiresAt` — but no provision for "this key is burned, never re-enable."
**Recommendation:** Add a `revokedAt` field to the org-key entry in `settings` (complementing `expiresAt`). Revoked keys are rejected immediately even if not yet expired. Add a `keyId` so rotation logs can identify which key an enrollment used. Document the "burn a key" procedure in the runbook.

---

## Low

### L1 — The `viewer's local time` rendering is straightforward in CSR but tricky in SSR Next.js
**File:** `docs/ARCHITECTURE.md` §8, `atlas-console/AGENTS.md` §Time
**Problem:** Decision #22 says "render in the viewer's local time." In a Next.js app with server components, the server doesn't know the viewer's timezone. Approaches: (a) client-side hydration with `Intl.DateTimeFormat().resolvedOptions().timeZone` (works for all modern browsers), (b) a `<TimezoneContext>` provider, (c) send `timezone` as a header on dashboard requests. Without an explicit choice, two implementers will pick two approaches and the dashboard will flicker between SSR UTC and client local on first paint.
**Recommendation:** Pin the approach in `atlas-console/AGENTS.md`: client-side conversion only, with a `useViewerTimezone()` hook that returns `Intl.DateTimeFormat().resolvedOptions().timeZone` and re-renders when it changes. Add a Phase 4 checklist item: "all dashboard times render in viewer's local zone on first paint without SSR/CSR mismatch."

### L2 — `Intervals` would benefit from MongoDB time-series collection
**File:** `docs/ARCHITECTURE.md` §7
**Problem:** M16 in glm-5.2 anticipates aggregation slowdowns and recommends a pre-aggregated `dailySummary` collection. A cheaper first step: switch the `intervals` collection to a **MongoDB time-series collection** (available in MongoDB 5.0+, requires replica set in production but works on a single mongod for dev). Time-series collections give ~10x compression, faster range scans, and are designed exactly for this workload (append-heavy, time-ranged queries). The dashboard queries that touch intervals (`{deviceId|employeeId, startUtc: {$gte, $lte}}`) become significantly faster without any aggregation pipeline.
**Recommendation:** When the console repo is created, set the `intervals` collection up as a time-series collection with `timeField: "startUtc"`, `metaField: { deviceId, employeeId }`, `granularity: "hours"`. Test the migration before production. Note: time-series collections don't support arbitrary secondary indexes; the `{deviceId, startUtc}` lookup pattern works but `{domain}` standalone does not — which actually dovetails with deepseek M13's recommendation to drop underused indexes.

### L3 — `dashboard` route doesn't have its own auth path in the Next.js app
**File:** `docs/ARCHITECTURE.md` §8
**Problem:** Even with Cloudflare Access in front (H7), the Next.js app should not assume the proxy does all the work. A request that bypasses Cloudflare (direct IP, misconfigured DNS) should be rejected by the app itself. A simple `middleware.ts` that checks for a Cloudflare-Access-issued JWT or a NextAuth session cookie on every `/dashboard/*` route handles this.
**Recommendation:** Add to Phase 0 (after H7): "Next.js middleware rejects unauthenticated requests to `/dashboard/*`." This is a 20-line `middleware.ts`.

### L4 — `intervals.receivedAt` semantics: server-set vs agent-set
**File:** `docs/ARCHITECTURE.md` §7, `docs/API-CONTRACT.md` §2
**Problem:** The `intervals` schema includes `receivedAt`. The contract doesn't say who sets it. If the agent sets it (using its `DateTime.UtcNow` at send time), clock-skewed agents produce misordered data. If the server sets it at insert, it's the "actual ingest time," which is more useful for the dashboard.
**Recommendation:** Specify: `receivedAt` is **server-set at insert** (Bangkok-time consistent). Add a `capturedAt` field on the interval (already implicit via `startUtc`/`endUtc`) for the agent-side timestamp. The dashboard uses `receivedAt` for "data freshness" and the `startUtc`/`endUtc` for "activity time."

### L5 — MongoDB on a single VPS with no replica set
**File:** `docs/ARCHITECTURE.md` §8, `docs/DECISIONS.md` #21
**Problem:** M13 in glm-5.2 covers backups. Adjacent: a single-node MongoDB is not a replica set, which means MongoDB transactions (recommended in H1 of glm-5.2 for atomic ingest) don't work. Either (a) require a replica set (1 primary + 1 secondary, even on a single VPS using `rs.initiate()`), or (b) redesign the atomicity around `batches` collection upserts without transactions.
**Recommendation:** Add to Phase 0 console checklist: "MongoDB initialized as a single-node replica set (`rs.initiate()`) to enable transactions; backups configured per M13 (glm-5.2)."

### L6 — `os` field on heartbeat is missing (cross-reference mimo L7)
**File:** `docs/API-CONTRACT.md` §3
**Problem:** Mimo L7 already flagged this; restating as it matters: the server has no per-device OS-version view, which is critical when a Chrome update or Windows 11 24H2 rollout breaks UIA on a subset of machines. Without `os` in the heartbeat, debugging is "log on to the affected machine."
**Recommendation:** Add `os: "Windows 11 Pro 23H2"` to the heartbeat body, sent only when changed. Trivial change, real diagnostic value.

### L7 — MongoDB unique index strategy for `batchId` is undefined
**File:** `docs/ARCHITECTURE.md` §7, `atlas-console/.agents/skills/atlas-api-contract/SKILL.md` §3
**Problem:** C2/C4 in deepseek/mimo argue for `deviceId` in the `batchId`; H2 in glm-5.2 says a `batches` collection; mimo H4 says response should echo `batchId`. The console skill says "unique index involving `batchId`" without specifying the field list. With `deviceId` in the `batchId` (deepseek C2, mimo C4), a single `{batchId: 1}` unique index is sufficient. Without it, the index must be compound `{deviceId: 1, batchId: 1}` to avoid the cross-device collision the other reviews warn about.
**Recommendation:** Pick one and document in `docs/ARCHITECTURE.md` §7: either include `deviceId` in `batchId` (recommended) **and** use a single unique index, or use a compound index. Do not leave this open.

### L8 — Agent's `dotnet-best-practices` skill includes irrelevant sections (Semantic Kernel, ResourceManager, etc.)
**File:** `atlas-agent/.agents/skills/dotnet-best-practices/SKILL.md`
**Problem:** Mimo L6 already noted this. Restating because it's a real footgun: an AI agent loading the skill might add `Microsoft.Extensions.Localization` or `Microsoft.SemanticKernel` packages for no reason. Add an override at the top of the project-specific `atlas-agent-conventions` skill: "When `dotnet-best-practices` recommends Semantic Kernel, ResourceManager, or AI features, **ignore** — this project has none. Also ignore any guidance on trimming AOT, single-file publish edge cases, or Blazor — this is a Windows-service console-only agent."

### L9 — Phase 7 "Hardening & edge cases" is too broad; risk of "Phase 7 drag"
**File:** `docs/PLAN.md` Phase 7
**Problem:** Phase 7 bundles: multi-monitor, fast user switching, RDP, sleep/resume, clock changes, buffer cap edge, browser variants, PWAs/Electron, token revocation, org key rotation, AV review, 100-device load test. That's 11+ substantial work items, each of which is its own design + implementation + verification cycle. In practice, Phase 7 becomes a tar pit that never finishes and the project ships with most of it unaddressed.
**Recommendation:** Distribute Phase 7's items into the phases where they're naturally owned. Sleep/clock (M5 here, glm-5.2 M3) → Phase 1. Buffer cap + revocation + org key rotation → Phase 2. Browser variants + PWAs → Phase 3. Multi-monitor + RDP + fast user switch → Phase 1 (helper design). AV review → Phase 5 (when binary is finalized). Load test → Phase 6. Phase 7 then becomes a short checklist of items that genuinely couldn't be moved.

### L10 — `intervals.employeeId` reassignment has no bulk endpoint (see H8)
**File:** `docs/ARCHITECTURE.md` §7
**Problem:** Once `employeeId` is denormalized (H8), admins need a way to reattribute historical intervals. Currently there's no endpoint for it.
**Recommendation:** Add `POST /api/v1/admin/intervals/reassign` (`{ fromEmployeeId, toEmployeeId, deviceId, fromUtc, toUtc, reason }`) that updates matching intervals and writes an `auditLog` entry. Phase 4 admin work.

---

## Nice-to-Have

### N1 — `processName` could carry integrity level for elevated-app detection
For the foreground window, include the process's integrity level (`low`/`medium`/`high`/`system`) in the interval metadata. The dashboard can then distinguish "user actively chose to elevate" from "Consent.exe popped up." Pairs with H3.

### N2 — `category` rules could be scoped to time windows
Currently a `youtube.com` rule is binary. A future enhancement: time-of-day rules (e.g., "youtube.com is `productive` during 9-11 AM, `unproductive` after"). Not v1, but a natural extension that doesn't require a schema change.

### N3 — Agent should register with Windows Event Log for IT visibility
Restating mimo N6: the MSI custom action should write to the Application log, and the service should write startup/shutdown/fatal-error events. Free for IT, zero user impact.

### N4 — `category` resolution should be cacheable at ingest for hot domains
The current "resolve at ingest using admin rules" approach re-evaluates the rules for every interval. For 100 agents × 300 intervals/day = 30k rule lookups/day. A small per-device cache (`Dictionary<domain, category>` with invalidation on rule change) reduces it to <100 lookups/day/device. Not material at 100 devices, but a free win.

### N5 — `GET /api/v1/devices` should support ETag for cheap polling on the admin dashboard
A 100-device list is small but the dashboard re-fetches it. A simple ETag on the response eliminates redundant work and makes it obvious when something changed.

### N6 — Manifest `changelogUrl` field (mimo N2, restated)
Adding a `changelogUrl` to the manifest lets IT review what changed before approving an update. Cheap; pairs with M10's config-version concept.

### N7 — Helper should write its own "helper-down" event when it can't reach the service pipe
If the named pipe is unreachable, the helper should write a Windows Event Log entry (one-time, not every retry) so IT can diagnose "service is up but helper can't talk to it." Pairs with M2 (glm-5.2) on capture health.

---

## Where the Design Is Solid

These are well-reasoned and need no change. (Many overlap with the other three reviews; calling them out so future agents don't relitigate.)

- **Contract-first, doc-first discipline** (`AGENTS.md` §5, `API-CONTRACT.md`): one authoritative wire format, both repos mirror it. Exactly right for two repos built by different AI agents. The version-bump-first workflow is the right defense against silent drift.
- **Phased plan with explicit exit gates + manual verification + review checkpoint** (`PLAN.md`): correct process for AI-built code. Phase 0 as a thin vertical slice (enroll → heartbeat → dashboard online) is the right first cut; it proves the whole pipe end-to-end before any real capture logic exists.
- **Service + Helper split** (`ARCHITECTURE.md` §2): the standard, correct Windows pattern. Service in Session 0 for persistence/secrets/network; helper in user session for observation. No secrets, no networking in the helper. Clean.
- **Interval aggregation (Model B)** (`DECISIONS.md` #5): ~100–300 rows/day per device vs ~17k raw samples. The right tradeoff.
- **Three-state activity model** (`DECISIONS.md` #7, `ARCHITECTURE.md` §4): the genuinely new idea in this design. Separating "watching media" from "away" is more informative than binary idle, and idle-splitting keeps active time honest. (My H1 above is a *fix* to the WASAPI fallback, not a critique of the model.)
- **ACK-before-purge + `batchId` idempotency + exponential backoff** (`DECISIONS.md` #9): the correct reliability primitive. The other reviews' recommendations on atomicity (glm-5.2 H1) and `batches` collection (glm-5.2 H2) are hardening, not redesign.
- **Per-device DPAPI tokens, hashed server-side, revocable individually** (`DECISIONS.md` #15, `ARCHITECTURE.md` §6): correct for the threat model. One machine can be revoked without re-keying the fleet.
- **Opaque DeviceId GUID with mutable attributes** (`DECISIONS.md` #13, #14): the classic rename-forks-history bug is sidestepped. Reformat = new device is honest and simple.
- **Self-contained single-file .NET 8 publish** (`DECISIONS.md` #19): correct for "target PCs need nothing pre-installed." No runtime install on 100 machines is the right operational posture.
- **SQLite local buffer** (`DECISIONS.md` #8, #12): embedded, crash-safe, invisible. MongoDB-on-every-desktop would be a wrong answer; Mongo belongs on the server.
- **URL granularity = domain + page title, not full path** (`DECISIONS.md` #2): the right privacy/legality/noise balance. (M2 above is a length cap, not a critique of the choice.)
- **Test policy: unit-test pure logic, manually verify OS-interop** (`DECISIONS.md` #26, agent skill §7): pragmatic. UIA/GSMTC reads are inherently integration-level; mocking them produces tests that pass CI but fail on real hardware.
- **Console as single Next.js app for API + UI** (`DECISIONS.md` #11): at ~0.3 req/s, splitting ingest and dashboard adds complexity for zero benefit.
- **Cloudflare R2 for updates** (`DECISIONS.md` #20): zero egress, CDN-fast, decoupled from the VPS. The 80 MB × 100 agents math makes the cost saving real.
- **Decision log + single-source-of-truth `AGENTS.md`** (`DECISIONS.md` #23, #24): strong project hygiene for an AI-agent-built system.
- **Threat model of "standard users only"** (`DECISIONS.md` #17): realistic and well-stated. The plan correctly doesn't try to beat a local admin. The heartbeat is the right backstop.

---

## Summary of Recommendations by Phase

| Phase | New items to resolve before starting |
|-------|---------------------------------------|
| **Phase 0** | H7 (dashboard auth), M4 (`settings` collection schema), M9 (test-data seed), L3 (Next.js middleware auth), L5 (MongoDB single-node replica set) |
| **Phase 1** | H1 (WASAPI false-positive fix), H3 (foreground hijack filter), H4 (idle hysteresis), M1 (per-HWND flush), M3 (FileDescription mapping), M5 (sleep-close hook), M7 (`sessionId` on intervals), M12 (DPI manifest), M8 (privacy/logging policy doc) |
| **Phase 2** | H5 (device `status` state machine), H6 (rate limits), H8 (`employeeId` denormalized on intervals), M2 (pageTitle length cap), M6 (MSE audio supplement), L1 (timezone rendering approach), L4 (server-set `receivedAt`), L7 (`batchId` unique-index spec) |
| **Phase 3** | H2 (URL normalization / eTLD+1), M12 (DPI verification on 4K) |
| **Phase 4** | M14 (timezone aggregation), M16 (pre-aggregation plan), L10 (interval reassign endpoint) |
| **Phase 5** | M11 (AppLocker/WDAC allow-list verification) |
| **Phase 6** | M10 (config schema version + migration), M13 (org key `revokedAt`), L2 (MongoDB time-series collection), L6 (`os` in heartbeat) |
| **Phase 7** | L9 (redistribute Phase 7 items into earlier phases) |

---

*End of review. No code or existing documents were modified; only this file was created.*
