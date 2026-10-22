# Design Document

**Project:** Agent Reliability Lab (ARL)
**Version:** 1.0
**Related:** `prd.md`, `architecture.md`, `security.md`

This document describes the detailed design: module responsibilities, interfaces, data models, algorithms, API, CLI, configuration, and UI. `architecture.md` covers how the pieces are deployed and connected.

---

## 1. Design Principles

1. **Evaluation rigor over features.** Correctness of scores matters more than UI polish.
2. **Separate infra failure from agent failure.** Only agent behavior is scored.
3. **Never trust a single run.** Repeats and statistics are first-class.
4. **Everything reproducible.** Every run stores the inputs needed to recreate it.
5. **Pluggable by contract.** Agents, evaluators, and stores sit behind small interfaces.
6. **Fail loud on spend.** Budget caps are enforced before, not after, the money is gone.

---

## 2. Module Overview

| Module | Path | Responsibility |
|---|---|---|
| Subject agent | `subject_agent/` | Demo support agent used as the system under test |
| Adapters | `apps/runner/adapters/` | Translate between the lab and any agent |
| Dataset loader | `apps/runner/datasets/` | Parse, validate, version tasks |
| Runner | `apps/runner/engine/` | Concurrency, timeouts, retries, repeats, budget |
| Evaluators | `evaluators/` | Deterministic checks and LLM judge |
| Calibration | `evaluators/calibration/` | Judge vs human agreement |
| Regression engine | `apps/runner/regression/` | Paired comparison, bootstrap CI, verdict |
| Storage | `apps/api/db/` | Postgres models, migrations, repositories |
| API | `apps/api/` | REST interface for CLI, CI, dashboard |
| CLI | `apps/runner/cli/` | `arl` command |
| Dashboard | `apps/dashboard/` | React UI |
| CI action | `.github/workflows/`, `action/` | PR gate and comment |

---

## 3. Core Interfaces

### 3.1 Agent adapter

```python
class RunConfig(BaseModel):
    model: str
    temperature: float = 0.0
    max_steps: int = 10
    timeout_s: int = 60
    seed: int | None = None
    extra: dict = {}

class ToolCall(BaseModel):
    name: str
    arguments: dict
    result: Any | None = None
    error: str | None = None
    latency_ms: int

class AgentResult(BaseModel):
    output: str
    tool_calls: list[ToolCall]
    steps: int
    tokens_in: int
    tokens_out: int
    cost_usd: float
    latency_ms: int
    trace_id: str | None
    error: str | None = None
    error_kind: Literal["infra", "agent", None] = None

class AgentAdapter(Protocol):
    name: str
    async def run(self, task_input: str, config: RunConfig) -> AgentResult: ...
    def fingerprint(self) -> str: ...   # hash of prompt + model + tools + config
```

- `LangGraphAdapter` imports the graph, invokes it, and extracts tool calls from graph state and callbacks.
- `HttpAdapter` POSTs `{"task_input": "...", "config": {...}}` and validates the response against `AgentResult`. A schema mismatch yields `error_kind="agent"`, category `format_error`.

### 3.2 Evaluator

```python
class EvalOutcome(BaseModel):
    evaluator: str
    version: str
    passed: bool
    score: float | None
    category: FailureCategory | None
    reasoning: str | None

class Evaluator(Protocol):
    name: str
    version: str
    async def evaluate(self, task: Task, result: AgentResult) -> EvalOutcome: ...
```

### 3.3 Store

Repository pattern (`RunRepo`, `ResultRepo`, `DatasetRepo`) so the runner never writes raw SQL and tests can use an in-memory fake.

---

## 4. Data Models

### 4.1 Task schema (YAML)

```yaml
id: refund-001
input: "Hi, order #123 arrived broken. I want my money back."
critical: false
max_steps: 8
tags: [refunds, multi-step]
expected:
  tool_calls: [lookup_order, issue_refund]      # required set
  tool_order: strict | loose | none             # default: loose
  forbidden_tools: [send_email]                 # optional
  must_contain: ["refund"]
  must_not_contain: ["I cannot help"]
  max_side_effects:
    issue_refund: 1                             # catches double refunds
  judge_rubric: >
    Agent confirms refund amount and timeline. Polite, concise.
```

Validation rules: unique `id`; `expected` has at least one check; rubric required if the task uses the judge; `max_steps` between 1 and 50.

### 4.2 Database schema

```
agents(id, name, adapter_type, created_at)
agent_versions(id, agent_id, fingerprint, git_sha, model, temperature,
               prompt_hash, config_json, created_at)
datasets(id, name, created_at)
dataset_versions(id, dataset_id, version_tag, content_hash, task_count)
tasks(id, dataset_version_id, task_key, spec_json, critical, tags[])
runs(id, agent_version_id, dataset_version_id, baseline_run_id NULL,
     status, repeats, concurrency, budget_usd, total_cost_usd,
     trigger (pr|nightly|manual), pr_number NULL,
     started_at, finished_at, env_json)
task_results(id, run_id, task_id, repeat_idx, output, tool_calls_json,
             steps, tokens_in, tokens_out, cost_usd, latency_ms,
             status, error_kind, trace_id, passed)
eval_scores(id, task_result_id, evaluator, evaluator_version,
            passed, score, category, reasoning)
comparisons(id, baseline_run_id, candidate_run_id, verdict,
            score_delta, ci_low, ci_high, cost_delta_pct,
            p95_latency_delta_pct, summary_json)
judge_calibrations(id, judge_model, judge_prompt_hash, n_items,
                   agreement, kappa, created_at)
calibration_items(id, task_key, agent_output, human_label, judge_label)
api_keys(id, name, key_hash, role, created_at, last_used_at, revoked_at)
```

Indexes: `task_results(run_id, task_id)`, `runs(agent_version_id, started_at DESC)`, `comparisons(candidate_run_id)`.

Run status values: `queued, running, completed, failed, aborted_budget, cancelled`.
Task result status values: `ok, timeout, infra_error, agent_error`.

---

## 5. Runner Design

### 5.1 Execution flow

```
load dataset@version -> validate
create run row (status=queued)
build work items: tasks x repeats
spawn N async workers (semaphore = concurrency)
 for each work item:
   budget check (reserve estimated cost)
   run adapter.run() under asyncio.timeout(timeout_s)
   classify outcome (ok | timeout | infra | agent)
   if infra error: retry with exponential backoff + jitter (max 3)
   persist task_result immediately
   run evaluators -> persist eval_scores
finalize run, compute aggregates, set status
```

### 5.2 Retry policy

| Condition | Retry? | Counted in score? |
|---|---|---|
| HTTP 5xx, connection reset, provider 429 | Yes, up to 3 with backoff | Only if still failing, as `infra_error` (excluded from pass-rate denominator) |
| Timeout | No | Yes, as failure with category `timeout` |
| Wrong answer / failed check | No | Yes |
| Schema mismatch | No | Yes, `format_error` |

Excluded `infra_error` results are reported separately; if more than 10% of results are infra errors the run is marked `failed` (not trustworthy).

### 5.3 Budget enforcement

- Estimated cost per item = rolling mean of completed items (initial default per model).
- Before dispatching an item: `spent + in_flight_estimate + estimate <= budget`, else stop dispatching.
- On breach: status `aborted_budget`, in-flight items finish, partial results are kept and flagged so comparisons refuse to treat them as complete.

### 5.4 Caching

Key: `sha256(model, temperature, seed, prompt_hash, task_input, tool_schema_hash)`. Cached only in CI and dev (flag `--cache`). Cached results carry `cached=true` and zero cost, but the original cost is stored so reports remain meaningful.

---

## 6. Evaluator Design

### 6.1 Deterministic evaluators

| Evaluator | Logic | Category on failure |
|---|---|---|
| `ToolCallMatch` | Required tools present; optional order check | `missing_tool` |
| `ForbiddenTool` | No tool from forbidden list | `wrong_tool` |
| `SideEffectLimit` | Count of calls per side-effect tool <= limit | `wrong_tool` |
| `ContainsCheck` | Required substrings present | `other` |
| `ForbiddenPhrase` | None of the forbidden phrases appear | `other` |
| `SchemaCheck` | Output parses against JSON schema if specified | `format_error` |
| `StepLimit` | `steps <= max_steps` | `loop` |

### 6.2 LLM judge

**Input to judge:** task input, rubric, agent output, a compact tool-call summary. The agent output is wrapped in clearly delimited tags and the judge system prompt states that content inside is untrusted data (see `security.md`, judge injection).

**Output (enforced by structured output / JSON schema):**
```json
{
  "score": 1-5,
  "pass": true,
  "reasoning": "string, max 400 chars",
  "failure_category": "hallucination | tone_issue | wrong_tool | other | null"
}
```

**Settings:** temperature 0, fixed model snapshot, prompt hashed and stored. Pass threshold: `score >= 4`.

**Bias controls:** judge does not see which agent version produced the output; rubric requires evidence from the output; a length-neutral instruction is included; calibration set includes verbose-wrong and terse-correct cases.

### 6.3 Final pass logic

```
passed = all(deterministic.passed) AND judge.passed   # if rubric defined
failure_category = first failing deterministic category, else judge category
```

### 6.4 Judge calibration

1. Export ~50-100 results to `calibration.csv` (balanced across pass/fail and tags).
2. Human fills `human_label` (pass/fail).
3. `arl calibrate` runs the judge, computes:
   - raw agreement = matches / n
   - Cohen's kappa = (p_o - p_e) / (1 - p_e)
   - confusion matrix (false-pass vs false-fail rates)
4. Stored in `judge_calibrations`; dashboard shows trend; weekly job alerts if kappa < 0.6.

---

## 7. Regression Engine Design

### 7.1 Inputs
Baseline run B and candidate run C on the same dataset version (enforced; mismatch returns an error).

### 7.2 Per-task scores
`p_task = passes / valid_repeats` (infra errors excluded). Tasks with zero valid repeats in either run are listed as `unscored`.

### 7.3 Per-task classification
Using a paired difference `d = p_C - p_B`:
- `regression` if `d <= -0.4` (configurable) or Fisher exact test p < 0.1 for that task's repeat counts
- `improvement` if `d >= +0.4`
- `unchanged` otherwise

### 7.4 Aggregate delta with bootstrap
```python
def bootstrap_delta(diffs: np.ndarray, n: int = 10_000, seed: int = 0):
    rng = np.random.default_rng(seed)
    idx = rng.integers(0, len(diffs), size=(n, len(diffs)))
    means = diffs[idx].mean(axis=1)
    return diffs.mean(), np.percentile(means, [2.5, 97.5])
```
Seeded so the verdict is deterministic for the same inputs.

### 7.5 Verdict rules (configurable in `arl.yaml`)

```
FAIL if ci_high < -min_effect            (default min_effect = 0.02)
FAIL if any task with critical=true is a regression
WARN if cost_per_task_delta  > +30%
WARN if p95_latency_delta    > +50%
WARN if flaky_task_count increased by > 3
PASS otherwise
```

### 7.6 Output object
```json
{
  "verdict": "FAIL",
  "score_baseline": 0.870, "score_candidate": 0.830,
  "delta": -0.040, "ci": [-0.061, -0.018],
  "regressions": [{"task": "refund-014", "from": 1.0, "to": 0.2}],
  "improvements": [],
  "failure_category_shift": {"wrong_tool": "+8"},
  "cost_per_task": {"baseline": 0.030, "candidate": 0.031},
  "p95_latency_ms": {"baseline": 5700, "candidate": 6200}
}
```

### 7.7 Flaky detection
A task is flaky if its pass-rate across repeats is in [0.2, 0.8] in either run. Flaky tasks are shown in a dedicated view and can be quarantined (excluded from gating, still reported) via a `quarantine` list in config.

---

## 8. API Design (REST, FastAPI)

Base path `/api/v1`. All write endpoints require `Authorization: Bearer <api_key>`.

| Method | Path | Description | Auth |
|---|---|---|---|
| GET | `/health` | Liveness and DB check | none |
| POST | `/agents` | Register agent config | admin |
| GET | `/agents/{id}/versions` | List versions | viewer |
| POST | `/datasets` | Register dataset version | admin |
| GET | `/datasets/{name}/versions` | List versions | viewer |
| POST | `/runs` | Start a run (body: agent_version, dataset_version, repeats, budget) | admin |
| GET | `/runs` | List runs (filters: agent, trigger, date) | viewer |
| GET | `/runs/{id}` | Run detail and aggregates | viewer |
| GET | `/runs/{id}/results` | Paginated task results | viewer |
| POST | `/runs/{id}/cancel` | Cancel | admin |
| POST | `/comparisons` | Compare two runs | admin |
| GET | `/comparisons/{id}` | Comparison with verdict | viewer |
| GET | `/trends` | Time series of score, cost, latency | viewer |
| GET | `/judge/calibrations` | Calibration history | viewer |
| POST | `/judge/calibrations` | Trigger calibration | admin |

Conventions: JSON, ISO-8601 timestamps, cursor pagination (`?cursor=&limit=`), RFC 7807 problem details for errors, `X-Request-ID` echoed on every response.

---

## 9. CLI Design

```
arl validate  --dataset <path|name@ver>
arl run       --agent <config.yaml> --dataset <name@ver>
              [--repeats 3] [--concurrency 10] [--budget 2.0]
              [--baseline main|<run_id>] [--cache] [--suite smoke|full]
arl compare   --baseline <run_id> --candidate <run_id> [--format md|json]
arl calibrate --judge <model> --labels calibration.csv
arl report    --run <run_id> --format md
arl faults    --suite fault_injection.yaml     # runs the fault-injection table
```

Exit codes: `0` PASS or WARN, `1` FAIL, `2` invalid input/config, `3` run untrustworthy (too many infra errors or budget abort), `4` internal error.

### Agent config (`arl.yaml`)
```yaml
agent:
  name: support-agent
  adapter: langgraph          # or http
  entrypoint: subject_agent.graph:build   # or url: https://...
  model: gpt-4o-mini
  temperature: 0
  prompt_file: subject_agent/prompts/system.md
gate:
  min_effect: 0.02
  critical_must_pass: true
  cost_warn_pct: 30
  latency_warn_pct: 50
quarantine: []
```

---

## 10. GitHub Action Design

```
on: pull_request
steps:
  checkout -> setup python -> install
  arl validate
  arl run --suite smoke --baseline main --cache
  arl compare --format md > report.md
  post/update sticky PR comment from report.md
  exit with arl exit code
on: schedule (nightly)
  arl run --suite full --repeats 5 -> store, update trends
```

PR comment template: verdict badge, score and delta with CI, regression list (top 5), failure-category shifts, cost and p95 latency delta, links to dashboard compare view and Langfuse traces.

---

## 11. Dashboard Design

**Stack:** React, TypeScript, Tailwind, TanStack Query, Recharts. Hosted on Vercel.

| Page | Contents |
|---|---|
| Runs | Table: run, agent version, dataset, score, cost, duration, verdict badge; filters |
| Run detail | Aggregates; task table with pass-rate, flaky flag, failure category; row opens task drawer |
| Task drawer | Input, output, tool calls, evaluator results with reasoning, "Open trace" link |
| Compare | Side-by-side score, delta + CI chart, regression/improvement lists, category shift chart |
| Trends | Line charts: pass rate, cost/task, p95 latency; filter by agent and date |
| Failure breakdown | Stacked bars by category over runs |
| Flaky tasks | Ranked by instability |
| Judge reliability | Kappa trend, confusion matrix, calibration set size |
| Demo banner | Shown in public mode: "Read-only demo data" |

UX rules: verdict color plus text/icon (never color alone), empty and error states for every page, skeleton loaders, URL-encoded filters so views are shareable.

---

## 12. Subject Agent Design

A LangGraph customer-support agent with mocked backends.

- **Tools:** `lookup_order`, `issue_refund`, `send_email`, `search_policy`
- **Mock backend:** in-memory orders and policy docs; no network side effects
- **Deliberate weaknesses** (so the lab has real findings): ambiguous refund policy, optional verification step, long-context confusion
- **Fault-injection variants:** flags in config to remove a tool, swap model, shorten step limit, inject a "be brief" prompt, add a subtle tone change

### Fault-injection table (reported in README)
| Fault | Expected category | Detected? | Time to detect |
|---|---|---|---|
| Remove `lookup_order` | missing_tool | |  |
| Weaker model | hallucination | |  |
| "Be extremely brief" prompt | wrong_tool | |  |
| Step limit = 2 | loop | |  |
| Subtle tone change | tone_issue | |  |

---

## 13. Error Handling Strategy

| Layer | Strategy |
|---|---|
| Adapter | Never raises to the runner; returns `AgentResult` with `error_kind` |
| Runner | Isolates each work item; one failure never stops the suite |
| Evaluators | Evaluator exceptions recorded as `evaluator_error` and surfaced, never silently passed |
| API | Problem-details responses, no stack traces to clients |
| CLI | Human-readable message plus exit code table |

## 14. Testing Strategy (design-level)

| Level | What | Tooling |
|---|---|---|
| Unit | Evaluators, bootstrap, budget logic, schema validation | pytest |
| Property | Bootstrap is seed-deterministic; comparison is symmetric | hypothesis |
| Integration | Full run against a stub agent with known outcomes, real Postgres | pytest + testcontainers |
| Fault injection | Break subject agent and assert detection | `arl faults` |
| Judge tests | Verbose-wrong, terse-correct, injection-attempt outputs | pytest fixtures |
| No-change rerun | Same agent twice must yield PASS (false-positive rate) | CI nightly |
| UI | Component tests and one Playwright smoke path | Vitest, Playwright |

## 15. Design Decisions (ADR index)

| ADR | Topic |
|---|---|
| 0002 | Postgres over SQLite |
| 0003 | OpenTelemetry with Langfuse as viewer |
| 0004 | Hybrid deterministic + LLM judge |
| 0005 | Bootstrap CI over raw score thresholds |
| 0006 | Exclude infra errors from scoring |
| 0007 | Datasets as YAML in Git (+ DVC option) |
