# Atlas Green — Pre-Implementation Design Review

**Reviewer model:** Kimi K2.7 Code  
**Date:** 2026-06-23  
**Scope:** Mother repo planning docs, `atlas-agent` conventions, `atlas-console` contract

## Executive Summary (5 lines)

The Atlas Green design is coherent and appropriately scoped for ~100 Windows desktops, with a sound service/helper split, a sensible interval-aggregation model, and honest threat-modeling. The largest risks are not architectural ambition but under-specified mechanics: URL reading via UI Automation is less reliable than the plan asserts, self-signed binaries will likely break silent deployment through SmartScreen/AV, and the service-to-helper launch mechanism is hand-waved. Several contract and data-model details (ACK semantics, batchId uniqueness, revocation, clock skew) need tightening before implementation starts. Phasing is generally right, though packaging/hardening and AV validation should move from Phase 7 into the packaging phase.

---

## Design Strengths

- **Service + Helper split** (`docs/ARCHITECTURE.md` §2) is the correct pattern for a silent, persistent Windows agent that still needs desktop visibility.
- **ACK-before-purge + `batchId` idempotency** (`docs/API-CONTRACT.md` §2, `docs/DECISIONS.md` #9) is the right reliability model for an offline-capable outbox.
- **Opaque `DeviceId` GUID with mutable attributes** (`docs/ARCHITECTURE.md` §5, `docs/DECISIONS.md` #13) cleanly separates identity from labels and avoids merge/split bugs on PC rename.
- **Store UTC, display local** (`docs/DECISIONS.md` #22) is the only sane choice for a multi-timezone team.
- **Phase 0 thin vertical slice** (`docs/PLAN.md` Phase 0) — heartbeat end-to-end before capture logic — de-risks the unknowns early.
- **Honest threat model** (`docs/ARCHITECTURE.md` §6) — defending only against standard users — avoids unrealistic security claims.
- **Self-contained .NET 8 single-file** (`docs/DECISIONS.md` #3, #19) is a good Windows-native deployment choice for company-owned machines.

---

## Findings

### Critical

#### C1 — Self-signed code signing will break the “silent” deployment promise
- **Files:** `AGENTS.md` §4; `docs/DECISIONS.md` #18; `docs/PLAN.md` Phases 5–6
- **Problem:** A self-signed MSI/`.exe` will trigger Windows SmartScreen and likely Microsoft Defender/SmartScreen reputation warnings on first run, requiring user interaction. Phase 7 lists “AV false-positive review” as a late hardening item, but by Phase 5 the installer must already be invisible. A non-admin user clicking “Don’t run” defeats the entire project.
- **Recommendation:** Either budget for a standard (OV) code-signing certificate before Phase 5, or explicitly require IT to pre-trust the self-signed cert on every target PC before the MSI is pushed. Start AV/reputation testing in Phase 5, not Phase 7: submit the signed binary to Microsoft Defender SmartScreen for analysis and document expected SmartScreen behavior per installer variant.

#### C2 — UI Automation URL capture is overstated as “all browsers / profiles / incognito”
- **Files:** `AGENTS.md` §3; `docs/DECISIONS.md` #2; `docs/PLAN.md` Phase 3
- **Problem:** Chromium can be launched with accessibility features disabled (`--disable-features=Accessibility`), Firefox’s accessibility tree for the address bar is inconsistent across versions, incognito windows do not guarantee a readable address-bar UIA control, and UIA can return stale or mid-typing values. The plan calls this the “key advantage over a browser extension,” but the reliability guarantee is weaker than claimed.
- **Recommendation:** Define explicit fallback behavior when UIA cannot resolve a URL: mark the interval as a browser with `domain: null` and `pageTitle` from the window title. Maintain and test a browser compatibility matrix (Chrome, Edge, Brave, Opera, Firefox) across normal windows, incognito/private windows, and at least one secondary profile. Document known limitations and do not market the feature as universal until validated.

#### C3 — Service-to-helper launch into the active desktop is under-specified
- **Files:** `docs/ARCHITECTURE.md` §2; `docs/PLAN.md` Phase 1
- **Problem:** The plan says the service “launches + supervises” the helper and restarts it on death or user login, but it does not specify *how* a Session-0 service obtains an interactive user token, selects the correct session, or handles multiple concurrent sessions/RDP. UIA also behaves differently depending on the helper’s token/integrity level. This is a central feasibility risk for the whole capture pipeline.
- **Recommendation:** Add a short design note or ADR before Phase 1 begins: use `WTSQueryUserToken`/`CreateProcessAsUser` with the target session ID, react to `WTS_SESSION_CHANGE` events, and explicitly decide whether the helper runs as the logged-on user (preferred for UIA) or as SYSTEM. Validate the launch mechanism in Phase 0/1 before building capture logic on top of it.

---

### High

#### H1 — `/ingest` ACK semantics can silently lose data on partial acceptance
- **File:** `docs/API-CONTRACT.md` §2
- **Problem:** The response `accepted: 37` is documented as the signal to purge. If the server validates per interval, drops a few malformed rows, and returns a count for the rest, the agent will purge rows that were never stored. The contract does not state whether `/ingest` is atomic.
- **Recommendation:** Make `/ingest` atomic: persist all intervals in the batch or none. Return `accepted` only after durable storage of the entire batch. If partial acceptance is required, return the list of accepted interval IDs and have the agent purge only those. Add a `422 Unprocessable Entity` response for batches containing invalid rows and update the contract accordingly.

#### H2 — `batchId` format is not robust against collisions
- **Files:** `docs/API-CONTRACT.md` §2; `docs/ARCHITECTURE.md` §7
- **Problem:** The example `batchId` is `b-2026-06-22T09:10:00Z-001`, based on time plus a sequence. A reinstalled agent, a reset sequence counter, or a cloned disk image can reuse a `batchId`. The architecture index note calls it “unique-ish,” which is not strong enough for idempotency. A unique index on `{batchId}` alone also breaks if two devices collide.
- **Recommendation:** Embed the `deviceId` in the `batchId` (e.g., `b-<deviceId>-<timestamp>-<random>`). Use a unique compound index `{deviceId, batchId}` on `intervals` or store batch metadata in a separate `batches` collection keyed by `{deviceId, batchId}`.

#### H3 — Revocation model allows a revoked device to re-enroll trivially
- **Files:** `docs/API-CONTRACT.md` §5; `docs/ARCHITECTURE.md` §6
- **Problem:** When the agent receives `401`, it re-enrolls with the org key. If a device is explicitly revoked (token invalidated), nothing in the contract prevents `/enroll` from issuing a fresh token for the same `deviceId`, effectively un-revoking the device. The contract does not define server-side device blocking.
- **Recommendation:** Add a `blocked`/`revoked` flag to the `devices` collection. If set, `/enroll` must return `403 Forbidden` for that `deviceId`. Document org-key rotation: when rotating the org key, old keys should be invalidated after a grace period so lost/revoked agents cannot re-enroll.

#### H4 — No monotonic-clock handling for interval timestamps
- **Files:** `docs/API-CONTRACT.md` §5; `docs/DECISIONS.md` #22
- **Problem:** The agent trusts the local UTC clock. NTP corrections, manual clock changes, or timezone travel can make `endUtc` earlier than `startUtc` or produce out-of-order intervals, breaking aggregation queries and timelines.
- **Recommendation:** Measure interval durations with a monotonic clock (`Stopwatch.GetTimestamp`) and anchor UTC to a stable reference point. Detect and handle backward clock jumps: either drop/adjust the offending interval or flag it for server-side anomaly handling. Store `receivedAt` server-side as a sanity check.

---

### Medium

#### M1 — Helper token/integrity level is unspecified and affects UIA reliability
- **Files:** `docs/ARCHITECTURE.md` §2; `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §2
- **Problem:** If the helper runs as SYSTEM in the user session, User Interface Privilege Isolation (UIPI) and lower-integrity target apps can block UIA reads. If it runs as the logged-on user, the service must obtain a non-elevated user token, which is non-trivial.
- **Recommendation:** Explicitly design the helper to run as the interactive user, not SYSTEM. Document the token-duplication path and test UIA against elevated browsers, UAC secure-desktop prompts, and sandboxed apps. Note these as known blind spots.

#### M2 — “Encrypted SQLite” lacks implementation detail
- **Files:** `docs/DECISIONS.md` #8; `docs/ARCHITECTURE.md` §6; `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §4
- **Problem:** The design says the local buffer is encrypted but does not specify the mechanism (SQLCipher, System.Data.SQLite encryption, DPAPI-encrypted file, etc.). Each option has different native dependencies, licensing, and key-management implications.
- **Recommendation:** Specify the exact SQLite encryption approach (e.g., SQLCipher via SQLitePCLRaw bundles, or DPAPI-encrypting the whole database file with a key protected at machine scope). Document that this protects against standard users, not against local administrators.

#### M3 — Buffer cap is vague
- **Files:** `docs/DECISIONS.md` #10; `atlas-agent/.agents/skills/atlas-agent-conventions/SKILL.md` §4
- **Problem:** “Cap ~2–3 months” is not a concrete boundary. A heavy browser user may generate far more rows than a light user, and disk or SQLite size could grow unexpectedly. The policy for eviction when the cap is reached is also unspecified.
- **Recommendation:** Define the cap as a row count and/or byte budget (e.g., max 100,000 unsynced rows or 50 MB) with FIFO eviction and a warning threshold. Surface buffer pressure in the heartbeat `health` block and dashboard.

#### M4 — Dashboard aggregation queries may not scale without strict date bounding
- **Files:** `docs/ARCHITECTURE.md` §7; `docs/PLAN.md` Phase 4
- **Problem:** Retention is indefinite and the `intervals` collection will grow continuously. Timeline and summary queries without tight date filters will eventually scan millions of documents.
- **Recommendation:** Require date-window parameters on every aggregation query and add compound indexes such as `{deviceId, startUtc, category}` and `{employeeId, startUtc, category}`. For v2, consider pre-aggregated daily summaries or a time-series collection pattern to keep dashboard queries fast.

#### M5 — Category rule precedence is undefined
- **Files:** `docs/ARCHITECTURE.md` §7; `atlas-console/.agents/skills/atlas-api-contract/SKILL.md` §4
- **Problem:** An interval may match both a domain rule and a process rule, or multiple rules of the same type. The design does not state which rule wins.
- **Recommendation:** Add a `priority` or `order` field to the `categories` collection and document the precedence order (e.g., exact match > domain > process). Implement deterministic tie-breaking in the ingest resolver and bulk re-resolve path.

#### M6 — Org-key rotation is not addressed
- **Files:** `docs/API-CONTRACT.md`; `docs/DECISIONS.md` #15
- **Problem:** The org key is embedded in the MSI. If it leaks or is suspected compromised, there is no documented rotation path short of rebuilding and redeploying the MSI to every machine.
- **Recommendation:** Allow multiple active org keys on the server with a created/revoked date. Document a rotation runbook: add a new key, rebuild the installer, deploy gradually, then revoke the old key after a grace period. Consider whether old agents should be allowed to re-enroll during the grace period.

---

### Low

#### L1 — Contract-version mismatch handling is unspecified
- **File:** `docs/API-CONTRACT.md` §5
- **Problem:** The agent sends `X-Atlas-Contract: 1`, but the contract does not say how the server responds to an unsupported major version or what the agent should do.
- **Recommendation:** Define a `426 Upgrade Required` or `400` response with an `X-Atlas-Contract-Expected` header. Define agent behavior: stop ingest/sync, surface the mismatch via heartbeat, and rely on the auto-update path to resolve it.

#### L2 — Manifest JSON itself is not signed
- **File:** `docs/API-CONTRACT.md` §3–4
- **Problem:** The heartbeat points the agent to a manifest on R2. The manifest is not signed, so an R2 compromise could redirect agents to a malicious package URL. The agent verifies the package signature and hash, which mitigates execution of a bad binary, but it could still be used to deny updates.
- **Recommendation:** Sign `manifest.json` with the same code-signing key and verify the manifest signature before trusting the `package` block. Alternatively, serve the manifest from the console API under HTTPS so it is protected by the TLS chain and not writable from the object-store side.

#### L3 — Missing index on `deviceTokens.tokenHash`
- **Files:** `docs/ARCHITECTURE.md` §7; `atlas-console/.agents/skills/atlas-api-contract/SKILL.md` §5
- **Problem:** Every ingest/heartbeat request must look up the device token by hash. Without an index, this will table-scan the `deviceTokens` collection.
- **Recommendation:** Add a unique index on `tokenHash` (or `{tokenHash, revokedAt}` if revocation queries need it).

#### L4 — Browser/URL test matrix is not documented
- **File:** `docs/PLAN.md` Phase 3
- **Problem:** The Phase 3 exit gate says URL capture must be verified “live” but does not list which browsers, profiles, or incognito modes must pass.
- **Recommendation:** Define an explicit manual QA matrix: Chrome, Edge, Brave, Opera, Firefox; each in normal, incognito/private, and a secondary profile. Include a case with UIA/accessibility disabled to verify fallback behavior.

---

### Nice-to-have

#### N1 — Add an admin audit-log collection
- **File:** `docs/ARCHITECTURE.md` §7
- **Problem:** There is no design for logging admin actions such as category changes, device revocation, employee mapping changes, or data deletion.
- **Recommendation:** Add an `auditLogs` collection with actor, action, target, and timestamp to support compliance and incident review.

#### N2 — Define device archive / decommission / erasure workflow
- **File:** `docs/ARCHITECTURE.md` §7
- **Problem:** The `archived` flag is mentioned, but the workflow for retiring a device, deleting historical data, or handling legal erasure requests is not detailed.
- **Recommendation:** Document the full lifecycle: archive (hide from dashboard), delete intervals (per-device erasure), and re-enrollment of a reformatted machine. Review against local employment/privacy law requirements.

#### N3 — Formalize the 100-agent load/chaos test
- **File:** `docs/PLAN.md` Phase 7
- **Problem:** “Load sanity at 100 devices” is a checklist item without a specific scenario or pass criteria.
- **Recommendation:** Define a soak test: simulate 100 agents for 24 hours with realistic interval generation, verify zero dropped batches, dashboard query latency under 2 seconds, and stable MongoDB memory/CPU on the VPS.

---

## Reviewed Dimensions Summary

1. **Technical feasibility** — Service/helper model is sound but helper launch details are missing; UIA URL reading is more fragile than claimed; 3-state model and idle splitting are feasible with care.
2. **Architecture & scalability** — Appropriate for 100 agents; dashboard queries need date bounding and indexes as data grows.
3. **Security & privacy** — Token, DPAPI, and ACL model are reasonable for the stated threat model. Revocation and org-key rotation need strengthening. Self-signed signing is a deployment risk.
4. **API contract** — Core shape is good, but ACK semantics, `batchId` uniqueness, contract-version mismatch, and revocation behavior need tightening.
5. **Data model & identity** — `DeviceId` GUID approach is solid. Add token-hash index, category precedence, and `{deviceId, batchId}` uniqueness.
6. **Sync/buffer reliability** — ACK-before-purge and idempotency are correct. Define atomicity, buffer cap, and monotonic-clock handling.
7. **Phasing** — Phase 0 is the right thin slice. Consider moving AV/SmartScreen validation and service-install zero-footprint checks earlier than Phase 7.
8. **Testability** — Unit-test scope is right; manual verification for OS interop is appropriate. Document the browser QA matrix.
9. **Edge cases / risks** — RDP/multi-session, sleep/resume, clock skew, AV false positives, and hardened browser configs are acknowledged but need deeper design before Phase 7.
10. **Over-engineered / missing / better approaches** — Nothing is materially over-engineered for the requirements. Main gaps are operational: audit logs, org-key rotation, decommission workflow, and concrete load testing.
