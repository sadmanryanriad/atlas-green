# Atlas Green — System Context (Mother Repo)

> This file is the **single source of truth** for the whole Atlas Green system.
> `CLAUDE.md` imports it. Opencode and other agents read this file directly.
> Each sub-repo (`atlas-agent`, `atlas-console`) has its own `AGENTS.md` with
> stack-specific detail — this file describes the **whole system** and the rules
> that apply everywhere.

---

## 1. What we are building

**Atlas Green** is an employee-activity monitoring system for **company-owned
Windows desktops**. The company is based in Bangladesh, devices are company
property, and monitoring is legally authorized. The goal is to give management an
honest picture of how work time is spent.

It has two deployed components plus this docs repo:

| Repo | Role | Stack |
|------|------|-------|
| **atlas-green** (this repo) | Master plan, architecture, API contract, cross-cutting context. No application code. | Markdown only |
| **atlas-agent** | Silent background agent installed on each Windows desktop. Captures activity, buffers locally, ships to the server. | C# / .NET 8 |
| **atlas-console** | Admin dashboard + ingest/enroll/heartbeat API. Stores and visualizes data. | Next.js + MongoDB |

The three repos live side-by-side inside a local `Atlas-Green/` workspace folder.
This mother repo **gitignores** the two child repo folders, so there is no nested
git conflict.

---

## 2. The single most important rule: the agent is SILENT

The agent must run with **zero user-visible footprint**:
- No window, no system tray icon, no taskbar entry, no notifications, no console.
- No popups, no sounds, no focus-stealing, ever.
- Minimal CPU/RAM; must not make the machine feel slow.
- It runs as a **Windows Service** (starts at boot, before/without login) plus a
  small **user-session helper** that the service launches into the active desktop
  to read the foreground window, URLs, and (in v2) screenshots.

If any task would create UI noise, that is a bug. The only "interface" is the
admin dashboard (atlas-console), which employees never see.

---

## 3. Capture scope

### v1.0.0 (what we build now)
- **Activity metadata**: foreground application, process name, window title,
  active vs idle, timestamps.
- **URL tracking**: **domain + page title** for every browser, every profile,
  including incognito (read from the address bar via UI Automation — this is the
  key advantage over a browser extension).
- **Three-state activity model** (not just active/idle):
  - **Active** — real keyboard/mouse input within the idle threshold.
  - **Passive (media)** — no input, but media is actively playing (YouTube,
    TikTok, movie, Spotify). Detected via Windows `GlobalSystemMediaTransport
    ControlsSessionManager` (GSMTC), with a **gated** audio-output (WASAPI)
    fallback — **calls (Teams/Zoom/Meet/etc.) are never counted as media**
    (see `docs/DECISIONS.md` #7).
  - **Idle / Away** — no input AND no media.

### v2.0.0 (planned, not now)
- **Screenshots**: randomized **10–15 min** interval, skip-on-idle, ~1280px WebP
  @ ~55% quality, all monitors stitched, uploaded to **Cloudflare R2** via
  presigned URLs, **7-day** retention.

---

## 4. Key architecture decisions (locked)

These are settled. Do not relitigate without the human's explicit say-so. Full
rationale is in [`docs/DECISIONS.md`](docs/DECISIONS.md).

- **Agent stack**: C# / .NET 8, **self-contained single-file** `.exe` (target PCs
  need nothing pre-installed). Two-part: **Windows Service + user-session helper**.
- **Sampling model**: **interval aggregation** ("Model B"). Sample foreground
  state every **3–5 s in memory**; write/flush a closed interval only when the
  **app changes, browser domain changes, or activity state flips**. Idle
  **splits** the interval so "active time" stays honest.
- **Idle threshold**: **5 minutes** of no input → idle (unless media is playing).
- **Local buffer**: **SQLite**, encrypted, in an ACL-locked folder under
  `C:\ProgramData\...`. Survives offline/reboot. Cap ~**2–3 months** (safety net;
  synced rows are purged immediately).
- **Sync**: every **5 min**, gzipped JSON batch over HTTPS. Exponential backoff.
  **Server must ACK before the agent purges** local rows (no data loss).
- **Server DB**: **MongoDB**.
- **Console stack**: **Next.js** (dashboard UI **+** API routes) + MongoDB +
  Cloudflare in front, on a VPS. Load is trivial (~0.3 req/s at 100 agents).
- **Identity**: each device has an opaque, permanent **DeviceId GUID** generated
  at enrollment. `pcName`, `hostname`, `osUsername`, `mac` are **mutable
  attributes** hanging off it. Renaming never changes identity. A reformatted
  machine **re-enrolls as a new device** (no hardware dedup in v1).
- **Enrollment**: MSI carries an **org key** + `PCNAME` install param → first run
  calls `/enroll` → server returns a **per-device token** stored via Windows
  **DPAPI (machine scope)**. Per-device tokens so one machine can be revoked alone.
- **Deployment**: ~**100** machines, MSI installed **manually PC-to-PC** by IT.
  PC name set at install and changeable later from the dashboard.
- **Lifecycle**: remote **signed auto-update** (poll manifest → download from R2 →
  verify signature → swap → restart). Service auto-restarts (SCM recovery); a
  watchdog keeps the helper alive; a **heartbeat** drives online/offline status.
  We defend against **standard users only** (employees are not admins); the
  heartbeat is the backstop if an agent goes silent.
- **Code signing**: **self-signed + internal trust** for v1 (push cert to company
  machines) until the CEO approves a purchased certificate.
- **Update hosting**: **Cloudflare R2** + a small version-manifest JSON.
- **Server retention**: **indefinite**, with **manual per-PC archive/delete** by
  an admin.
- **Time**: store everything in **UTC**, display in the viewer's **local time**
  (the team spans multiple time zones).

---

## 5. The API contract is sacred

The agent and the console communicate through a small, versioned HTTP contract:
`/enroll`, `/ingest`, `/heartbeat`, plus an update manifest. The **authoritative
definition** is [`docs/API-CONTRACT.md`](docs/API-CONTRACT.md) in THIS repo.

- Both `atlas-agent` and `atlas-console` must implement it **exactly**.
- If a change is needed, change `docs/API-CONTRACT.md` **first**, bump the version,
  then update both sides. Never let the two implementations drift silently.

---

## 6. How we work (workflow for AI agents)

This project is built by AI coding agents (Claude Code and Opencode), with a human
conductor and Claude as architect/reviewer.

- **Phased delivery.** Work proceeds phase by phase per [`docs/PLAN.md`](docs/PLAN.md).
  Each phase has a checklist. Tick items as you complete them.
- **A phase is not "done"** until: all checklist items are ticked, tests are green,
  and the human has manually verified the phase goal. Then the human checkmarks the
  phase and we move on.
- **Review checkpoint** after every phase: the human returns to Claude for a
  codebase review before starting the next phase.
- **Consult skills before coding** (see §7).
- **Keep READMEs current.** Human-readable `README.md` in each repo is updated as
  features land.
- **Tests gate phases.** Pure logic (interval aggregation, the 3-state machine,
  categorization, sync/retry, payload building) must have unit tests. OS-interop
  (reading the real foreground window / address bar) is verified manually, not
  unit-tested.
- **Do not invent scope.** If a decision isn't covered here or in `docs/`, ask the
  human rather than guessing.

---

## 7. Skills (read before you code)

Each child repo has skills under `.claude/skills/`. Before writing or modifying
code in a repo, **load and follow its relevant skill(s)**:

- `atlas-agent` → `dotnet-best-practices` (community) and `atlas-agent-conventions`
  (project-specific: silent-running rules, service+helper pattern, buffer/sync
  discipline, what to test vs verify manually).
- `atlas-console` → `next-best-practices` (community) and `atlas-api-contract`
  (project-specific: how to implement the API contract, device identity, UTC rules).

Skills are markdown instruction files. Claude Code surfaces them automatically and
loads the full text on demand. Opencode reads this directive and the skill files
directly. **The rule "consult the relevant skill before coding" applies to every
agent regardless of tooling.**

---

## 8. Repos & remotes

- Mother: `Atlas-Green/` (this folder). GitHub: `atlas-green` (to be created).
- `atlas-agent` → https://github.com/sadmanryanriad/atlas-agent.git
- `atlas-console` → https://github.com/sadmanryanriad/atlas-console.git

This mother repo's `.gitignore` excludes `atlas-agent/` and `atlas-console/` so the
three histories stay independent.

---

## 9. Glossary (shared vocabulary — use these exact terms)

- **Agent** — the Windows program (atlas-agent). Never "client app" or "spyware".
- **Console** — the Next.js dashboard + API (atlas-console).
- **DeviceId** — the permanent opaque GUID identifying a machine.
- **Interval** — one aggregated activity record (app/domain/state + start/end).
- **State** — `active` | `passive_media` | `idle`.
- **Heartbeat** — periodic agent ping that drives online/offline status.
- **Manifest** — the JSON describing the latest agent version for auto-update.
- **Org key** — the shared secret used only during enrollment.
- **Device token** — the per-device secret used for all post-enroll API calls.
