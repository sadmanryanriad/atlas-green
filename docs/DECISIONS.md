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
| 7 | **Media = not idle**; 3-state model (`active`/`passive_media`/`idle`) via **GSMTC** (+ gated WASAPI fallback). WASAPI audio only counts as `passive_media` when the **recent (≤60 s)** active GSMTC source is a **non-communications** app; **calls (Teams/Zoom/Meet/Discord/Slack huddle) are never `passive_media`**, and **an active communications call overrides** even when a non-comms source (e.g. Spotify) plays concurrently (refined in r2). | Watching video has no input but isn't "away." Separating "at desk watching media" from "away" is more useful (and more damning) than one idle bucket. Raw audio output fires on calls, notification dings, and screen-reader TTS — counting a 3-hour client call as "passive media" would be wrong and damaging for a system whose pitch is honesty. (Review 2026-06.) |
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
| 18 | **Code signing = self-signed + internal trust** for v1 until CEO approves a purchased cert. **IT pre-trusts the self-signed cert to `LocalMachine\Root` BEFORE the MSI rollout** — via **Group Policy** on AD-joined machines, or (for **workgroup/non-domain** PCs, likely in a BD SME) a PowerShell `Import-Certificate … -CertStoreLocation Cert:\LocalMachine\Root` or an MSI custom action. **AV/SmartScreen testing moves to Phase 5** (not Phase 7). | Free, and the company controls the machines' trust store; needed anyway for the update signature check. Pre-trusting the cert is what keeps the install "silent" — an untrusted self-signed binary triggers SmartScreen/Defender prompts, and a non-admin clicking "Don't run" defeats the project. (Review 2026-06.) |
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
| 28 | **`batchId` = client-generated UUID v4**; dedup uniqueness lives **only on the `batches` receipt collection** (`{deviceId, batchId}` unique). On `intervals`, `{deviceId, batchId}` is a **non-unique** index (for find/delete-by-batch). `batches` has a **TTL** (~180 d, past the buffer cap) so it can't grow unbounded. | A UUID removes collision/restart-reuse; the receipt decouples dedup from interval lifecycle (no stale-replay resurrection). A *unique* index on `intervals` would reject every batch of >1 interval (they share a `batchId`) — uniqueness must sit on `batches` only. (Review r2: glm C1, qwen H1 — caught a real bug.) |
| 29 | **`/ingest` is atomic per batch** (MongoDB transaction, **≤1000 intervals/txn**, `maxCommitTimeMS`, retry on `TransientTransactionError`); **MongoDB runs as a single-node replica set** (`rs.initiate()` bound to `127.0.0.1`). | "ACK before purge" is only safe if a partial insert + retry can't lose or duplicate rows; transactions require a replica set even on one VPS. A single-node RS provides `{w:1, j:true}` durability with a narrow journal-commit crash window (acceptable v1; the unique `batches` index makes retries safe regardless). (Review r2: kimi H1, mimo H2.) |
| 30 | **Buffer encryption = app-layer AES-GCM**, key stored in **DPAPI machine scope**; the ProgramData **ACL is the primary control**, DPAPI/encryption are defense-in-depth. | "Encrypted SQLite" needed a concrete mechanism; app-layer AES keeps the self-contained single-file publish free of native deps (no SQLCipher build). |
| 31 | **Dashboard admin auth = Cloudflare Access + a Next.js `middleware.ts`**, wired in **Phase 0**. | The dashboard exposes all employee monitoring data and had no auth anywhere in the plan; it must never be deployed open, and the app must reject direct-IP/bypass requests too. |
| 32 | **Helper launch**: the service runs as **LocalSystem**, launches the helper via **`WTSQueryUserToken` + `CreateProcessAsUser`**, and reacts to **`WTS_SESSION_CHANGE`**; v1 monitors the **console (physical) session only** — RDP sessions are ignored. | The single trickiest Windows-interop piece; if mis-specified the service heartbeats "online" while nothing is captured. |
| 33 | **Durations from a monotonic clock** (`Stopwatch`); `durationSec` is the **canonical** duration (never derive it from `endUtc−startUtc`). The open interval is **closed on sleep/suspend via the service's `SERVICE_CONTROL_POWEREVENT` handler** (not `SystemEvents.PowerModeChanged`, which needs a message pump a Session-0 service lacks), with a **monotonic sanity fallback** (force-close an open interval whose elapsed time > idle-threshold × 3, catching missed/Modern-Standby events) and a **sub-30 s power-blip debounce**. On **resume the new interval's state is re-derived from the next sample** (not hardcoded `idle` — media may auto-resume). A **clock-skew offset** is stored per device and applied to **both** display and day-bucketing; buffered timestamps are never rewritten. | Wall-clock jumps, sleep/resume, BD power blips, and Modern Standby otherwise produce phantom intervals and wrong-day data; a service can't use the windowed power API. (Review r2: kimi C2, glm M1/M4, deepseek M5, mimo M4.) |
| 34 | **Idle entry is debounced, exit is immediate**: enter `idle` only after **300 s of continuous no-input** — **any input resets** the 300 s timer (this kills boundary oscillation); **leave `idle` on any input** → `active`. **Media transitions bypass hysteresis** (`idle`↔`passive_media` are immediate). Sub-~5 s intervals are merged. | "30 s sustained input to leave idle" would record an entire call as `idle` (participants type in short bursts), contradicting #7. Debouncing on *entry* (reset-on-input) fixes oscillation without ever suppressing real activity. (Review r2: glm H4, mimo C1, qwen M2, kimi H4.) |
| 35 | **URL capture has a fallback chain** — UIA → window-title parse → `domain: null` — recorded via a `urlCaptureMethod` field; **domains normalized to eTLD+1** before buffering. | UIA is fragile (esp. Firefox/hardened configs); never silently drop data, and `www.`/`m.`/`youtu.be` variants must collapse to one domain or every report fragments. |
| 36 | **`pageTitle` capped at 256 chars, `windowTitle` at 512**, truncated at the agent. | Titles can carry sensitive strings (email subjects, doc/customer names); caps bound both storage and the privacy surface. Paired with #31 (auth) this is the agreed v1 privacy posture — titles are kept for their analytical value, not dropped. |
| 37 | **`osUsername` and `employeeId` both denormalized onto every interval** at ingest. `osUsername` is a **capture-time snapshot, never back-filled**. `employeeId` is resolved from the current mapping at ingest but is **re-resolvable**: a device→employee remap updates future intervals and can **bulk-reassign a historical range** (audit-logged). | The hot query is per-employee-by-date. `osUsername` is an OS fact (immutable history); `employeeId` is an admin mapping that sometimes must be corrected for the past — different back-fill rules. (Review r2: glm H5, qwen H2.) |
| 38 | **Org keys = an array in `settings`** with created/expiry/revoke per key; a **blocked/revoked device gets `403` at `/enroll`** (org-key validated **first** → `401`, then block → `403`). | Enables org-key rotation with a transition window, and stops a revoked device from trivially re-enrolling itself. |
| 39 | **Device state = liveness + orthogonal flags** (server-owned). `liveness`: `pending` → `online`/`offline` (from `lastSeenAt` vs `heartbeatIntervalSec × 3`). Independent `flags`: `outdated`, `clock_skewed`, `revoked`, `archived`, `health_warning`. Dashboard reads liveness for the badge, renders flags as chips. | A flat enum couldn't show "online AND outdated AND clock-skewed" at once — IT couldn't tell if an outdated device was still phoning home. Liveness+flags keeps the one-authoritative-source intent while representing co-occurring conditions. (Review r2: minimax H4, glm M3.) |
| 40 | **Day/week bucketing uses a fixed org timezone (`settings.orgTimeZone`, Asia/Dhaka)**; timeline rendering uses the **viewer's** local zone. The agent's **device IANA timezone is captured** as a mutable attribute (so history can be re-bucketed later), and an optional **`employees.tzOverride`** buckets a non-Dhaka hire correctly. The clock-skew offset is applied **before** bucketing. Changing `orgTimeZone` is audit-logged and affects future bucketing only. | 99% of staff are in Bangladesh, so a fixed org tz is right — but capturing the device tz now (cheap) keeps the per-employee option open and the override handles the rare remote hire without full per-employee bucketing. (Review r2: glm H2, kimi C1, deepseek H5.) |
| 41 | **Heartbeat carries capture health** (`helperRunning`, `lastSampleUtc`, `helperRestartCount`) **and the server rate-limits** `/ingest`, `/heartbeat`, `/enroll`. | "Service online" must not hide a dead/killed helper; rate limits cap a misbehaving or rogue agent. |
| 42 | **Drop `mac` from enrollment; keep only `machineHints.machineGuid`** (informational, still no dedup in v1). The **service** (LocalSystem) reads it from `HKLM\…\Cryptography`; the standard-user helper cannot. `null` if unavailable. Also sent in the **heartbeat** so a disk-clone (one DeviceId, differing `machineGuid`) is detectable. | The MAC is multi-NIC-ambiguous and unused; `machineGuid` is the one cheap hint that could later detect a disk-clone. (Review r2: minimax M8, mimo H4, glm M8.) |

## Decisions added from the iteration-2 (v1.2) review

The same six models re-reviewed v1.1. They found a real bug (#28 above) and several
contradictions the v1.1 fixes introduced; #43–#48 are the new policies, and #7, #18,
#28, #29, #33, #34, #37, #38, #39, #40, #42 above were amended.

| # | Decision | Why |
|---|----------|-----|
| 43 | **Enrollment recovery rotates on token loss.** A re-enroll **presenting a valid token** is a no-op (already enrolled). A re-enroll **without** a valid token (DPAPI blob lost/corrupted) **rotates**: server marks the old `deviceTokens` row `supersededAt` and issues a new token. Blocked/revoked → `403`. | v1.1 said "re-enroll returns the existing token," but tokens are stored **hashed** (#15) — the server can't return one. Rotation is safe here because batches are keyed by `deviceId`+`batchId`, not token era, so buffered rows still sync. (Review r2: glm H1.) |
| 44 | **`/enroll` rate limit = 10/min/IP burst + a corp-CIDR allowlist** in `settings` (not 1/hour/IP). | All ~100 office PCs share one NAT public IP; 1/hour/IP would make the manual rollout take ~100 hours. Per-device throttling + a corporate-network allowlist still blocks a scripted rogue burst. (Review r2: glm H3, kimi H7.) |
| 45 | **Revoked device stays visible**: `/heartbeat` **accepts a revoked token** and returns `200` + `flags:[revoked]` (device stops capturing, keeps its buffer); `/ingest` and `/enroll` still reject it (`401`/`403`). A `reauthorized` flag in the heartbeat response lets an un-revoked device self-heal. | A revoked token returning `401` everywhere meant a revoked machine vanished (looked "offline") instead of showing as `revoked`. (Review r2: kimi C4, deepseek L4.) |
| 46 | **Named-pipe anti-spoofing**: the helper sends a **per-instance UUID** in a connect handshake (so the service dedupes samples only within one helper instance, not across restarts), and the service **verifies the pipe peer** (`GetNamedPipeClientProcessId` → image path → code signature) before accepting samples. | The helper runs as the interactive user, so a standard user could connect to the pipe and inject fake samples; and a naive global seq drops the first samples after every helper restart. (Review r2: kimi C3, deepseek H2.) |
| 47 | **`426 Upgrade Required` applies only to `/enroll` and `/ingest`** — **`/heartbeat` always returns `200`** with the `update` block (plus a `contractDeprecated` flag), and the agent has a **compiled-in fallback `manifestUrl`**. | If a major contract bump made `/heartbeat` 426 too, the agent could never fetch the manifest URL to self-update — bricked with no recovery path. (Review r2: glm M6.) |
| 48 | **"Forget device" admin action** atomically: revokes the token, sets `blocked`, deletes the device's `intervals` **and** its `batches` receipts. Outside that action, receipts follow the TTL (#28). | Deleting intervals but keeping receipts left a device→timestamp link (incomplete erasure) and a possible partial-replay path. (Review r2: kimi H2, mimo C2.) |

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
- **Admin audit log** (`auditLog` collection) — **infrastructure + write helper land
  in Phase 0** (Phase 0 already ships the device→employee mapping admin action, which
  must be audited from day one); the audit-log **viewer** is Phase 4. (Review r2:
  minimax H3.)
- **Multi-monitor media attribution** — a `passive_media` interval attributes media to
  the foreground app, so media from a non-foreground monitor shows the foreground
  app/domain. Documented limit for v1; an optional `mediaSource` field is a later
  option. (Review r2: glm M9.)
- **Contract-bump checklist** — when `API-CONTRACT.md` is bumped, a step must verify
  both child `AGENTS.md` files match (iteration-2 caught the console one still listing
  a deleted route). (Review r2: deepseek C1.)
- Alerts, departments/org chart, CSV/PDF export — post-v1 dashboard features.
