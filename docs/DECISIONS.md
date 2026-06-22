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
| 7 | **Media = not idle**; 3-state model (`active`/`passive_media`/`idle`) via **GSMTC** (+ WASAPI fallback). | Watching video has no input but isn't "away." Separating "at desk watching media" from "away" is more useful (and more damning) than one idle bucket. |
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
| 18 | **Code signing = self-signed + internal trust** for v1 until CEO approves a purchased cert. | Free, and the company controls the machines' trust store; needed anyway for the update signature check. |
| 19 | **Runtime = self-contained single-file `.exe`.** | Target PCs need nothing installed. |
| 20 | **Update hosting = Cloudflare R2** + manifest JSON. | Zero egress (80 MB × 100 ≈ 8 GB/wave), CDN-fast for remote staff, decoupled from the VPS, binary stays private. |
| 21 | **Server retention = indefinite**, with **manual per-PC archive/delete** by admin. | Management wants long history; admin keeps control of individual records. |
| 22 | **Time = store UTC, display local.** | Remote employees/admins span multiple time zones. |
| 23 | **Repos = three**: atlas-green (mother/docs), atlas-agent, atlas-console. Mother gitignores child folders. | Different stacks, deploy targets, and release cadences; focused repos keep AI agents sharp; the contract gets one authoritative home. |
| 24 | **Context = AGENTS.md canonical**, `CLAUDE.md` = `@AGENTS.md`. | One source of truth, usable by Claude Code and Opencode without drift. |
| 25 | **Skills**: `next-best-practices` (console), `dotnet-best-practices` (agent), + custom `atlas-api-contract` and `atlas-agent-conventions`. "Consult before coding" enforced via AGENTS.md. | Keeps AI-built code consistent; few skills to avoid confusion. |
| 26 | **Testing**: unit-test pure logic (aggregation, state machine, categorization, sync/retry); verify OS-interop manually. xUnit / Vitest+Playwright; tests gate phases in CI. | Tests are the safety net against AI regressions; unit-testing real window/UIA reads isn't worth it. |
| 27 | **Workflow**: phased plan + checklists + manual test gate + Claude review after each phase. Claude codes Phase 0; Opencode codes Phases 1+. | Human-conducted AI team; keeps quality control tight. |

## Open / deferred questions

- Shared single Windows login attribution (rare here) — discuss with CEO later.
- Purchased code-signing cert — pending CEO approval.
- Employee consent / acceptable-use policy — recommended for legal defensibility
  (company decision, not an engineering blocker).
- Alerts, departments/org chart, CSV/PDF export — post-v1 dashboard features.
