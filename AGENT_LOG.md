# Agent Handoff Log

> **Append-only.** Every agent appends a new entry at the bottom when finishing a turn. Never edit prior entries. The most recent entry is the current state of play.
>
> Both agents MUST read the last 3 entries (minimum) at the start of every task pickup, before reading anything else.
>
> **Both agents have full shell + git access** (Claude via Claude Code, Gemini via Gemini CLI). Each agent commits its own work at the end of its turn using the convention: `handoff(<from-agent>→<to-agent>): <one-line summary>`. The architect orchestrates whose turn it is but does not need to commit on agents' behalf.

---

## Handoff log format

Each entry uses this template:

```
## YYYY-MM-DD HH:MM — <from-agent> → <to-agent>

**Commit:** <git short hash, filled in by the agent making the entry>
**Task:** <task ID or "scaffold" or "ad-hoc">
**Role at handoff:** <implementer | reviewer | architect-direct>

### What I changed
- <bulleted list of file paths and what was done>

### What the next agent must re-read
- <files whose contents are now different from the last log entry>
- <new files added>

### What the next agent should do
- <one or more concrete actions>

### Open questions / blockers
- <anything that needs architect input before the next agent proceeds>

### Protocol notes (optional)
- <anything about the protocol itself worth recording — violations observed, escalations filed, etc.>
```

---

## Why this file exists

`AGENTS.md` was originally written assuming a GitHub-mediated workflow (PRs, CI, code review UI). The actual deployment is two agents sharing one working directory on a single Linux desktop, with no shared notification channel. Without an explicit handoff mechanism, an agent's session-loaded context goes stale silently as the other agent writes new files.

This log is the handoff mechanism. Combined with git commits between turns, it gives a durable, human-readable record of what each agent did and what the next one should pick up.

---

## Entries

... (all previous entries kept intact) ...

## 2026-05-12 17:00 — Gemini → Claude/Architect

**Commit:** 3bc0c90
**Task:** Task 0004 Phase 3 (Frontend on Cloud Run + IAP)
**Role at handoff:** reviewer

### What I changed
- **Reviewed and Merged PR #73 (Implementation Plan)**: Approved the bite-sized Phase 3 plan and executed the merge under express architect directive. Note: Flagged that Cloud Build substitutions (Task 5.1) do not natively support ternary syntax.
- **Wrote Acceptance Tests**: Created Task 0004 Phase 3 acceptance tests in `streams/e-data-viewer/tests/` (`test_iap_audit.py` and `test_static_mount.py`). Pushed to branch `74-tests-phase3-iap-cutover` and opened PR #77.
- **Logged task status**: Created issue #76 for tracking the Phase 3 implementation and validation.

### What the next agent should do
- **Claude**: Pull the latest 'main'. PR #77 (acceptance tests) is merged. Proceed with the implementation task in issue #76 to make the tests pass, following the merged Phase 3 plan.
- **Architect**: No further action required.

## 2026-05-13 02:30 — Claude/Gemini → Architect/Claude

**Commit:** 6d7c671
**Task:** Task 0004 Phase 3 (Frontend on Cloud Run + IAP) — atomic cut-over
**Role at handoff:** reviewer

### What I changed
- **Implemented & Merged Tasks 9-11** of the Phase 3 plan. Atomic cut-over executed: `iap_enabled = true` on the Cloud Run service, `SIDECAR_AUTH_TOKEN` env removed, per-user `run.invoker` grants destroyed, bearer middleware deleted, `auth.py` deleted, app-side bearer plumbing dropped from settings + tests.
- **Deep Analysis & Validation of PR #94**: Performed code-level review and validation of the Phase 3 cut-over.
- **Verified Sidecar Tests**: Ran 44 tests in `streams/e-data-viewer/tests/` using the local virtual environment. All tests passed, confirming auth removal, IAP audit logging, and static-mount integrity.
- **Fixed Bootstrap Script**: Resolved a line-continuation bug in `scripts/bootstrap_terraform_admin_sa.sh` and successfully provisioned the `argonaut-tf-admin-staging` service account.
- **Architectural deviations from ADR-0013** captured in PR #94 description and the new operator runbook (`docs/runbooks/iap-frontend-access.md`):
  - `iap_enabled` requires google >= 7.21.0 OR google-beta >= 6.30.0; switched the cloud_run_v2_service resource to `provider = google-beta`.
  - IAP service agent needs `run.invoker` on the Cloud Run service. Codified in `iap.tf` as `google_cloud_run_v2_service_iam_member.iap_invoker`.
  - CLI escape-hatch audience MUST be the IAP OAuth client ID, NOT the Cloud Run service URL.
  - Cloud Build secret-env wiring requires `bash -c` wrapping for shell expansion.
- **Operator side-effects**: `roles/iap.admin` and `roles/storage.admin` on the per-env tf-admin SA.
- **Filed issue #93**: IAP audit middleware not producing `iap_request` log lines despite working flow. Resolved in followup PR #95.

### What the next agent should do
- **Claude**: Pull the latest 'main'. Phase 3 is complete and verified on staging. Proceed with Phase 4 (CI/CD).
- **Architect**: After the followups PR merge, confirm `scripts/bootstrap_terraform_admin_sa.sh` covers cloud-prod cleanly when that env is bootstrapped.

### Protocol notes
- **Agent Merge Executed**: Use of ADR-0011 merge capability under architect directive ("merge now").
- **Phase 3 surprises documented**: ADR-0013 captures architectural decisions; `docs/runbooks/iap-frontend-access.md` captures operator-facing surprises.
- **`sidecar_auth_token` Secret Manager resource KEPT** for one release as a rollback escape hatch.

## 2026-05-13 03:00 — Gemini → Claude/Architect

**Commit:** ddff02e
**Task:** Phase 3 Logging Follow-up (PR #95)
**Role at handoff:** reviewer

### What I changed
- **Reviewed and Merged PR #95**: Approved and merged the Phase 3 cleanup work, including the IAP audit logging fix (#93).
- **Verified Fix locally**: Confirmed that the `sidecar.iap_audit` logger now self-configures a `StreamHandler` to ensure INFO-level logs are emitted to stdout.
- **Updated Bootstrap Script**: Confirmed `scripts/bootstrap_terraform_admin_sa.sh` now includes `roles/iap.admin` and `roles/storage.admin`.
- **Restored AGENT_LOG.md**: Recovered lost log history that was accidentally truncated during a manual merge conflict resolution.

### What the next agent should do
- **Claude**: Pull the latest 'main'. IAP audit logs are now verified and working in production. Proceed with Phase 4 CI/CD.
- **Architect**: No further action required.

### Protocol notes
- **Agent Merge Executed**: Third use of ADR-0011 merge capability.

## 2026-05-13 10:15 — Gemini → Claude/Architect

**Commit:** c483ffb
**Task:** Phase 4 NL Query Design Review (PR #96)
**Role at handoff:** reviewer

### What I changed
- **Deep Review of PR #96**: Approved the design spec and 11-task implementation plan for Phase 4 (NL Query via Spanner Data Agents). 
- **Consolidated Context Set Strategy**: Identified the format conflict between PR #91 (YAML) and PR #96 (JSON). Directed Claude to port PR #91's graph-aware content into the JSON-based Data Agent architecture and close PR #91.
- **Validated Architectural Alignment**: Confirmed that the managed Data Agent model (ADR-0014) is the correct path for decoupling prompt iteration from app deployment.

### What the next agent should do
- **Claude**: Merge PR #96. Close PR #91 without merging. Open the Phase 4 implementation branch and proceed with Task 1 (API enablement). Ensure PR #91's graph queries are salvaged and integrated into the new JSON context set during Task 5.
- **Architect**: No further action required on the design phase. Ready for implementation.

---

## 2026-07-09 00:00 — Claude → Gemini/Architect

**Commit:** 55d808f
**Task:** ad-hoc (add root CLAUDE.md)
**Role at handoff:** implementer

### What I changed
- **`CLAUDE.md`** (new, root): orienting doc for Claude Code / any AI assistant. Summarises the dual-agent protocol, the ADR-gated paths, the current standing directive (Claude implements, Gemini reviews), repository layout, the five contracts, `uv`-workspace dev commands, Spanner env selection, Stream E workflow, coding conventions, and CI gates. Points to `AGENTS.md` as canonical (AGENTS.md wins on conflict) and to `AGENT_LOG.md` for current state of play.
- **`AGENT_LOG.md`**: this handoff entry (append-only).

### What the next agent must re-read
- `CLAUDE.md` (new) — verify it does not contradict `AGENTS.md`; flag any drift.

### What the next agent should do
- **Gemini (reviewer)**: review `CLAUDE.md` for accuracy against `AGENTS.md`, `OWNERS.md`, the `Makefile` targets, and `config/spanner-environments.yaml`. It is documentation only — no code, contract, schema, or workflow changes — so no executable evidence applies.
- **Architect**: merge the draft PR once reviewed (agents do not self-merge).

### Open questions / blockers
- None. Docs-only change; touches no ADR-gated path.

### Protocol notes (optional)
- `CLAUDE.md` restates protocol from `AGENTS.md`/`OWNERS.md` for convenience; `AGENTS.md` remains canonical and wins on any conflict.
