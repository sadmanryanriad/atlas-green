# Review: Atlas Green Pre-Implementation Design Review

**Reviewer:** qwen3.7-max
**Date:** 2026-06-23

## Executive Summary

The Atlas Green design is unusually well-structured for a monitoring system of this scope — the Service+Helper split, ACK-before-purge sync, and 3-state activity model are all sound. The phased plan with explicit exit gates is disciplined. However, several implementation risks are underspecified: UI Automation for cross-browser URL capture is the single largest technical gamble and needs a concrete fallback strategy; the helper launch mechanism from Session 0 needs explicit handling for pre-login and multi-session scenarios; and the security model has gaps around admin dashboard authentication and org-key rotation. Most issues below are addressable within the existing architecture — they need specification, not redesign.

---

## Critical

### C1. UI Automation for browser URL capture is underspecified and high-risk
**File:** `docs/ARCHITECTURE.md` §3, `docs/PLAN.md` Phase 3, `AGENTS.md` §3

UI Automation does **not** work uniformly across browsers:
- **Firefox** uses its own accessibility tree (IAccessible2/AT-SPI), not the standard UIA pattern. The address bar (`Awesome Bar`) is not reliably accessible via `System.Windows.Automation`. You will likely need a separate code path using Firefox's `IAccessible` COM interface or fall back to window-title parsing only.
- **Chrome/Edge/Brave** (Chromium) expose the omnibox via UIA, but the accessibility tree structure changes between major versions. The `AutomationId` for the address bar is not stable.
- **Incognito/Private windows** may have different window class names and different UIA tree structures.
- **Hardened browser configs** (e.g., `--force-renderer-accessibility` disabled) can suppress the UIA tree entirely.

**Recommendation:** Before Phase 3, build a spike that reads the address bar from Chrome, Edge, Firefox, and Brave (normal + incognito) via UIA. Document which `AutomationId`/`ClassName`/`ControlType` path works for each. For Firefox, plan a fallback to window-title parsing (`domain` from title pattern or `nsIURI` if you accept the complexity). Define a "URL unavailable" state in the interval schema so the dashboard can show "Firefox — URL unknown" rather than silently dropping data. Add a `urlCaptureMethod` field (`uia`, `title-fallback`, `unavailable`) to intervals for debugging.

### C2. Helper process launch from Session 0 is non-trivial and underspecified
**File:** `docs/ARCHITECTURE.md` §2, `docs/PLAN.md` Phase 1

The service runs in Session 0. To launch the helper into the active user's desktop, you need `CreateProcessAsUser` with the user's session token, obtained via `WTSQueryUserToken`. This has several failure modes:
- **No user logged in** (boot, or user logged out): no session token available. The helper cannot run. The plan says "restart on user login" but doesn't specify how the service detects login events (WTS session notification via `WM_WTSSESSION_CHANGE` or `WTSRegisterSessionNotification`).
- **UAC / elevated processes**: If the user's shell is elevated, `CreateProcessAsUser` with the linked token may behave unexpectedly.
- **Fast user switching**: Multiple sessions exist simultaneously. Which session gets the helper? The docs say "active desktop" but don't define how to pick when multiple sessions are connected.

**Recommendation:** Specify in the agent conventions: (1) Use `WTSRegisterSessionNotification` to detect logon/logoff/connect/disconnect events. (2) Launch the helper only when a console session (not RDP, or handle RDP separately in Phase 7) has an active user token. (3) Define behavior for multi-session: monitor the console session only in v1, note RDP as a Phase 7 item. (4) When no user is logged in, the service should still heartbeat (it can, from Session 0) but produce no activity intervals.

### C3. No admin authentication for the dashboard
**File:** `docs/ARCHITECTURE.md` §8, `docs/PLAN.md` Phase 4

The console serves a management dashboard with employee activity data, but no admin authentication mechanism is specified anywhere. The API contract covers agent→server auth (org key, device token) but says nothing about who can access the dashboard or the admin-facing API routes. This is a significant gap — employee activity data is sensitive.

**Recommendation:** Add to the plan (Phase 4 at latest): admin auth via NextAuth.js or a simple JWT-based login. At minimum, one admin account with password auth. Add to `docs/ARCHITECTURE.md` §6 (security model). Consider whether the dashboard needs role-based access (viewer vs. admin who can manage category rules and device mappings).

---

## High

### H1. GSMTC and WASAPI media detection needs Windows version floor
**File:** `docs/ARCHITECTURE.md` §4, `AGENTS.md` §3

`GlobalSystemMediaTransportControlsSessionManager` requires **Windows 10 build 17763 (1809)** or later via the WinRT `Windows.Media.Control` namespace. In .NET 8, this requires the `Microsoft.Windows.CsWinRT` package and a `WindowsSdkPackage` reference. If any target machines run older Windows 10 builds (common in Bangladesh offices), GSMTC will not be available.

WASAPI loopback capture as a fallback is also underspecified: detecting "audio is playing" via loopback requires distinguishing media audio from system notification sounds (which are short bursts). A simple amplitude threshold will false-positive on every Windows notification ding.

**Recommendation:** (1) Define a minimum Windows version (10 1809+) in the docs and check it at install time. (2) For WASAPI fallback, specify a minimum audio duration threshold (e.g., continuous audio output for >10 seconds) to filter out notification sounds. (3) If GSMTC is unavailable, fall back to WASAPI-only with the duration filter. (4) Add the required NuGet packages (`Microsoft.Windows.CsWinRT`, Windows SDK package) to the agent conventions skill.

### H2. Buffer encryption method not specified
**File:** `docs/ARCHITECTURE.md` §3, `docs/DECISIONS.md` #8

The docs say "SQLite, encrypted" but don't specify the encryption mechanism. Options:
- **SQLCipher** (via `SQLitePCLRaw` with SQLCipher provider): transparent page-level encryption, well-tested, but adds ~2MB to the binary and requires a separate native library.
- **Application-level encryption**: encrypt interval JSON before inserting into a plain SQLite DB. Simpler but the DB schema/metadata is still visible.
- **File-level encryption** (EFS or DPAPI on the folder): simplest but less granular.

**Recommendation:** Specify the encryption approach. For a silent agent defending against standard users, application-level encryption (encrypt the interval payload with a key derived from DPAPI) before SQLite insert is sufficient and avoids the SQLCipher native dependency. Document this in the architecture or agent conventions.

### H3. `batchId` collision risk
**File:** `docs/API-CONTRACT.md` §2

The example `batchId` format is `b-2026-06-22T09:10:00Z-001` — timestamp-based with a sequence suffix. But there's no specification for:
- How the sequence counter works (per-day? per-session? global?).
- What happens if the agent restarts and resets the counter.
- Whether the server enforces uniqueness strictly or "unique-ish."

**Recommendation:** Specify `batchId` as `{deviceId-short}-{unixTimestampMs}-{monotonicCounter}` or simply a UUID v4. The server must enforce a **unique index** on `batchId` in the `intervals` collection (not "unique-ish" as the architecture doc hedges). A UUID is simpler and collision-free. Update `API-CONTRACT.md` §2 with the exact format spec.

### H4. Graceful shutdown behavior not specified
**File:** `docs/ARCHITECTURE.md` §3, `docs/PLAN.md`

When the service stops (reboot, update, SCM restart), the current open interval in memory is lost — it hasn't been flushed to SQLite yet. Over many restarts, this could lose significant activity time.

**Recommendation:** Specify that on `IHostApplicationLifetime.ApplicationStopping`, the agent must: (1) close and flush the current open interval to SQLite, (2) attempt one final sync if time permits (with a short timeout), (3) stop the helper. Add this to the agent conventions skill and Phase 1 checklist.

### H5. Org key rotation is undefined
**File:** `docs/API-CONTRACT.md` §5, `docs/DECISIONS.md` #15

The docs say the org key is "rotatable" but don't specify the rotation flow. The org key is embedded in the MSI on each of ~100 machines. Rotating it means:
- Existing enrolled devices still have the old key baked in (but they don't need it — they use device tokens).
- New installs need the new key.
- A device that lost its token and needs to re-enroll needs the current key.

**Recommendation:** Document the rotation flow: (1) Server accepts both old and new org keys during a transition window. (2) New MSI builds embed the new key. (3) Existing devices are unaffected (they use device tokens). (4) Add a `settings` collection field for valid org keys (array, not single value). Add this to Phase 7 or document it as a runbook.

### H6. MongoDB `batchId` index must be strictly unique
**File:** `docs/ARCHITECTURE.md` §7

The architecture says `{ batchId }` is "unique-ish for idempotent ingest." This must be a **strict unique index**. If two batches somehow share a `batchId`, the second insert must fail (caught and returned as `duplicate: true`). A non-unique index defeats the entire idempotency guarantee.

**Recommendation:** Change to `{ batchId: 1 }, { unique: true }`. The idempotency check should be: attempt insert; on duplicate key error, return `duplicate: true` with the original `accepted` count (which requires storing the count with the batch).

---

## Medium

### M1. No minimum interval duration
**File:** `docs/ARCHITECTURE.md` §3, `docs/PLAN.md` Phase 1

The aggregator flushes on every app/domain/state change. Rapid Alt-Tab switching could produce dozens of 1–3 second intervals, creating noise in the dashboard and inflating row counts.

**Recommendation:** Define a minimum interval duration (e.g., 5 seconds). Intervals shorter than this are merged into the adjacent interval or discarded. Document in the aggregator spec.

### M2. Heartbeat `config` response is inconsistent with enroll `config`
**File:** `docs/API-CONTRACT.md` §1 vs §3

The `/enroll` response includes `bufferCapDays` in the config object. The `/heartbeat` response config does not. If the admin changes `bufferCapDays` remotely, it can only be delivered via re-enrollment.

**Recommendation:** Make the heartbeat `config` response include all the same fields as the enroll config, or explicitly document which fields are enroll-only vs remotely-tunable. Consistency is simpler.

### M3. RDP and remote sessions need explicit handling
**File:** `docs/PLAN.md` Phase 7

Listed as a Phase 7 item, but RDP is common in office environments (remote IT support, WFH). When a user RDPs in:
- A new session is created; the console session may be disconnected.
- The helper in the console session stops seeing input.
- If the helper is in the RDP session, it sees the remote user's activity.

**Recommendation:** Move basic RDP awareness to Phase 1 (the helper must at least not crash or produce garbage when the session changes). Full RDP monitoring can stay in Phase 7. Specify: in v1, monitor the console session only; if disconnected, treat as idle.

### M4. `machineHints.mac` assumes single NIC
**File:** `docs/API-CONTRACT.md` §1

Most desktops have one NIC, but some have Wi-Fi + Ethernet, or VPN adapters. A single `mac` field may not be the "primary" MAC.

**Recommendation:** Change to `macs: string[]` or document that it should be the MAC of the default route's adapter. Low priority since v1 has no hardware dedup.

### M5. No server-side health/monitoring for the console itself
**File:** `docs/ARCHITECTURE.md` §8

The system monitors 100 desktops but nothing monitors the console. If MongoDB goes down or the Next.js app crashes, all agents will buffer and retry, but no one is alerted.

**Recommendation:** Add a simple health endpoint (`GET /api/v1/health`) that checks MongoDB connectivity. Use an external uptime monitor (UptimeRobot, Better Stack) to alert on failures. Low effort, high value.

### M6. Clock skew handling is incomplete
**File:** `docs/API-CONTRACT.md` §5

The contract says the agent "reads `serverTimeUtc` to detect/sync drift" but doesn't specify what it does with that information. It also says it "never rewrites already-buffered timestamps." If a machine's clock is 30 minutes fast, all its intervals will have timestamps 30 minutes ahead of reality, and the dashboard will show incorrect timelines.

**Recommendation:** Specify: (1) The agent computes `clockOffset = serverTimeUtc - localUtc` at enroll and each heartbeat. (2) It applies this offset to **new** interval timestamps (not buffered ones). (3) If the offset exceeds a threshold (e.g., 5 minutes), it logs a warning. (4) The dashboard could show a "clock skew detected" flag for devices with large offsets.

### M7. Self-signed certificate trust deployment is underspecified
**File:** `docs/DECISIONS.md` #18, `docs/PLAN.md` Phase 6

"Push cert to company machines" is mentioned but not specified. For 100 machines, this means either: (a) a Group Policy pushing the cert to Trusted Root, (b) manual install on each machine, or (c) the MSI installs it. Each has different complexity.

**Recommendation:** Specify the trust distribution method. The MSI installer is the natural vehicle — it already runs with admin privileges and can install the cert to `LocalMachine\Root` during install. Document this in Phase 6 and the IT runbook.

### M8. Category re-resolution on rule change is underspecified
**File:** `docs/ARCHITECTURE.md` §7, `docs/PLAN.md` Phase 4

"Re-resolvable in bulk when rules change" is mentioned but the mechanism is not. With indefinite retention, this could mean re-categorizing millions of intervals.

**Recommendation:** Specify: (1) A background job (triggered from the dashboard) that iterates intervals matching the changed rule and updates the `category` field. (2) Use a MongoDB bulk write with a filter on `domain` or `processName`. (3) Log the re-resolution count. (4) Consider whether re-resolution should be async (queued) for large datasets.

### M9. No specification for what constitutes "online" vs "offline" threshold
**File:** `docs/API-CONTRACT.md` §3

"Devices not seen within ~3× the heartbeat interval" is mentioned but the default heartbeat interval is 300s, so the offline threshold is ~15 minutes. This should be explicit and configurable.

**Recommendation:** Define the offline threshold explicitly: `heartbeatIntervalSec * 3` (default 15 minutes). Consider exposing it in the dashboard settings.

---

## Low

### L1. Agent log rotation limits not specified
**File:** `docs/PLAN.md` Phase 0

"Structured logging to a rotating local file" is mentioned but no size limit, retention period, or max file count is defined.

**Recommendation:** Specify: max 10MB per file, keep last 5 files (~50MB total). Standard Serilog `RollingFile` defaults are fine.

### L2. No `User-Agent` or agent identification header
**File:** `docs/API-CONTRACT.md`

The agent sends `agentVersion` in request bodies but no `User-Agent` header. This makes server-side log analysis and debugging harder.

**Recommendation:** Add `User-Agent: AtlasAgent/{version}` to all agent HTTP requests. Standard practice, trivial to implement.

### L3. The `health` field in heartbeat has no alerting thresholds
**File:** `docs/API-CONTRACT.md` §3

CPU and memory are reported but no thresholds define "unhealthy." This data is collected but not actionable without thresholds.

**Recommendation:** Defer alerting to post-v1, but document that the dashboard should flag agents reporting >5% CPU or >200MB RAM as anomalous. Add to Phase 4 dashboard as a subtle indicator.

### L4. No CSV/PDF export mentioned in v1 scope
**File:** `docs/ARCHITECTURE.md` §8

Listed as "deferred" but management in Bangladesh offices often expects spreadsheet exports for HR reviews. This may become a priority request soon after launch.

**Recommendation:** Note it as "likely v1.1" rather than indefinite deferral. No action needed now.

### L5. `pageTitle` capture from window title may include noise
**File:** `docs/API-CONTRACT.md` §2

Window titles often include the site name suffix ("— YouTube", "- Google Chrome"). The contract doesn't specify whether `pageTitle` should be cleaned or stored raw.

**Recommendation:** Store raw, clean in the dashboard or at query time. Document this choice.

---

## Nice-to-have

### N1. Consider a contract test suite
A shared test suite (e.g., a Postman collection or a set of curl scripts) that validates both the agent and the console against the contract would catch drift early. Could live in the mother repo.

### N2. Agent self-diagnostics endpoint
A local-only HTTP endpoint (e.g., `http://localhost:PORT/diag`) that returns the agent's current state (last interval, buffer size, sync status) would help IT debug issues without reading log files. Must be localhost-only and not create any UI.

### N3. Dashboard dark mode
Minor but appreciated for admin users reviewing timelines for extended periods.

---

## What's solid

- **Service+Helper architecture**: Correct Windows pattern. Session 0 isolation for persistence, user session for observation. The helper holding no secrets is a clean separation.
- **ACK-before-purge with batchId idempotency**: This is the right sync model. No data loss on retry, no duplicates on redelivery. Well-specified.
- **3-state activity model**: The active/passive_media/idle split is genuinely useful and more informative than binary active/idle. GSMTC as primary with WASAPI fallback is the right priority.
- **Identity model**: Opaque GUID with mutable attributes is clean. Reformatted machine = new device is pragmatic for v1.
- **Threat model honesty**: "We defend against standard users only" with heartbeat as backstop is realistic and avoids over-engineering.
- **Phased plan with exit gates**: Disciplined. Phase 0 as a thin vertical slice (dummy heartbeat end-to-end) is the right first step.
- **DPAPI for token storage**: Correct Windows-native choice. Machine scope means the token survives reboots without a user logged in.
- **UTC storage + local display**: Correct for a multi-timezone team.
- **MongoDB at this scale**: At ~0.3 req/s and ~300 intervals/day/device, MongoDB is more than adequate. The collection design and indexes are sound.
- **Self-contained single-file publish**: Right call for zero-dependency deployment to 100 machines.

---

## Summary by severity

| Severity | Count | Key themes |
|----------|-------|------------|
| Critical | 3 | Browser UIA feasibility, helper launch mechanics, missing dashboard auth |
| High | 6 | Windows version floor, encryption method, batchId spec, shutdown behavior, org-key rotation, unique index |
| Medium | 9 | Min interval duration, config consistency, RDP, clock skew, cert distribution |
| Low | 5 | Log rotation, User-Agent header, health thresholds |
| Nice-to-have | 3 | Contract tests, local diagnostics, dark mode |

The criticals are addressable without architectural changes — they need specification and spike work, not redesign. The highest priority action is a UIA feasibility spike across all target browsers before committing to Phase 3's approach.
