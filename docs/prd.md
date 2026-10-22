# Product Requirements Document (PRD)

**Project:** Agent Reliability Lab (ARL)
**Version:** 1.0 (MVP)
**Status:** Draft
**Owner:** Atul
**Last updated:** 2026-10-22

---

## 1. Overview

Agent Reliability Lab is "CI for AI agents." It lets a developer register any LLM agent, run it against a versioned suite of test tasks, score the results with deterministic checks and a calibrated LLM judge, compare the run against a baseline using statistics, and block a pull request if quality, cost, or latency regresses.

### 1.1 One-line pitch
Unit testing for non-deterministic AI agents, with a merge gate.

### 1.2 Problem statement
Agents are changed constantly (prompts, models, tools, retrieval settings). These changes silently degrade behavior because:
- Outputs are non-deterministic, so one-off manual checks are unreliable.
- "Correct" is often fuzzy (tone, completeness), so simple assertions are not enough.
- Quality, cost, and latency trade off against each other and are rarely tracked together.
- Failures are hard to diagnose without step-level traces.

Teams discover regressions from user complaints, not before shipping.

### 1.3 Vision
Every agent change is evaluated automatically, with statistical rigor, before it reaches users.

---

## 2. Goals and Non-Goals

### 2.1 Goals (v1)
| ID | Goal |
|---|---|
| G1 | Evaluate any agent via a small adapter contract (LangGraph adapter + generic HTTP adapter). |
| G2 | Produce trustworthy scores: deterministic checks plus an LLM judge whose agreement with humans is measured and published. |
| G3 | Handle randomness: repeated runs, pass-rate per task, variance, flaky-task detection. |
| G4 | Detect regressions with statistical significance, not raw score differences. |
| G5 | Gate pull requests in CI and post a readable report as a PR comment. |
| G6 | Give step-level visibility (traces) and failure categories so failures are diagnosable. |
| G7 | Be deployed, publicly viewable, and cheap to run. |

### 2.2 Non-Goals (v1)
- Multi-tenant SaaS, billing, organizations, RBAC beyond admin/viewer.
- Support for every agent framework (only LangGraph and HTTP in v1).
- Training or fine-tuning models.
- Production runtime monitoring of live user traffic (planned for v2).
- A visual dataset editor (datasets are YAML in Git for v1).
- Human annotation UI (labeling for judge calibration is done via CSV in v1).

---

## 3. Target Users and Personas

| Persona | Description | Needs |
|---|---|---|
| **Agent Developer (primary)** | Builds agents with LangGraph or similar; changes prompts/models often. | Fast feedback on whether a change helped or hurt; PR gate; traces for failures. |
| **Reviewer / Tech Lead** | Reviews PRs and decides what ships. | One-glance verdict, list of regressions, cost/latency impact. |
| **Evaluator / QA Engineer** | Owns the test suite. | Easy dataset authoring, versioning, flaky-test visibility, judge reliability numbers. |
| **Hiring manager / visitor (secondary)** | Opens the public demo. | Instant understanding, seeded demo data, no login or API keys needed. |

---

## 4. User Stories and Acceptance Criteria

### Epic A: Agent registration
**A1. As a developer, I can register an agent so the lab can run it.**
- Given a YAML config (name, adapter type, entrypoint or URL, model, temperature, prompt reference), the system validates it and stores an `agent_version` identified by a content hash.
- Invalid configs return a specific, field-level error.

**A2. As a developer, I can use the HTTP adapter for any agent.**
- The lab POSTs `{task_input, config}` and accepts the documented `AgentResult` JSON.
- Responses that do not match the schema are recorded as `format_error`, not crashes.

### Epic B: Datasets
**B1. As an evaluator, I can define tasks in YAML with expectations.**
- Task supports: `id`, `input`, `expected.tool_calls`, `must_contain`, `must_not_contain`, `judge_rubric`, `max_steps`, `tags`, `critical` flag.
- Schema validation runs in CI; duplicate IDs are rejected.

**B2. As an evaluator, datasets are versioned and immutable.**
- A reference like `refunds@v3` always resolves to the exact same task set (Git tag or DVC revision).

### Epic C: Running evaluations
**C1. As a developer, I can run a suite from the CLI or API.**
- Runs tasks concurrently with a configurable limit.
- Each task runs `N` repeats (default 3).
- Per-task timeout (default 60s) marks the task `timeout`.

**C2. As a developer, infra errors are retried but agent errors are not.**
- HTTP 5xx, network errors, and provider rate limits retry with backoff (max 3).
- Wrong answers are never retried.

**C3. As a developer, a run cannot exceed a cost budget.**
- When projected or actual spend exceeds `MAX_COST_PER_RUN_USD`, the run stops, is marked `aborted_budget`, and partial results are preserved.

**C4. As a developer, every run is reproducible.**
- A run stores: agent version hash, git SHA, model, temperature, seed (if supported), dataset version, evaluator versions, judge model and prompt hash.

### Epic D: Evaluation and scoring
**D1. Deterministic evaluators** (tool-call match, forbidden/required strings, JSON schema, step limit, duplicate side-effect detection).

**D2. LLM judge** returns structured JSON (`score`, `pass`, `reasoning`, `failure_category`) at temperature 0.

**D3. Judge calibration.**
- A labeled calibration set (>= 50 items) is stored.
- The system computes Cohen's kappa and raw agreement between judge and human labels and displays them.
- A weekly job re-runs calibration and alerts if kappa drops below a threshold.

**D4. A task passes only if all deterministic checks and the judge pass.**

**D5. Failures are categorized** (`wrong_tool`, `missing_tool`, `hallucination`, `loop`, `format_error`, `tone_issue`, `timeout`, `infra_error`, `other`).

### Epic E: Comparison and regression detection
**E1. As a reviewer, I can compare a candidate run to a baseline.**
- Per-task paired comparison: regression / improvement / unchanged.
- Aggregate delta with 95% bootstrap confidence interval.

**E2. As a reviewer, I get a clear verdict: PASS, WARN, or FAIL.**
- FAIL if the CI upper bound of score delta < -threshold, or any `critical` task regresses.
- WARN if cost/task rises > 30% or p95 latency rises > 50%, or if flaky tasks increase.
- Thresholds are configurable.

### Epic F: CI integration
**F1. As a developer, the CLI exits non-zero on FAIL.**
**F2. As a reviewer, a GitHub Action posts a PR comment** containing score, delta, regressions, top failure categories, cost, latency, and links to the report and traces.
**F3. Smoke suite on PRs, full suite nightly.**

### Epic G: Observability and dashboard
**G1. Step-level traces** (LLM calls, tool calls, latency, tokens) viewable per task result.
**G2. Dashboard pages:** runs list, run detail, compare view, trends, failure breakdown, flaky tasks, judge reliability.
**G3. Public demo mode** serves pre-seeded read-only data without credentials.

### Epic H: Operations
**H1.** Health endpoint, structured logs, spend alerts.
**H2.** Runbook for common failures.

---

## 5. Functional Requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-1 | Agent adapter interface with LangGraph and HTTP implementations | Must |
| FR-2 | YAML dataset schema, validation, versioning | Must |
| FR-3 | Async runner with concurrency, timeouts, retries, repeats | Must |
| FR-4 | Cost tracking per call, per task, per run; hard budget cap | Must |
| FR-5 | Deterministic evaluator library | Must |
| FR-6 | LLM judge with structured output and rubric | Must |
| FR-7 | Judge calibration with kappa reporting | Must |
| FR-8 | Persistent storage of runs, results, scores | Must |
| FR-9 | Regression engine with bootstrap CI | Must |
| FR-10 | Failure categorization | Must |
| FR-11 | CLI (`arl run`, `arl compare`, `arl calibrate`, `arl validate`) | Must |
| FR-12 | GitHub Action with PR comment | Must |
| FR-13 | OpenTelemetry tracing exported to Langfuse | Must |
| FR-14 | REST API for runs, comparisons, datasets | Must |
| FR-15 | React dashboard | Must |
| FR-16 | LLM response caching for CI | Should |
| FR-17 | Flaky-task detection and reporting | Should |
| FR-18 | Fault-injection suite proving detection capability | Should |
| FR-19 | Pluggable evaluator interface (custom evaluators) | Could |
| FR-20 | Support for a second framework adapter (CrewAI) | Could |

---

## 6. Non-Functional Requirements

| ID | Category | Requirement |
|---|---|---|
| NFR-1 | Performance | A 100-task suite with 3 repeats completes in <= 5 minutes at concurrency 10 (excluding agent slowness above 10s/task). |
| NFR-2 | Performance | Dashboard pages load in < 2s for runs with up to 1,000 task results. |
| NFR-3 | Reliability | Runner survives provider outages via retries; no data loss on crash (results persisted incrementally). |
| NFR-4 | Reproducibility | Two runs with identical config and cached responses produce identical scores. |
| NFR-5 | Cost | Smoke suite on a PR costs <= $0.25; nightly full suite <= $3. |
| NFR-6 | Security | See `security.md`; no secrets in repo, API key auth, sandboxed agent execution. |
| NFR-7 | Observability | Structured JSON logs, request IDs, trace IDs correlated across API, runner, and agent. |
| NFR-8 | Maintainability | >= 80% test coverage on core logic; typed Python (mypy strict on core); documented ADRs. |
| NFR-9 | Portability | `docker compose up` starts the full stack locally. |
| NFR-10 | Accessibility | Dashboard meets WCAG AA contrast and keyboard navigation basics. |

---

## 7. Success Metrics

### 7.1 Product quality metrics (published in README)
| Metric | Target |
|---|---|
| Fault-injection detection rate | >= 80% of injected faults detected (misses documented honestly) |
| Judge vs human agreement (Cohen's kappa) | >= 0.6 (target 0.7+) |
| False-positive gate rate on no-change reruns | < 5% |
| Suite wall-clock time (100 tasks x 3) | <= 5 min |
| Cost per 100-task run | Reported, with trend |

### 7.2 Portfolio outcome metrics
- Live demo URL reachable and seeded.
- Public trace/dashboard link.
- 3 documented real incidents in the failure analysis doc.
- 2-minute demo video and one blog post published.

---

## 8. Scope and Release Plan

| Milestone | Window | Contents |
|---|---|---|
| M0 Foundation | Week 0 | PRD, design, architecture, security docs; repo; ADRs; subject agent |
| M1 Core engine | Week 1 | Adapter, runner, evaluators, storage, API |
| M2 Intelligence + gate | Week 2 | Judge calibration, regression engine, CLI, GitHub Action, dashboard v1, tests |
| M3 Production | Week 3 | Deployment, monitoring, runbook, public demo seed |
| M4 Polish | Week 4 | Failure analysis, README numbers, video, blog |

---

## 9. Assumptions
- Users bring their own LLM API keys; the lab never stores them in plaintext outside the secret manager.
- Task suites are small to medium (<= 500 tasks).
- Single-tenant deployment is sufficient for v1.

## 10. Dependencies
- LLM provider APIs (OpenAI/Anthropic) for agent and judge.
- Langfuse (cloud or self-hosted) for trace viewing.
- Postgres (Neon/Supabase), container host (Fly.io/Railway/Cloud Run), Vercel for dashboard.
- GitHub Actions for CI.

## 11. Risks and Mitigations
| Risk | Impact | Mitigation |
|---|---|---|
| LLM judge is biased or inconsistent | Wrong verdicts | Calibration set, kappa tracking, deterministic checks as the primary signal, rubric versioning |
| Flaky results cause false gate failures | Developer distrust | Repeats, bootstrap CI, flaky-task quarantine, no-change rerun test |
| API cost overruns | Budget loss | Hard cap, response cache, cheap model for smoke suite, spend alerts |
| Provider outages/rate limits | Failed CI | Retry/backoff, infra_error category excluded from scoring, fallback judge model |
| Scope creep | Missed timeline | Strict v1 non-goals, milestone gating |
| Agent under test executes unsafe actions | Security incident | Mocked tool backends, sandboxing, network egress limits (see `security.md`) |

## 12. Open Questions
1. Should dataset storage use Git-only or DVC from day one?
2. Which judge model is default, and is a second model used for cross-checking?
3. Is pass@k or mean pass-rate the primary headline metric?
4. Should critical tasks require 100% of repeats to pass?

## 13. Glossary
- **Agent version:** Content-hashed combination of prompt, model, tools, and config.
- **Task:** One test case with input and expectations.
- **Run:** One execution of a dataset version against an agent version.
- **Baseline:** The reference run (usually from `main`) a candidate is compared against.
- **Regression:** A statistically or critically significant quality drop.
- **Flaky task:** A task whose per-run pass rate is between 0.2 and 0.8.
- **Judge calibration:** Measuring how well the LLM judge agrees with human labels.
