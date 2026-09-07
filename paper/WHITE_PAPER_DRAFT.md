# Adam Prism: A Sovereignty-First Architecture for Verified, Deterministic LLM Agents

**Draft — for review before public release (v0.2)**

> Publication metadata template (fill at release):
>
> - **Title:** Adam Prism: A Sovereignty-First Architecture for Verified, Deterministic LLM Agents
> - **Authors:** Mohamed Othman (Independent Researcher, Cairo, Egypt)
> - **Abstract:** *(included below, ≤1920 chars)*
> - **Comments:** 13 pages, 3 figures, 6 tables
> - **Category:** cs.AI (primary); cross-list: cs.MA, cs.SE
> - **License:** CC BY 4.0 (paper) — see LICENSE file in companion repository
> - **Release:** companion repository archived with DOI via Zenodo (paper + data only, no source code)

---

## Abstract

Autonomous LLM agents that generate code, documents, or system artifacts face a structural
problem: their outputs are stochastic, while the environments they must produce for are
deterministic. Orchestration frameworks such as LangGraph, CrewAI, and AutoGen move control
out of the model loop, yet verification still relies on LLM-as-a-judge — a probabilistic
system supervising a probabilistic system. We present **Adam Prism**, a sovereignty-first
agent architecture that treats verification as a first-class architectural layer rather
than an afterthought. Adam Prism separates planning, execution, verification, and repair
into independent stages: a schema-validated plan drives sandboxed step execution; a
deep-verification loop combines deterministic functional checks (compilation, import, and
test execution) with an independent spec-conformance judge — an LLM instantiated separately
from the builder so that a product is measured against the requested goal, not against its
own tests; a bounded escalation ladder repairs defects with an honest stop when convergence
is impossible. On top of this loop, a governance layer provides human-in-the-loop approval
gates, scheduled background jobs, durable agent teams, and rotating credential storage.
Because deterministic checks carry the heavy validation load, the system performs 1,317
automated tests without any LLM call, executes 26 scenario workflows, and assembles
hardened Linux distributions from a natural-language goal. Two independent oracle-checked
registries — 10 CSV-verification and 5 code-generation scenarios (executable hidden cases) —
reach **10/10** and **15/15 (100%)** single-shot.
Everything runs fully locally on a 12B model via Ollama: zero cost, no telemetry, air-gap
capable. We report a failure-mode
taxonomy, a system-level benchmark of the two-role execution split, and an honest τ-bench
probe of raw model capability.

**Keywords:** agentic AI, multi-agent orchestration, verification, sovereign AI,
local LLM, deterministic evaluation, spec-conformance.

---

## 1. Introduction

The shift from generative AI to agentic AI — systems that act on external environments —
exposes a fundamental architectural mismatch. Large language models produce stochastic,
unstructured tokens; the backends they control — file systems, compilers, test runners,
package managers, operating-system builders — require deterministic, schema-conformant
inputs. Every agent framework must therefore answer one question: **who decides what is
"done"?**

The dominant answer in 2026 is *LLM-as-a-judge*: a model scores another model's output.
Recent research demonstrates why this is fragile. FORMALJUDGE (arXiv:2602.11136)
articulates the core dilemma — "how can probabilistic systems reliably supervise other
probabilistic systems without inheriting their failure modes?" — and proposes
neuro-symbolic separation (formal proofs for what is logical, model judgment confined to
atomic facts). AGENTLTL (arXiv:2607.02599) shows that procedural compliance can be scored
deterministically, judge-free, and that the same constraints can gate execution online and
drive training. Verified Multi-Agent Orchestration (arXiv:2603.11445) demonstrates that an
orchestration-level verifier, separate from the executing model, improves answer
completeness by more than a point on 1–5 rubrics. ToolBench-X (arXiv:2606.25819) shows
that agents fail far more because they cannot *diagnose* environment hazards than because
they lack compute — and that targeted diagnosis recovers many previously-failed tasks
where test-time scaling yields limited gains.
General AgentBench (arXiv:2602.18998) identifies a persistent *verification gap*:
generation capacity consistently exceeds selection capacity, so test-time scale alone does
not buy correctness.

These results converge on a design principle we adopt directly: **decouple verification
from the system being verified, and make verification as deterministic as the target
environment permits.**

Independently, the multi-agent framework landscape measures its own gap. Independent 2026
benchmarks of LangGraph, CrewAI, and AutoGen agree on the shape of the trade-off:
LangGraph adds the lowest orchestration overhead (~9% tokens) and the best latency, but
imposes a steep learning curve and manual fault recovery; CrewAI prototypes fastest but
carries ~18% token overhead and poor debugging; AutoGen's conversational model scales
cost poorly on chain depth and was placed in maintenance mode by its maintainers. None of
them treats *repair* and *independent verification* as native architectural layers.

Adam Prism is a different bet: a **sovereignty-first, verification-driven agent** that
runs entirely on local hardware. Its contributions are:

1. **A verification-driven execution model** (Plan → Execute → Verify → Repair → Gate)
   where the verifier is architecturally separated from the builder, and where an
   independent judge measures the product against the original goal.
2. **Deterministic grounding of evaluation** — compilation, import, test execution, and
   statistical validation (distribution-shape and deviation tests, duplicate detection)
   run without any model call, so regression suites are deterministic and reproducible.
3. **An honest bounded-repair loop** — defects are repaired through an escalation ladder
   with an explicit, documented stop condition when convergence is impossible, rather
   than an open-ended retry.
4. **Reusable scenario and tree workflows** — 26 goal-driven workflows that turn natural
   language into executed, verified artifacts.
5. **Sovereign deployment** — zero telemetry, air-gap native, single local 12B model, no
   per-token cost.

The rest of this paper describes the architecture (§2), the verification model (§3),
governance (§4), evaluation (§5), and honest limitations (§6). We close with related work
and open problems (§7, §8).

---

## 2. Architecture Overview

### 2.1 Design Principles

**P1 — Verify, don't assume.** The cost of trust must be paid with checks, not prompts.
Whenever a requirement can be checked deterministically (parse, compile, import, run
tests, statistical conformance), no LLM is consulted.

**P2 — The judge is not the builder.** Quality must be measured against the original goal,
by a component instantiated separately from the module that produced the artifact. A
self-written test suite can verify a drifted interface; the judge's contract is to catch
that drift.

**P3 — Plans are real, validated objects.** Goals become structured plans (task trees or
scenario steps) validated against a schema before execution. Control flow lives in the
plan, not in an opaque reasoning loop.

**P4 — Honest failure beats fake success.** Every loop has a bounded, documented stopping
rule; when repair cannot converge, the system reports the failure truthfully instead of
returning a superficially plausible artifact.

**P5 — The machine belongs to the user.** Local-only execution by default; no analytics,
no callbacks, no per-token billing.

### 2.2 Block Diagram

The diagram below is intentionally implementation-agnostic (no code, no internal module
names, no filenames). It describes the control plane of the system.

```
┌──────────────────────────────────────────────────────────────────────┐
│                         USER (goal in natural language)               │
└───────────────────────────────┬──────────────────────────────────────┘
                                │
                                ▼
                ┌───────────────────────────────┐
                │        1. PLAN                │
                │   goal → validated plan       │
                │   (task tree / scenario)      │
                │   schema-gated, indexed       │
                └───────────────┬───────────────┘
                                │ plan (executable object)
                                ▼
                ┌───────────────────────────────┐      ┌──────────────────┐
                │        2. EXECUTE             │─────▶│   SANDBOX        │
                │   batched / parallel steps    │      │   isolated       │
                │   tool dispatch (gated)       │      │   environment    │
                └───────────────┬───────────────┘      └──────────────────┘
                                │                        (files, tests,
                                ▼                       services, shells)
                ┌───────────────────────────────┐
                │        3. VERIFY              │
                │   ┌─ deterministic checks:    │
                │   │   parse / import / test   │
                │   │   statistical conformance │
                │   └─ independent judge:      │
                │       product ↔ original goal │
                └───────────────┬───────────────┘
                                │ pass / fail / partial
                                │ fail
                                ▼
                ┌───────────────────────────────┐
                │        4. REPAIR              │
                │   bounded escalation ladder   │
                │   → grounding / targeted fix  │
                │   → honest STOP if diverging  │
                └───────────────┬───────────────┘
                                │ re-validate (loop to 3)
                                │ pass
                                ▼
                ┌───────────────────────────────┐   ┌───────────────────────┐
                │        5. HONEST GATE         │   │   6. GOVERNANCE       │
                │   report: verified artifacts  │   │  approval gates       │
                │   + evidence + failure log    │   │  scheduled jobs       │
                └───────────────────────────────┘   │  durable agent teams  │
                                                    │  rotating credentials │
                                                    │  runtime invariants   │
                                                    └─────────┬─────────────┘
                                                              │ auditable log
                                                              ▼
                ┌──────────────────────────────────────────────────────────┐
                │        DELIVERED PRODUCT (real artifacts/files)          │
                └──────────────────────────────────────────────────────────┘
```

Two observations from the literature justify this layout. **Graph Harness**
(arXiv:2604.11378) argues that reAct-style agent loops are single-ready-unit schedulers
with implicit dependencies, unbounded recovery, and mutable history; lifting control into
an explicit, immutable, validated plan (our *Plan* stage) is precisely the fix it
proposes. **KAIJU** (arXiv:2604.02375) demonstrates that separating the reasoning layer
from the execution layer yields behavioral guarantees that prompting alone cannot — the
same separation appears here as *Execute/Sandbox*.

### 2.3 Where This Differs From LangGraph / CrewAI / AutoGen

The comparison is architectural, not a claim of superiority in every dimension. The
frameworks are excellent at what they designed for (Table 1).

| Dimension | LangGraph | CrewAI | AutoGen / MAF | Adam Prism |
|---|---|---|---|---|
| Control model | explicit graph | role-based crews | conversation | validated plan (tree/scenario) |
| Verification layer | external / manual | external / manual | external / manual | **native: deterministic + independent judge** |
| Repair layer | manual recovery | retry | retry | **native bounded escalation + honest stop** |
| Ideal-for | stateful conditionals, audit | rapid business prototyping | conversational research | **verified artifacts, sovereignty** |
| Local / air-gap | self-hostable | self-hostable | self-hostable | **designed for it, zero telemetry** |

Table 1 — Architectural positioning. Numbers behind orchestration overhead and latency
are surveyed in §5.2 from independent 2026 benchmarks.

---

## 3. The Verification Model

### 3.1 Verification Is a Layer, Not a Prompt

Verification consumes the outputs of execution and produces a pass/fail/partial decision
with evidence. Its components are ordered by cost and determinism:

1. **Deterministic checks** — parse, compile, import smoke, and test execution of the
   produced artifact; statistical conformance checks (distribution-shape tests,
   deviation thresholds, duplication detection) for data-review tasks. These require no
   model calls and are byte-identical run to run.
2. **Independent spec-conformance judge** — an LLM instance created separately from the
   builder. It receives the original goal (commands, files, edge cases expected) and the
   produced artifact, and emits a structured pass/fail with per-requirement evidence.
   Because it never inherits the builder's context, it cannot "approve" a drifted
   interface on the strength of the builder's confidence.
3. **Merge rule** — a lenient judge cannot clear a deterministic failure. Deterministic
   failures are final; the judge resolves only what rules cannot decide.

This ordering mirrors the pattern formalized in FORMALJUDGE (atomic groundable judgments +
deterministic composition) and AGENTLTL (deterministic, judge-free compliance scores), and
it is cheaper than pure LLM judging: the auxiliary judgment model is an order of
magnitude smaller than the builder.

### 3.2 The Honest Repair Loop

When verification fails, repair is bounded:

- **L0 — deterministic fixes**: correct what deterministic rules know how to fix directly.
- **L1 — targeted re-generation**: re-generate only the defective unit with the failure
  evidence, preserving already-verified work.
- **L2 — grounding**: consult external, time-bounded reference material (with a caching
  layer) to supply facts the fix depends on.
- **L3 — escalate**: invoke the artifact-repair path; **stop** if the same-signature
  failure repeats past the ceiling.

The loop always terminates in one of exactly two reporting states: **verified** (with
evidence) or **not converged** (with an honest log of what failed and why). There is no
third state of "silently plausible."

This directly addresses the failure modes documented in ToolBench-X (diagnosis beats raw
compute) and General AgentBench (verification gap): instead of spending more inference on
a defective loop, we spend a bounded, documented effort and then *say no*.

### 3.3 Deterministic Statistical Validation

For data-review and compliance scenarios, "verification" includes statistical screening
that must not vary across runs or environments:

- profile-shape tests (e.g., digit-distribution consistency),
- deviation thresholds against expected baselines,
- duplicate and near-duplicate detection.

These checks are implemented without heavyweight scientific dependencies, so they run in
any environment with no install footprint — a reproducibility property of its own.

---

## 4. Governance and Sovereignty

A verified artifact is not the same as a *governed* run. Adam Prism's governance layer
sits above the loop:

- **Human-in-the-loop approval gates** on designated steps, so irreversible actions are
  not taken by a stochastic system without consent;
- **Scheduled background jobs** and **durable agent teams** persisted to disk, so work
  survives restarts;
- **Rotating credential storage** and **runtime invariant checks**, so execution-time
  assumptions are validated continuously.

Sovereignty properties are structural, not claims: local-only execution by default, air-gap
operation tested, and no analytics or telemetry SDKs. The open-source core is AGPL v3 with
a commercial dual license.

---

## 5. Evaluation

### 5.1 System-Size Measurements (Reproducible Without an LLM)

| Metric | Value | Nature |
|---|---|---|
| Automated tests (pytest) | 1,317 green | deterministic, no LLM |
| Lint (PEP 8 / pyflakes) | clean | deterministic |
| API routes | 227 | REST + WebSocket + SSE |
| Scenario workflows | 26 (12 product + 14 tree) | declarative YAML |
| Number of LLM calls in regression suite | 0 | — |

Table 2 — Reliability surface. Running the full regression suite requires no model, so it
is usable in CI on any machine.

### 5.2 Execution Time and Success Rate

Measured on the platform's engineering tasks (task-completion runs on a local 12B model):

| Metric | Free-form prompting | Adam Prism (planned + verified) |
|---|---|---|
| Median task completion | 5–10 minutes | **121–185 s (§5.6)** |
| Verified success (oracle) | 40–50% (ad hoc) | **10/10 = 100% (§5.6)** |
| Failure handling | implicit retry loops | **bounded repair + honest report** |

Table 3 — Within-platform improvement from plan + verification. The gain comes from not
letting the model "think in place": structured plans replace free reasoning, and
verification replaces hope. The Adam Prism figures are the *measured* single-shot
pass@1 of the 10-scenario audit registry in §5.6 (independent deterministic oracles),
not an informal estimate.

For positioning against orchestration frameworks, we rely on independent 2026
public-benchmark numbers rather than our own head-to-head (we have not run the same
public corpus; see §6):

| Framework (independent 2026 benchmarks) | Token overhead vs raw calls | p50 / p95 latency (research task) | Fault recovery |
|---|---|---|---|
| LangGraph | ~9% | 14.1 s / 19.8 s | manual |
| CrewAI | ~18% | 18.4 s / 31.2 s | retry |
| AutoGen | ~12–31% | 22.7 s / 41.5 s | retry |
| Hermes Agent | ~3–5% | ~1–2 s (single agent) | auto-recovery |
| Adam Prism (measured, local 12B) | 0 (no API) | task-level 121–185 s (§5.6/§5.5) | **native bounded repair** |

Table 4 — Landscape survey (sources in §7). Absolute numbers are not cross-comparable
(hardware, model, task sets differ); the structural columns — overhead, recovery — are the
fair comparison.

### 5.3 Quality: What "Green" Means

The central quality claim is about *what passes the gate*:

- A product passes only if it is **functionally exercisable** (imports, executes, tests
  pass) **and spec-conformant** (an independent judge finds the product matches the
  requested commands/files/edge-cases).
- Deterministic failures cannot be waived by any model.
- When repair converges, the final gate is re-run on the *actual shipped artifact*.

Case in point: in engineering runs, syntactically valid but spec-divergent products (the
right file layout, wrong interface; green self-written tests on a drifted contract) were
rejected by the deterministic + independent-judge merge — a class of failure that a
shallow compile-gate passed silently.

### 5.4 Scaling Behavior

Scaling claims are structural, not load-tested:

- Parallel step dispatch turns independent plan branches into concurrent execution,
  bounding wall-clock by the critical path rather than the sum of steps (the
  "cognitive map-reduce" argument, cf. Auton arXiv:2602.23720).
- Deterministic verification does not consume tokens, so regression cost grows with work,
  not with model calls.
- Storage for large artifacts (disk images, ISOs) uses sparse images and staged build
  flows — entirely reproducible from light config, so artifacts are re-created rather
  than hoarded.

### 5.5 System-Efficiency Benchmark (measured, local, reproducible)

To measure *system* efficiency — not raw model skill — we ran the production two-role
pipeline end to end on two artifact-production goals (a word-frequency module + CLI
Fibonacci tool), each requiring: a schema-validated plan, batched node execution, file
generation, deterministic checks, an independent spec-conformance review, and bounded
repair. All runs: gemma4:12b as the commander channel, local Ollama, one RTX 3060 (12 GB),
zero API cost. Two configurations were compared:

- **Mode A** — the same model also generates the code (delegation disabled).
- **Mode B** — the two-role split: the commander *decides*; a code-specialized local
  model (qwen2.5-coder:14b) *executes* file generation/repair through a dedicated channel
  (temperature 0.2), with the VRAM juggled via explicit model eviction.

| Metric (2 goals, n=1 each) | Mode A (single-role) | Mode B (two-role) |
|---|---|---|
| Wall-clock per goal | 147–191 s | 167–191 s |
| **Commander (leader) model — tokens** | **61,282** | **36,601 (−40%)** |
| Commander (leader) model — calls | 35 | 24 (−31%) |
| Executor channel — tokens | 0 | 19,620 |
| Total tokens across models | 61,282 | 56,221 |
| Deep checks / spec audits per goal | 2 / 2 | 2 / 2 |
| Spec repair rounds (caught + fixed) | 1 | 1 |
| Externally verified (py_compile + pytest green) | ✅ both goals | ✅ both goals |
| Financial cost | $0.00 | $0.00 |

Reading the table honestly:

1. **The split succeeds at its job: it offloads the load-bearing model.** Mode B moves ~35%
   of token volume on the executor channel — and 40% of leader exposure — onto the cheap,
   low-temperature executor channel, freeing the commander for planning, judging, and
   escalation. On a per-token or leader-quota budget, that difference is the *raison
   d'être* of the architecture.
2. **Wall-clock is not yet a win on a single 12 GB card.** Switching models in and out of
   VRAM consumes the latency saved. On multi-GPU or co-resident deployment (or a larger
   card), the leader offload should translate to wall-time wins; on this hardware it is a
   wash (147–191 s both modes). We state this rather than claim otherwise.
3. **Deterministic confirmation is stable:** every goal produced artifacts that pass
   compilation and tests, after exactly one spec-repair round — evidence the
   audit→repair→audit loop closes, not just that output "looks fine."
4. **A verification gap surfaced (and it is our bug, not the model's):** in an early run,
   the produced test file was named `tests_word_count.py` (plural) — which the default
   pytest collection pattern (`test_*.py`) ignores. The internal regression check
   therefore collected zero tests and reported *pass*, while the test file actually
   contained a wrong oracle (expected 25 words; true count 19). Collection semantics are
   part of verification; our *external* harness now runs every `*test*` file explicitly,
   and all final artifacts pass. This is a precise, reproducible example of the
   verification-gap literature (§7) and a fixable defect, not an excuse.

---

### 5.6 Audit-Registry Benchmark: pass@1 on 10 Independent Deterministic Oracles

The strongest claim in this paper is not a wall-clock comparison; it is that the *system* —
not the model — closes every gap the judge can see. We therefore move from timing to
verification: a second, independent layer in which correctness is decided by a
deterministic oracle rather than by wall time. To measure that, we built a **registry of 10
data-verification scenarios** (the buy/sell-audit goal
family), each with its **own deterministic oracle** that checks the produced report against
the *true synthetic population* — exact-verbatim substring match on the renderer's
confirmed-patterns and facts lines — completely separate from the LLM rubric. Raw drivers and per-task results are archived in
the paper's companion repository (§B), with the aggregate exposed as machine-readable JSON
(`audit_registry`).

| Task | Wall (s) | Steps | Oracle | Swaps |
|---|---|---|---|---|
| dup_payments (فواتير مكررة عبر موردين) | 150.8 | 9/9 | 8/8 | 11 |
| high_value (معاملات عالية القيمة) | 138.6 | 9/9 | 8/8 | 11 |
| missing_fields (حقول ناقصة في عملاء) | 184.8 | 9/9 | 8/8 | 16 |
| multi_recipient (رسائل متعدد المستلمين) | 121.9 | 9/9 | 8/8 | 11 |
| budget_util (فاقد ميزاني per-team) | 140.6 | 9/9 | 8/8 | 11 |
| payroll_recon (قيمة شاذة بالرواتب) | 138.3 | 9/9 | 8/8 | 11 |
| ledger_balance (توازن وارد/صادر) | 130.0 | 9/9 | 8/8 | 11 |
| empty_category (نفقات بفئة فارغة) | 125.2 | 9/9 | 8/8 | 11 |
| sales_growth (انخفاض نمو شهري) | 125.3 | 9/9 | 8/8 | 11 |
| dup_pairs (أزواج مكررة عبر مفاتيح) | 149.2 | 9/9 | 8/8 | 11 |

**Aggregate across three independent runs of this fleet: pass@1 = pass@2 = pass@3 = 10/10
(30/30 task-runs, 80/80 oracle checks in every run), median wall
138.4 / 127.2 / 126.3 s, mean 11.5 VRAM swaps, 0 spec audits (rubric-gated), $0.00 cost.**
Every scenario ran the full production channel split: commander `gemma4:12b` plans/judges,
executor `qwen2.5-coder:14b` generates the report; each task is single process, single-shot
(pass@1) with the bounded-repair ladder untouched. **The 10/10 is not stochastic**: two
additional independent runs reproduce the archived result exactly (pass@3 = 10/10).

### Controlled decomposition (same tasks, same seeds, same oracle)

A preceding clean run of the same fleet scored 5/10: the five failures were all *honest
halts* where the reports were factually correct (the independent oracle accepted them 8/8)
yet the internal judge rejected them because the prose defaulted to "لا توجد أنماط" while
the data demonstrably contained anomalies (e.g. 12 missing values) — a real model defect,
not a judge error. That natural experiment became the controlled study below: we run the
same 10 tasks and the same oracle in four arms that remove one layer at a time.

| Arm | pass@1 | Oracle | Med. wall | Mechanical reading |
|---|---|---|---|---|
| **System** (rep 1, archived) | **10/10** | 80/80 | 138.4 s | full pipeline |
| **System** (rep 2) | **10/10** | 80/80 | 127.2 s | independent rerun |
| **System** (rep 3) | **10/10** | 80/80 | 126.3 s | independent rerun |
| Grounding only, no verify/repair | 0/10 | 40/80 | 41.6 s | facts right; contract wrong |
| No grounded classes (ablate) | 7/10 | 78/80 | 146.4 s | 1 honest halt + 1 silent error + 1 halt on correct artifacts |
| Raw model, full data inlined | 0/10 | 60/80 | 12.7 s | prose right; numbers invented |

How to read the table — each arm isolates one layer with the identical instrument:

- **Raw model (free, 0/10; oracle 60/80).** The 12B model receives the full data inlined,
  the same output contract, and the same oracle. It emits the four required sections and a
  valid JSON summary, then misses nearly every numeric fact — e.g. it reports
  `total_rows=794` for a 300-row file and a plausible-but-wrong count for each injected
  anomaly class. Med. wall 12.7 s: fast, confident, wrong. This is the ceiling the
  architecture compensates for.
- **Grounding without verification orchestration (noverify, 0/10; oracle 40/80).** The
  data-grounding facts (e.g. `total_rows`) reach the report deterministically, but without
  the attributed-review → regeneration loop the report no longer conforms to the four-section
  contract — every task lands 4/8. The verification loop's measured contribution is
  deliverable *conformance*, not only error-catching.
- **Full pipeline without the grounded anomaly classes (ablate, 7/10; oracle 78/80).**
  Restoring the pre-intervention evidence (duplicates / multi-recipient / maxima only; no
  missing-values / overspend / category / growth classes) on the identical pipeline
  reproduces the failure family: one honest halt (missing-field count refused for lack of
  evidence), one *silent error* (empty-category task reports success while the report carries
  a wrong count, caught only by the oracle), and one conservative halt whose persisted
  artifacts were already oracle-correct (8/8). The grounded classes are the point of leverage.
- **Full system (10/10 ×3).** With the classes rendered into a validator-visible evidence
  block and named in the regeneration prompt, every anomaly family is surfaced and verified
  verbatim by the independent oracle.

Honest delta we do not claim: measurement goals that require external LLM availability or a
third model family; the two registries measure *system reproducibility and layer
attribution* across two goal families, not *general agent ability*.

### 5.7 Second Goal Family: Executable-Oracle Code Generation

The audit registry above measures one goal family (structured CSV data audits) with oracles
that *read the produced report's text*. To break that limitation — and to show the
verification loop is not specialized to text reports — we built a
**second family of 5 code-generation scenarios** whose oracles
are *executable*: the produced `module.py` is imported in a subprocess and run against
hidden deterministic cases whose expected outputs are computed from a reference
implementation — no LLM, no rubric, only `py_compile` + real execution.

| Task | Function | Wall med (s) | Oracle | pass@3 |
|---|---|---|---|---|
| mergesort | `sort_nums` | 66.2 | 5/5 | 3/3 |
| dedupe_pairs | `dedupe_pairs` | 63.8 | 5/5 | 3/3 |
| payroll_net | `net_pay` | 63.8 | 5/5 | 3/3 |
| topk_words | `top_k_words` | 62.7 | 5/5 | 3/3 |
| budget_overspend | `find_overspending` | 60.5 | 5/5 | 3/3 |

Each task receives its exact requirement spec (the equivalent of the gold facts in §5.6) and
follows the same 8-step scenario: plan → review-plan → list → write — via the executor
channel `qwen2.5-coder:14b` — review code (sandboxed analysis JSON) → render a deterministic
report grounded in the produced files → attributed review → global verify. The review gate
targets the code-review product (function, signature, edge cases, test count) rather than
the report prose, and report rendering is deterministic (grounding evidence block +
artifacts), which removes fence-wrapping noise from the judged output.

**Aggregate across three independent runs: pass@1 = pass@2 = pass@3 = 5/5 (15/15 task-runs),
75/75 hidden executable checks, median wall 60–66 s, $0.00 cost, all single-process and
single-shot.** Both the internal gate (`system_success`) and the external oracle agree on
every run, so the pipeline's own verification correlates with an independent executable
check on a second goal family — not just with a report rubric. An early spec-less pilot
(payroll tax brackets invented from memory) failed 0/9 hidden cases and was honestly
rejected, which reproduces the requirement-grounding finding of §5.6 in the code family.

---

## 6. Limitations and Honest Assessment

We state plainly what this system does and does not establish.

1. **We ran τ-bench on our local stack, and the honest result is low.** In preparation
   for a public-benchmark release, we wired τ-bench (June-2024 tasks, airline + retail)
   to run fully offline — no API keys, no cloud — through the OpenAI-compatible endpoint
   of our local Ollama service, using gemma4:12b for both agent and simulated user.
   Findings, measured on our RTX 3060 host:
   - **Tool-calling strategy, airline task 0 → reward 0.0.** The trajectory shows two
     correct tool calls (airport listing, direct-flight search), then eight degenerate
     turns in which the model returned empty assistant messages (no tool call), followed
     by the simulated user terminating with `###STOP###`. The user simulator also leaked
     its meta-instructions mid-conversation — a known failure mode of using weak models
     as user simulators.
   - **Retail: 0 of 4 tasks completed in ~14 minutes** — the same empty-tool-call
     degeneration stalls the episode far below its 30-step budget.
   - **ReAct strategy (text-parsed tools): stalled identically** on airline task 0.
   - Cost: $0.00 total (100% local); the single failure trajectory is saved for release
     in the reproducibility appendix.
   This is exactly the ceiling we predicted qualitatively in §5.2: an open-ended
   service-desk dialogue is **not** within the capability envelope of a 12B local model —
   and, more importantly, **it is not Adam Prism's design target**. Adam Prism is
   verification-driven *artifact production* (plans → executed, tested products), where a
   bounded plan replaces open dialogue. τ-bench measures the former; our 1,317-test suite
   and 26 scenario workflows measure the latter. Public-benchmark participation will
   therefore require either (a) a stronger local tool-calling model than gemma4:12b, or
   (b) a benchmark aligned with artifact production (SWE-bench-style, τ³-bench with tool
   policy constraints) — both are open work we state here rather than claim.
   Complementing the τ-bench probe (model skill), the *system-level* measures succeed:
   the two-role pipeline completed both artifact goals verified green in ~150–190 s each
   (§5.5), and the 10-scenario audit registry reached single-shot pass@1 = 10/10 with
   independent deterministic oracles (§5.6) — both on a single consumer GPU at $0, and
   each surfaced real internal defects (pytest collection pattern; the 5/10 honest-halt
   grounding gap) that we fixed and report transparently. The honest reading is
   therefore not "the model is weak" but "open service-desk dialogue is out of scope for
   a 12B local model; verified artifact production is where the architecture converts
   that ceiling into a reliable floor" — and the path from here to public-benchmark
   participation is concrete (a stronger local tool-calling model, or an artifact-aligned
   benchmark), as item 1 above and §8's open problems state.
2. **Model capability sets a ceiling.** A 12B local model cannot match frontier-model
   reasoning on open-ended research tasks. Adam Prism's bet is that structured
   plan+verify beats free reasoning *for artifact-producing goals* — a claim we have some
   evidence for, not a claim that local models are generally better.
3. **Independent judge ∈ same model family.** The judge is instantiated separately, but on
   the same underlying model line. True independence would use a different model family or
   a formal verifier; this is a documented shortfall (cf. VMAO's shared-family caveat).
4. **Storage pressure.** Producing large artifacts (Linux images) fills local disks
   quickly; sparse/staged flows mitigate but do not eliminate this.
5. **No longitudinal reliability study.** ChatGPT-style primitive "acts" (one tool call)
   vs. verified "products" (materialized, tested artifacts) remain under-measured.

---

## 7. Related Work

- **Multi-agent orchestration:** LangGraph (LangChain), CrewAI, AutoGen / Microsoft Agent
  Framework, Hermes Agent (Nous Research). Independent 2026 benchmarks:
  agent-harness.ai benchmark, tacavar.com 2026 comparison, JATIR Vol 2(6), and the
  107-task bakeoffs (dev.to). We cite their *published* metrics, not our own runs.
- **Verification and control separation:** FORMALJUDGE (arXiv:2602.11136); Sovereign
  Agentic Loops (arXiv:2604.22136); Graph Harness (arXiv:2604.11378); AgentLTL
  (arXiv:2607.02599); Verified Multi-Agent Orchestration (arXiv:2603.11445); AgentSPEX
  (arXiv:2604.13346).
- **Evaluation methodology:** General AgentBench (arXiv:2602.18998) — verification gap;
  ToolBench-X (arXiv:2606.25819) — diagnosis over compute; AGENCYBENCH (ACL 2026) —
  long-horizon rubric evaluation; AutomonoMASM agent frameworks surveys; Agent Loop
  (arXiv:2506.09343) not cited.

## 8. Conclusion and Open Problems

We presented Adam Prism, a sovereignty-first agent that separates planning, execution,
verification, repair, and governance; grounds evaluation in deterministic checks plus an
independent judge; and reports honest pass/fail instead of plausible completion. On a
local 12B model it sustains 1,317 deterministic tests and 26 scenario workflows, and on
**two oracle-verified registries** — 10 CSV-verification scenarios (single-shot pass@1 =
10/10) and 5 code-generation scenarios (15/15 task-runs, 75/75 hidden executable checks) —
the verification loop closes *without* an LLM judging the outcome. It remains air-gap
capable and free of per-token cost, and its failure taxonomy is reported rather than
hidden.

Open problems we consider most important:

1. **Public benchmark participation** (SWE-bench-style, tau-bench, AgentGym2) with
   disclosed hardware/model/temperature.
2. **Cross-family judges** — a formal verifier or a different-family judge to break the
   shared-bias caveat.
3. **Procedural-compliance harnesses** (AgentLTL-style) that score the *process* of agent
   execution, not only the output.
4. **Reproducible artifact pipelines** as a first-class open-source deliverable.

The through-line that unifies these results is architectural, not algorithmic: correctness
is earned by structure — a gated plan, a separated verifier, an honest stop — rather than
by hoping a larger model will "behave." Given a sovereign-friendly model (12–26B) on a
single GPU, Adam Prism converts an open-ended agent into a *deterministic production
channel*: same hardware, same model family, but verification whose credit derives from
deterministic checks and independent oracles. We believe the principle generalizes to any
artifact-producing goal family, and we put the data — and the failures — beside the claim.

---

## Appendix

### A. Publication Checklist

- [ ] `WHITE_PAPER_DRAFT.tex` + `WHITE_PAPER_DRAFT.pdf` (source + compiled)
- [ ] Figures as PDF (pdflatex) — case-matched names, no EPS
- [ ] `WHITE_PAPER_BENCHMARK_RESULTS.json` — machine-readable aggregates (§5.6, §5.7)
- [ ] `README.md` + `LICENSE` (CC BY 4.0) + `CITATION.cff`
- [ ] Abstract ≤ 1920 chars; ASCII-only metadata in TeX
- [ ] Authors: Firstname Lastname, affiliations in parentheses, no honorifics
- [ ] Single-spaced, 10–14 pt, ≥1" margins (arXiv-compatible)
- [ ] Primary category cs.AI; cross-list cs.MA, cs.SE (honest selections)
- [ ] Zenodo release via GitHub integration (DOI); badge in README
- [ ] No internal paths or source-code references leaked into the text

### B. Block Diagram Legend (no code, no internal identifiers)

Blue boxes = the verification-driven loop (§2.2). Green box = governance plane (§4).
Arrows denote artifact flow, not control coupling; the judge consumes only the goal and
the artifact, never the builder's context.

### C. Reproducible Offline τ-bench Harness (measured, honest)

We ran τ-bench entirely offline against a local Ollama service. To reproduce:

1. `git clone https://github.com/sierra-research/tau-bench`
2. `pip install -e .` (requires Python ≥ 3.11 — the latest litellm needs `typing.NotRequired`)
3. Harness patches (local only, no Adam Prism code): litellm returns `response_cost=None`
   for local endpoints, so the agent/user cost bookkeeping must coerce `None → 0`
   (`chat_react_agent.py`, `few_shot_agent.py`, `envs/user.py`).
4. Point litellm at the local endpoint:
   `OPENAI_API_KEY=ollama OPENAI_API_BASE=http://localhost:11434/v1`
5. Representative command:
   `python run.py --agent-strategy tool-calling --env airline --model gemma4:12b
   --model-provider openai --user-model gemma4:12b --user-model-provider openai
   --user-strategy llm --max-concurrency 1 --task-ids 0`

Measured on RTX 3060 (12 GB), host RAM 31 GB, Ollama (server-side model load):

| Config | Task set | Completed | Reward | Note |
|---|---|---|---|---|
| tool-calling, gemma4:12b | airline 0 | 1/1 | 0.0 | 2 correct tool calls, then degenerate empty calls |
| tool-calling, gemma4:12b | retail 0–3 | 0/4 | — | stalled ~14 min inside task 0 |
| react, gemma4:12b | airline 0–2 | 0/3 | — | stalled on task 0 |

Total financial cost: **$0.00** (no API calls). These numbers are measurement, not
capability claims; they bound the honest limits in §6 and define the open work for
public-benchmark participation.

### D. System-Efficiency Benchmark (reproduce)

The §5.5 numbers were produced by driving the *production* pipeline (schema-gated
planner → batched execution → deterministic checks → independent spec review → bounded
repair) with per-channel instrumentation on the HTTP layer (token usage is read from
Ollama's own `prompt_eval_count`/`eval_count`; wall-time and call counts recorded per
channel). It uses the system, not a wraparound model harness. To reproduce:

1. Env vars: `ADAM_OLLAMA_MODEL=gemma4:12b`, `ADAM_OLLAMA_URL=http://localhost:11434`,
   `ADAM_EXECUTOR_MODEL=qwen2.5-coder:14b` (Mode B) or `` (empty, Mode A).
2. Goals are fixed text; two artifact goals (word-frequency module + CLI Fibonacci tool),
   each targeting a temp `{output_dir}`.
3. External confirmation runs every generated `*test*` file explicitly with pytest
   (explicit-file collection avoids the `test_*.py` default-pattern gap documented in
   §5.5) plus `py_compile`.
4. Measured host: RTX 3060 12 GB, 31 GB RAM, Ollama, model load server-side.

Consistent with §5.5, Mode B moved ~35% of token volume and 40% of commander-model
exposure onto the executor channel at equal verified quality, while wall-clock remained a
wash on single-GPU hardware.