# Atlas Green — Decision Log

Every locked decision, with the reasoning, so future agents (and future us) don't
relitigate settled choices. To change one, discuss with the human, then edit here.

| # | Decision | Why |
|---|----------|-----|
| 1 | **Scope v1** = activity metadata + URL (domain + page title) + 3-state model. Screenshots = v2. | Real visibility without the storage/legal weight of screenshots up front. |
| 2 | **URL granularity = domain + page title** (not full path). | Very reliable via window title + UIA, low noise; answers "working vs slacking" without messy partial paths. |
| 3 | **Agent = C# / .NET 8**, self-contained single-file. | Best Windows citizen: native Win32, **UI Automation**, Windows Service support; MS-signed runtime → less AV friction; zero runtime install on targets. |
| 4 | **Two-part agent**: Windows Service + user-session helper. | A Session-0 service is persistent/protected but can't see the desktop; the helper (in the user session) does the observing. Standard pattern. |
| 5 | **Sampling = interval aggregation (Model B)**, 3–5 s in-memory, flush on app/domain/state change. | ~100–300 rows/day vs ~17k; "time spent" becomes a trivial query; tiny payloads. |
| 6 | **Idle = 5 min** no input; idle **splits** the interval. | Below feels twitchy; above over-credits idle as work. Splitting keeps "active time" honest. |
| 7 | **Media = not idle**; 3-state model (`active`/`passive_media`/`idle`) via **GSMTC** (+ gated WASAPI fallback). WASAPI audio only counts as `passive_media` when the active/most-recent GSMTC source is a **non-communications** app; **calls (Teams/Zoom/Meet/Discord/Slack huddle) are never `passive_media`**. | Watching video has no input but isn't "away." Separating "at desk watching media" from "away" is more useful (and more damning) than one idle bucket. Raw audio output fires on calls, notification dings, and screen-reader TTS — counting a 3-hour client call as "passive media" would be wrong and damaging for a system whose pitch is honesty. (Review 2026-06.) |
| 8 | **Local buffer = SQLite**, encrypted, ACL-locked in `C:\ProgramData\...`. | Embedded, crash-safe, zero process/footprint (unlike running MongoDB on every desktop). The buffer is just an outbox queue. |
| 9 | **Sync = 5-min gzipped batch**, exp. backoff, **ACK before purge**, `batchId` idempotency. | Near-real-time yet efficient/battery-friendly; no data loss across failures/retries. |
| 10 | **Buffer cap ~2–3 months**. | Without screenshots, ~150 KB–1 MB/day → tens of MB over months. Cap is just a safety net for long-offline machines. |
| 11 | **Server DB = MongoDB**; **console = Next.js** (UI + API routes) + Cloudflare on a VPS. | Matches the human's expertise; load at 100 agents (~0.3 req/s) is trivial; Cloudflare for TLS/WAF/DDoS. |
| 12 | **Local buffer is NOT MongoDB.** | MongoDB is a server (`mongod.exe`) — heavy, visible, wrong for a silent agent. Mongo belongs on the server where queries/expertise matter. |
| 13 | **Identity = opaque DeviceId GUID** generated at enroll; pcName/hostname/user/mac are mutable attributes. | Renaming or user change never forks/merges identity or history. |
| 14 | **Reformatted machine = new device** (no hardware dedup in v1). | Simplest; admin just re-maps. Hardware-hint dedup can be added later if annoying. |
| 15 | **Enrollment**: MSI org key + `PCNAME` param → `/enroll` → **per-device token** via **DPAPI machine scope**. | Org key only enables enrollment (rotatable); per-device tokens allow single-machine revocation; DPAPI stops a curious user reading it. |
| 16 | **Deployment**: ~100 PCs, MSI installed manually by IT; PC name set at install, changeable from dashboard. | Matches the office; install-param + dashboard label keeps names flexible without touching identity. |
| 17 | **Lifecycle**: signed remote auto-update + SCM auto-restart + helper watchdog + heartbeat online/offline. Defend against **standard users only**. | Can't walk to 100 PCs per fix. Can't beat a local admin — heartbeat (silent agent = red flag) is the backstop. |
| 18 | **Code signing = self-signed + internal trust** for v1 until CEO approves a purchased cert. **IT pushes the self-signed cert to `LocalMachine\Root` via Group Policy on every target PC BEFORE the MSI rollout**, and **AV/SmartScreen testing moves to Phase 5** (not Phase 7). | Free, and the company controls the machines' trust store; needed anyway for the update signature check. Pre-trusting the cert is what keeps the install "silent" — an untrusted self-signed binary triggers SmartScreen/Defender prompts, and a non-admin clicking "Don't run" defeats the project. (Review 2026-06.) |
| 19 | **Runtime = self-contained single-file `.exe`.** | Target PCs need nothing installed. |
| 20 | **Update hosting = Cloudflare R2** + manifest JSON. | Zero egress (80 MB × 100 ≈ 8 GB/wave), CDN-fast for remote staff, decoupled from the VPS, binary stays private. |
| 21 | **Server retention = indefinite**, with **manual per-PC archive/delete** by admin. | Management wants long history; admin keeps control of individual records. |
| 22 | **Time = store UTC, display local.** | Remote employees/admins span multiple time zones. |
| 23 | **Repos = three**: atlas-green (mother/docs), atlas-agent, atlas-console. Mother gitignores child folders. | Different stacks, deploy targets, and release cadences; focused repos keep AI agents sharp; the contract gets one authoritative home. |
| 24 | **Context = AGENTS.md canonical**, `CLAUDE.md` = `@AGENTS.md`. | One source of truth, usable by Claude Code and Opencode without drift. |
| 25 | **Skills**: `next-best-practices` (console), `dotnet-best-practices` (agent), + custom `atlas-api-contract` and `atlas-agent-conventions`. "Consult before coding" enforced via AGENTS.md. | Keeps AI-built code consistent; few skills to avoid confusion. |
| 26 | **Testing**: unit-test pure logic (aggregation, state machine, categorization, sync/retry); verify OS-interop manually. xUnit / Vitest+Playwright; tests gate phases in CI. | Tests are the safety net against AI regressions; unit-testing real window/UIA reads isn't worth it. |
| 27 | **Workflow**: phased plan + checklists + manual test gate + Claude review after each phase. Claude codes Phase 0; Opencode codes Phases 1+. | Human-conducted AI team; keeps quality control tight. |

## Decisions added from the 2026-06 multi-model review

Six independent models (deepseek-v4-pro, glm-5.2, mimo-v2-5-pro, kimi-k2.7-code,
minimax-m3, qwen-3.7-max) reviewed the plan before Phase 0. Full findings live in
[`reviews/`](reviews/). No architectural flaw was found; these rows close
specification gaps the reviewers surfaced.

| # | Decision | Why |
|---|----------|-----|
| 28 | **`batchId` = client-generated UUID v4**; server dedupe via a **strict unique index `{deviceId, batchId}`** plus an independent **`batches` receipt collection**. | A UUID removes the timestamp/sequence-collision and restart-reuse risks of the old `b-<time>-<seq>` format; a receipt collection decoupled from `intervals` means an admin-deleted device can't be silently resurrected by a stale offline replay. (All 6 models.) |
| 29 | **`/ingest` is atomic per batch** (MongoDB transaction); **MongoDB runs as a single-node replica set** (`rs.initiate()`). | "ACK before purge" is only safe if a partial insert + retry can't lose or duplicate rows; transactions require a replica set even on one VPS. |
| 30 | **Buffer encryption = app-layer AES-GCM**, key stored in **DPAPI machine scope**; the ProgramData **ACL is the primary control**, DPAPI/encryption are defense-in-depth. | "Encrypted SQLite" needed a concrete mechanism; app-layer AES keeps the self-contained single-file publish free of native deps (no SQLCipher build). |
| 31 | **Dashboard admin auth = Cloudflare Access + a Next.js `middleware.ts`**, wired in **Phase 0**. | The dashboard exposes all employee monitoring data and had no auth anywhere in the plan; it must never be deployed open, and the app must reject direct-IP/bypass requests too. |
| 32 | **Helper launch**: the service runs as **LocalSystem**, launches the helper via **`WTSQueryUserToken` + `CreateProcessAsUser`**, and reacts to **`WTS_SESSION_CHANGE`**; v1 monitors the **console (physical) session only** — RDP sessions are ignored. | The single trickiest Windows-interop piece; if mis-specified the service heartbeats "online" while nothing is captured. |
| 33 | **Interval durations come from a monotonic clock** (`Stopwatch`), UTC stamped from the wall clock; the open interval is **closed on sleep/suspend** (`PowerModeChanged`); a **clock-skew offset** is stored on the device and skewed devices are flagged (buffered timestamps are never rewritten). | Wall-clock jumps and sleep/resume otherwise produce phantom multi-hour intervals and wrong-day data. |
| 34 | **Idle has hysteresis**: enter `idle` at 300 s no-input, **leave only after ~30 s sustained input**; sub-~5 s intervals are merged. | Crisp thresholds make the state machine oscillate every sample near the boundary, flushing constantly and defeating Model B aggregation. |
| 35 | **URL capture has a fallback chain** — UIA → window-title parse → `domain: null` — recorded via a `urlCaptureMethod` field; **domains normalized to eTLD+1** before buffering. | UIA is fragile (esp. Firefox/hardened configs); never silently drop data, and `www.`/`m.`/`youtu.be` variants must collapse to one domain or every report fragments. |
| 36 | **`pageTitle` capped at 256 chars, `windowTitle` at 512**, truncated at the agent. | Titles can carry sensitive strings (email subjects, doc/customer names); caps bound both storage and the privacy surface. Paired with #31 (auth) this is the agreed v1 privacy posture — titles are kept for their analytical value, not dropped. |
| 37 | **`employeeId` is denormalized onto every interval** at ingest (a snapshot resolved from the current device→employee mapping; never back-filled). | The hottest dashboard query is "employee X over date range Y"; this avoids a join on every read, mirroring why `osUsername` is already denormalized. |
| 38 | **Org keys = an array in `settings`** with created/expiry/revoke per key; a **blocked/revoked device gets `403` at `/enroll`**. | Enables org-key rotation with a transition window, and stops a revoked device from trivially re-enrolling itself. |
| 39 | **Device `status` is an explicit server-owned state machine**: `pending` → `online`/`offline`, plus `outdated`/`revoked`/`archived`. | One authoritative enum instead of three ad-hoc derived fields (`lastSeenAt`, `archived` bool, `revokedAt`) the dashboard has to compose inconsistently. |
| 40 | **Daily/weekly bucketing uses a fixed org timezone (Asia/Dhaka)**; timeline rendering still uses the **viewer's** local zone. | 99% of staff are in Bangladesh (UTC+6); per-employee timezone bucketing is over-engineering for v1. |
| 41 | **Heartbeat carries capture health** (`helperRunning`, `lastSampleUtc`, `helperRestartCount`) **and the server rate-limits** `/ingest`, `/heartbeat`, `/enroll`. | "Service online" must not hide a dead/killed helper; rate limits cap a misbehaving or rogue agent. |
| 42 | **Drop `mac` from enrollment; keep only `machineHints.machineGuid`** (informational, still no dedup in v1). | The MAC is multi-NIC-ambiguous and unused; `machineGuid` is the one cheap hint that could later detect a disk-clone. |

## Decisions explicitly NOT taken (reviewer suggestions we rejected for v1)

- **MongoDB time-series collection for `intervals`** — tempting for compression,
  but it restricts secondary indexes and complicates the atomic-ingest/transaction
  story (#29). Plain collection + good indexes now; revisit if scale demands.
- **Per-employee timezone day-bucketing** — see #40; deferred.

## Open / deferred questions

- Shared single Windows login attribution (rare here) — discuss with CEO later.
- **Purchased OV code-signing cert** — future upgrade from self-signed + GPO (#18);
  pending CEO approval. Would remove the GPO pre-trust step.
- Employee consent / acceptable-use policy — recommended for legal defensibility
  (company decision, not an engineering blocker); page titles are stored (#36).
- **Pre-aggregated `dailySummary` collection** — deferred until dashboard
  aggregation queries actually slow down; ingest path designed so it's a drop-in.
- **Admin audit log** (`auditLog` collection) — planned for Phase 4 (record every
  destructive/rules-changing admin action).
- Alerts, departments/org chart, CSV/PDF export — post-v1 dashboard features.
