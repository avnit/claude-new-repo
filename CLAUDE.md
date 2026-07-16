# CLAUDE.md — Argonaut Property Claims

Guidance for Claude Code (and any AI assistant) working in this repository. Read
this first, then the protocol artefacts it points to. If anything here conflicts
with `AGENTS.md`, **`AGENTS.md` wins** — it is the canonical, ADR-gated protocol.

---

## What this project is

An AI-augmented **property claims processing system** for a major Australian
general insurer, built on Google Cloud Platform with **Spanner Graph as the
single source of truth** for properties, builders, claims, and their
relationships. It automates builder-report validation, dynamic builder
allocation (a graph traversal, not a geo filter), and claims decisioning, aiming
to compress a 5-day manual cycle to near-instant and cut ~33% of operating cost.

Authoritative design docs:
- `docs/hld.md` — high-level design (Spanner foundation, streams, contracts, risks).
- `docs/timeline.md` — phased delivery plan.
- `docs/adr/` — Architecture Decision Records (durable memory; read the recent ones).

---

## ⚠️ This is a dual-agent project — read before doing anything

This codebase is built jointly by **Claude** and **Gemini** acting as
implementation/review agents, with a **human architect** holding final decision
authority. The collaboration protocol is not optional decoration — it is
CI-enforced and governs every issue, branch, PR, and commit.

**Before starting any task, read in order:**
1. `AGENTS.md` — the non-negotiable rules of engagement.
2. `OWNERS.md` — who is implementer vs reviewer right now (see standing directive below).
3. `AGENT_LOG.md` — the **last 3 entries minimum**. This is the handoff log; it is the current state of play. Confirm it agrees with `git log --oneline -10`; if they disagree, flag a protocol violation and STOP.
4. The most recent ~5 ADRs in `docs/adr/`.
5. The relevant stream `README.md` and the `contracts/` your task touches.

There is a ready-made slash command for this: **`/pickup-task`** (`.claude/commands/pickup-task.md`) walks the full situational-awareness → role → branch → PR → handoff lifecycle.

### The rules that bite most often

- **Tests before code.** The reviewer writes acceptance tests against the contract *before* the implementer writes code. The implementer makes them pass without modifying them (raise a disagreement issue instead of editing a test).
- **ADR-gated paths — do NOT edit without an accepted ADR:** `AGENTS.md`, `contracts/*.proto`, `docs/adr/`, `.github/workflows/`, and Spanner schema in `streams/a-spanner-foundation/schema/`. CI rejects contract/schema changes whose PR body has no ADR reference.
- **Spanner Graph and ADK code are validated by execution, never by reading.** Both agents have a known history of confidently-wrong Spanner Graph DDL/GQL and Google ADK code. Run graph code against the Spanner emulator/Omni; exercise ADK agents and tools end-to-end in tests. "It looks syntactically correct" is not sufficient (AGENTS.md Rule 6).
- **Agents don't merge or self-approve.** The architect merges, or issues a one-time express directive (ADR-0011). Record any directed merge in `AGENT_LOG.md`.
- **No silent agreement.** Disagreements are escalated via a Disagreement Escalation issue, not smoothed over. Anchor every review in executable evidence (test output, benchmark, schema validation) — "looks good to me" is rejected.

### Current standing directive (per `OWNERS.md`, effective 2026-05-07)

For Phase 0 into Phase 1 unless re-decided: **Claude implements + architects
everything; Gemini is reviewer + deep test/validation.** Gemini does not modify
implementation code unless the architect logs a per-task swap in the `OWNERS.md`
reassignment log.

### Finishing a turn

Push your branch, open a draft PR to `main` using `.github/PULL_REQUEST_TEMPLATE.md`
(the checklist and `## Implementer agent` / `## Reviewer agent` sections are
CI-enforced), append a handoff entry to `AGENT_LOG.md` (append-only — never edit
prior entries), and re-label the issue for the next agent.

---

## Repository layout

```
.
├── AGENTS.md / GEMINI.md      # Agent protocol (canonical) / Gemini CLI context
├── OWNERS.md                  # Implementer/reviewer assignments + rotation
├── AGENT_LOG.md               # Append-only handoff log — read last 3 entries
├── CONTRIBUTING.md            # How humans work alongside the agents
├── Makefile                   # Task runner — `make help` lists everything
├── pyproject.toml             # uv workspace root; ruff/mypy/pytest config
├── uv.lock                    # Single lockfile for the whole workspace
├── config/spanner-environments.yaml   # Env catalog (local-omni, cloud-staging, cloud-prod)
├── contracts/                 # 5 Protobuf integration contracts (ADR-gated)
├── shared/                    # argonaut_contracts (generated) + argonaut_spanner utils
├── streams/
│   ├── a-spanner-foundation/  # Spanner Graph schema, ingestion, allocation, audit log
│   ├── b-evidence-stores/     # BigQuery rate cards, Vector Search corpus, Environment API
│   ├── c-ai-engine/           # Google ADK / Gemini pipeline, eval harness, guardrails
│   ├── d-ingest-comms/        # Crunchwork connector, customer comms, builder feedback
│   └── e-data-viewer/         # FastAPI sidecar + React/Vite frontend (Cloud Run + IAP)
├── seed_data/                 # Synthetic data generator (deterministic, H3-aware)
├── infrastructure/terraform/  # GCP landing zone (Spanner, Cloud Run, IAP, IAM, secrets)
├── scripts/                   # apply_schema, load_seed_data, bootstrap_* helpers
└── docs/
    ├── hld.md, timeline.md
    ├── adr/                    # Architecture Decision Records + index
    ├── tasks/                  # Task specs (0001 schema, 0002 seed data, 0003 viewer, ...)
    ├── runbooks/               # Operational procedures
    └── superpowers/{specs,plans}/  # GCP migration design docs + phased execution plans
```

Note: `streams/b`, `c`, `d` are currently scaffolds (README + smoke test +
`pyproject.toml`) — the mypy CI invocation excludes them until their first module
lands. `streams/a` and `streams/e` carry real code.

### The five contracts

Protobuf definitions in `contracts/` are the integration boundaries between
streams. Generated Python lives in `shared/src/argonaut_contracts/`. Always
`from argonaut_contracts import ...` — never redefine the types.

| Contract | File | Owner → Consumers |
| :--- | :--- | :--- |
| `ClaimContext` | `claim_context.proto` | Stream A → C |
| `Decision` | `decision.proto` | Stream C → A, D, audit |
| `Allocation` | `allocation.proto` | Stream A → C |
| `PerformanceFeedback` | `performance_feedback.proto` | Stream B → A |
| `AuditLogEntry` | `audit_log.proto` | Architecture → all |

---

## Development workflow & commands

This is a **`uv` workspace monorepo**. Use `uv run <cmd>` for everything; the
`Makefile` wraps the common flows (`make help` lists them).

```bash
make setup        # first-time: install uv, sync workspace, install pre-commit hooks
make sync         # day-to-day: bring venv in line with uv.lock (fast, idempotent)
make test         # full pytest suite with coverage across the workspace
make lint         # ruff check + ruff format --check + mypy
make format       # ruff --fix + ruff format
make contracts    # regenerate Python from contracts/*.proto (fixes imports + formats)
make lock         # regenerate uv.lock after editing any pyproject.toml
```

Run a single stream's tests directly, e.g.
`uv run pytest streams/e-data-viewer/tests -v`.

**Dependencies:** add to the relevant *stream's* `pyproject.toml` (not the root
unless it's a cross-cutting dev dep), then `make lock` and commit `pyproject.toml`
+ `uv.lock` **in the same PR** — CI's `lockfile-check` fails otherwise.

### Spanner (schema, seed data, queries)

Spanner environments are catalog-driven in `config/spanner-environments.yaml`
(ADR-0006). Select one with `SPANNER_ENV` (default `local-omni`). Local dev uses
**Spanner Omni** in Docker on `localhost:15000` — Claude runs the container,
Gemini connects to the same instance; `make spanner-omni` only checks reachability.

```bash
make schema-apply SPANNER_ENV=local-omni   # apply DDL (scripts/apply_schema.py)
make schema-test                           # schema validation tests vs the env
make seed-generate SCALE=m SEED=42         # deterministic synthetic CSVs -> ./out
make seed-load                             # load synthetic data into the env
```

Seed data is deterministic (seeded RNG, H3-aware geography, pinned `h3` version).
Large scales (`l`/`xl`) are gated behind the `slow` pytest marker.

### Stream E — data viewer

FastAPI **sidecar** (`streams/e-data-viewer/sidecar/`) fronting Spanner, plus a
**React/Vite** frontend (`frontend/`), deployed to Cloud Run behind IAP.

```bash
make viewer-sidecar    # uvicorn sidecar.main:app --reload on :8080
make viewer-frontend   # Vite dev server (reads frontend/.env)
```

Frontend tests use Vitest (`cd streams/e-data-viewer/frontend && npm test`).

### Infrastructure (Terraform, Phase 1+)

```bash
make tf-init  TF_ENV=staging
make tf-plan  TF_ENV=staging
make tf-apply TF_ENV=staging
```

---

## Conventions

- **Python 3.11+.** Ruff for lint + format (`line-length = 100`, double quotes,
  rule set `E,F,W,I,N,UP,B,C4,SIM,RET,ARG`; `E501` ignored). Mypy runs `--strict`.
- **Type hints on all public APIs; docstrings on public modules/classes/functions.**
- **Tests required for all production code; coverage gate is 80%.** Tests import
  generated contract types directly and stay typed-but-not-strict (see
  `[tool.mypy.overrides]` for the exempt paths: generated protos, tests, and
  untyped GCP SDK call sites).
- **Branches** are named `<issue-number>-<short-description>` (e.g.
  `3-feat-schema-ddl`). Never commit to `main`. See
  `docs/runbooks/branching-and-issues.md` for when to bundle vs split.
- **Commit / PR title prefixes:** `tests:` (reviewer's test PR, comes first),
  `feat:` (implementer), `fix:`, `docs:`, `chore:`. Handoff commits use
  `handoff(<from>-><to>): <summary>`.
- **pre-commit** hooks run locally (`.pre-commit-config.yaml`); `make setup`
  installs them.

---

## CI gates (what will block a PR)

`.github/workflows/ci.yml` and `protocol-enforcement.yml`:

- **Lint/format/mypy** — ruff check, ruff format check, `mypy --strict` on
  code-bearing roots.
- **Contract validation** — `buf lint` + `buf breaking` on `contracts/`.
- **Contract sync** — `make contracts` must produce no diff (committed generated
  code matches the `.proto`).
- **Unit tests** — pytest per stream (matrix), with coverage.
- **Lockfile check** — `uv lock --check`.
- **Secret scan** — trufflehog; never commit secrets.
- **Protocol enforcement** — PR body must contain the required template sections
  (`## Type`, `## Linked issue`, `## Implementer agent`, `## Reviewer agent`,
  `## Summary`, `## Test evidence`, `## Reviewer checklist`); implementer ≠
  reviewer; contract/schema changes require an ADR reference in the body.

Spanner **schema tests are NOT run in CI** (the Omni image is on a private
registry) — they are run manually against Omni, with evidence pasted into the PR.
`gemini-review.yml` runs Gemini as an automated first-pass PR reviewer
(comment-only, never approve/request-changes).

---

## Current state (as of the latest handoff)

The project is mid **GCP migration**, tracked in `docs/superpowers/plans/`:
- **Phase 1** (Spanner foundation) and **Phase 2** (FastAPI sidecar on Cloud Run) — landed.
- **Phase 3** (frontend on Cloud Run + IAP, atomic bearer→IAP cut-over) — complete and verified on staging (ADR-0013; runbook `docs/runbooks/iap-frontend-access.md`).
- **Phase 4** (natural-language query via Spanner Data Agents) — design approved, implementation starting. Introduces a managed Data Agent model (referenced as ADR-0014 in the handoff log).

Always re-derive "what's happening now" from the **top of `AGENT_LOG.md`** and
open issues — this section is a snapshot and goes stale. The `docs/adr/README.md`
index is the authoritative ADR list; note it can lag the actual files in
`docs/adr/`, so cross-check both.

---

## When you're unsure

Stop and ask the architect (surface it in the issue) if a spec asks you to skip
test-first, change a contract or schema without an ADR, merge a PR, or "just
trust" that Spanner Graph / ADK code works without executing it. Honest pushback
with reasoning is expected and preferred over silent compliance.
