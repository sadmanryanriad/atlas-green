# Atlas Green

Internal employee-activity monitoring for company-owned Windows desktops.

This is the **mother repository** — it holds the project plan, architecture, and
the API contract that the two application repos implement. It contains **no
application code**.

## Repositories

| Repo | What it is |
|------|-----------|
| **atlas-green** (this) | Plan, architecture, API contract, cross-cutting docs. |
| [atlas-agent](https://github.com/sadmanryanriad/atlas-agent) | The silent Windows background agent (C# / .NET 8). |
| [atlas-console](https://github.com/sadmanryanriad/atlas-console) | Admin dashboard + ingest API (Next.js + MongoDB). |

All three are checked out side-by-side inside one local `Atlas-Green/` folder:

```
Atlas-Green/
├── docs/             ← plan, architecture, API contract, decisions
├── AGENTS.md         ← whole-system context for AI agents
├── atlas-agent/      ← child repo
└── atlas-console/    ← child repo
```

## Start here

- **[docs/PLAN.md](docs/PLAN.md)** — the phased build plan with checklists.
- **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)** — how the system fits together.
- **[docs/API-CONTRACT.md](docs/API-CONTRACT.md)** — the agent ↔ console contract.
- **[docs/DECISIONS.md](docs/DECISIONS.md)** — every locked decision and why.
- **[AGENTS.md](AGENTS.md)** — read this first if you are an AI coding agent.

## What it does (plain English)

A tiny program runs silently on each office desktop. It records which app or
website the employee is using and whether they're actively working, passively
watching media, or away — then sends that to a central dashboard where managers
can see how work time is spent. It is installed by IT, runs invisibly, and updates
itself remotely.

## Status

🚧 Planning complete. Implementation begins with **Phase 0** (see the plan).
