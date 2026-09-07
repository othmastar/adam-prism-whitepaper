# SYSTEM_TRACE — data_audit_fraud (مراجعة بيانات واكتشاف مخالفات)

> Living trace of Adam Prism's REAL mechanism + tool-calling on a built-in scenario
> (local RTX 3060 12GB, Ollama, both models on one GPU). Raw data in
> `WHITE_PAPER_BENCHMARK_RESULTS.json` → `trace_runs[0]`.

## Execution summary
- 9 steps, 143.4 s wall, **6/9 completed → honest `success=False`**
- 13 LLM calls, ~32.2K prompt chars in, ~7.8K out, est. 10,003 tokens
- **13 model evictions** (VRAM swaps: qwen2.5-coder:14b ×6, gemma4:12b ×7) — each
  two-role task costs a pair of `keep_alive=0` unload/reload cycles on ONE GPU.

## Tool → mechanism call chain observed
```
step  tool              mode     result            ms        channel
0    plan_generator     PLAN     plan text         14,373    gemma4 (652+402 tok)
0r   plan_reviewer      REVIEW   plan review        3,219    gemma4 (1,451+31)
1    list_files         EXECUTE  file index             1    deterministic
2    code_sandbox       EXECUTE  loader code+run   23,482    qwen (2,694+477) sandbox
3    code_sandbox       EXECUTE  anomaly code+run  12,450    qwen (2,974→8,999 across 3 att) sandbox
4    llm_analyze        EXECUTE  classified JSON    3,060    qwen (604→2,111 across 3 att)
5    template_render    EXECUTE  report (HTML)          1    deterministic (template)
R    output_evaluator   REVIEW   verdict fail 0.52 12,524    gemma-as-judge (rubric)
     → Replan x3 (attempts 1/2, 2/2): re-runs steps 3→4→5→R each cycle
V    verify_global      REVIEW   final gate         5,183    gemma (1,520+136)
```

## Why it halted (root cause, honest)
- `output_evaluator` (LLM rubric) scored **step5/report = 0.52** twice; step3/step4 scored 1.0.
- The report is produced by `template_render` — **deterministic** (same template → same output),
  so replan cycles regenerated the *code* (steps 3–4, which already passed) but could not
  change the failing report → 3 replan cycles exhausted `max_retries` → `fallback none`.
- No auto-repair ladder engaged (`repairs=0`): the repair ladder only triggers on
  **deep-check failures** (`deep_checks=4 run, all passed`), not on review-verdict failures.
- Spec gate: `spec غير قابلة للتحقق (llm, 0 متطلبات) → functional checks only`; `spec_audits=2`.

## Complexity map (layers that actually fired)
| Layer | Observed |
|---|---|
| Deterministic engine | list_files (1ms), template_render (1ms), schema validation on every step |
| Sandbox | code generation AND execution isolated in `code_sandbox` (2 tools × 3 attempts) |
| Two-role LLM channels | gemma4 commander (plan/review/evaluate/verify), qwen executor (code) + VRAM swap orchestration |
| Step-level validation | is_json / not_empty gates per step |
| Review loop (LLM judge) | output_evaluator rubric → replan cycle (re-executes dependents) |
| Verification loop | 4 deep-checks, 0 failed |
| Spec-conformance judge | ran (2 audits), skipped (0 spec requirements) → functional-gate fallback |
| Retry/fallback ladder | validation-retry × replan × max_retries → **honest failure recorded** |

## Takeaway
Even a curated "successful" scenario does NOT pass 9/9 in one shot with 12B local models —
instead the system refuses to claim success (blocks on a 0.52 report while code passes 1.0),
spending ~143s / 13 calls / 13 VRAM swaps. The refusal is the product: deterministic steps +
LLM rubric evaluate + replan fallback = fail-closed quality control, but it also means
**deterministic report steps are unfixable by the replan loop** (valuable design note: route
report generation through the LLM channel or exempt deterministic artifacts from rubric review).
## Fix applied + verified (2026-09-05)

Fail-closed was correct behavior, but the report step _should_ improve — so the executor now
routes around the trap. Changes (backed by 1313+6 unit tests + ruff clean):

| Component | Change |
|---|---|
| `_replan` | review-failure attribution: failing `TEMPLATE_RENDER` targets → `template_force_llm=True` |
| `_handle_template_render` | 3 paths: LLM regeneration / deterministic renderer / legacy stub |
| `_render_template_report` | deterministic Markdown report (executive summary, artifacts, recommendations) + grounding (real file list, patterns) + `min_len` structural appendix; persisted to `final_report.md` (+ optional HTML) |
| `_compute_data_grounding` | stdlib `csv`/`json` only: rows, missing values, **duplicates**, **multi-recipient heuristic**, sum/mean/max, JSON keys; merged into every `code_sandbox` analysis as `_data_grounding` + persisted `data_summary.json` |
| `_v13_spec_conformance` | goal augmented with scenario `expected_output_spec` (as `_reference_spec`) for both audit and repair — judge now enforces the real spec (file list, patterns, risk classification, recommendations, JSON summary) |

**Results (live two-role pipeline, real Ollama):**

| Run | Steps | Wall | Calls | VRAM swaps | Spec audits/repairs/failures | Outcome |
|---|---|---|---|---|---|---|
| v2 stub baseline | 6/9 | 143.4s | 13 | 13 | —/-/- | honest fail (report 0.52) |
| v4 after template fix | 6/9 | 690.9s | 15 | 41 | 2/2/2 | honest fail (zero-defaults report) |
| v6 grounded + spec-goal | 9/9 | 485.2s | 11 | 15 | 2/0/0 | **pass** |
| v7 final (txt rows fix) | 9/9 | 285.1s | 11 | 13 | 2/0/0 | **pass** |

Evidence: `docs/evidence/final_report_run7.md` (3881 chars, sections: قائمة الملفات المفحوصة /
الأنماط المكتشفة / تصنيف الخطر / التوصيات; 10 files, 21,071 rows) + `docs/evidence/data_summary_run7.json`.

**Why v6→v7 faster:** txt files now contribute real row counts instead of no-op → more grounded
prompt → reviewer/step5 converge on fewer regen rounds (285s vs 485s; call counts identical).

**Paper lessons:** (1) fail-closed "refusal" is a feature and gives a clean fleet-wide metric;
(2) the fix is **validator-visible intermediates**: the decisive step was making judge-required
categories (files/patterns/risk/recommendations/JSON) appear as *grounded artefacts* the judge can
audit — mirror for all scenarios targeting paper tables; (3) honest LLM-judge "zero-defaults"
finding caught fabricated numbers that a deterministic checker would have missed.

---

## AUDIT REGISTRY — 10 scenarios, independent deterministic oracles (2026-09-06)

A second, *independent* measurement layer: a **registry of 10 data-verification scenarios**
(same buy/sell-audit goal family as `data_audit_fraud`), each with its own **independent
deterministic oracle** that checks the produced report against the TRUE synthetic population —
not against the LLM rubric. The oracle confirms exact-verbatim substrings on the
renderer's confirmed-patterns + facts lines. Raw drivers in
`docs/evidence/registry_final/` (`run_registry.py`, `run_all.sh`, `reg_final.log`,
`out/results_<task>.json`); controlled study in the same dir
(`run_experiment.py`, `run_exp_all.sh`, `exp_out/results_<arm>_<task>.json`,
`analyze_exp3.py` → `exp_out/exp_summary.json`); aggregate in
`WHITE_PAPER_BENCHMARK_RESULTS.json` → `audit_registry`.

**Scenario family (all `max_retries=4`, step5 `min_len=2000`, 10 tasks):**

| Task | Goal (Arabic) |
|---|---|
| dup_payments | اكتشاف فواتير مكررة تُدفع لأكثر من مورد |
| high_value | كشف المعاملات عالية القيمة القابلة للمراجعة |
| missing_fields | فحص الحقول الناقصة في قاعدة عملاء |
| multi_recipient | رصد رسائل متعدد المستلمين ذوي الصلاحيات |
| budget_util | تقييم فاقد الاستغلال الميزاني على مستوى الفرق |
| payroll_recon | مراجعة قوائم الرواتب عن قيم شاذة |
| ledger_balance | تحقق من توازن دفتر الأستاذ بين الوارد والصادر |
| empty_category | رصد النفقات في فئة فارغة أو غير مصنفة |
| sales_growth | قياس انخفاض نمو المبيعات الشهري |
| dup_pairs | اكتشاف أزواج السجلات المكررة عبر مفاتيح متعددة |

**Final clean batch (single full run of all 10, freshly copied per task):**

| Metric | Value |
|---|---|
| pass@1 (single-shot, no repair retries) | **10/10 (100%)** |
| Oracle checks passed | **80/80** |
| System steps completed | 9/9 on all 10 |
| Wall mean / min / max | 140.5 s / 121.9 s / 184.8 s (total 1404.7 s) |
| VRAM swaps mean | 11.5 (gemma4 ×63, qwen2.5-coder ×52) |
| Spec audits / repairs / failures | 0 / 0 / 0 (rubric-gated, not spec-gated) |
| Financial cost | $0.00 (100% local) |

**Why this is a meaningful (and honest) result — and the failure taxonomy that shaped it:**
an earlier clean run scored **5/10** — all five failures were *honest halts* with factually
correct reports that the LLM judge had rejected because the report's own prose defaulted to
«لا توجد أنماط» while the data demonstrably contained anomalies (e.g. 12 missing values).
That was a **real model defect** (the judge was right), not a judge false-negative. The fix —
lifting more anomaly classes out of the data into *validator-visible grounded intermediates* —
is the same architectural lesson as the trace above, and it took the fleet to 10/10 with **no
residual failure class**: every one of the 10 distinct anomaly families (duplicates, high
value, missing fields, multi-recipient, overspend, anomalous payroll, ledger imbalance, empty
category, negative growth, cross-key dup pairs) is now surfaced by the renderer and verified
verbatim by an independent oracle.

### Controlled decomposition (EXP-3, same tasks / same seeds / same oracle)

The 5/10 natural experiment became a controlled study: 30+ fresh runs across four arms that
remove one layer at a time. **PASS = pipeline success AND oracle green.**

| Arm | pass@1 | Oracle | Med. wall | Reading |
|---|---|---|---|---|
| System rep1 (archived clean run) | **10/10** | 80/80 | 138.4 s | full pipeline |
| System rep2 | **10/10** | 80/80 | 127.2 s | independent rerun |
| System rep3 | **10/10** | 80/80 | 126.3 s | independent rerun |
| Grounding only, no verify/repair | 0/10 | 40/80 | 41.6 s | facts right; contract wrong |
| No grounded classes (ablate) | 7/10 | 78/80 | 146.4 s | 1 honest halt + 1 silent error + 1 halt on correct artifacts |
| Raw model, full data inlined | 0/10 | 60/80 | 12.7 s | prose right; numbers invented |

Mechanical reading, per arm:

1. **Raw model (free)** — full CSVs inlined + same contract + same oracle: emits the four
   sections and valid JSON, then misses nearly every numeric fact (`total_rows=794` for a
   300-row file; wrong counts per anomaly). Med. wall 12.7 s: fast, confident, wrong.
2. **Grounding only (noverify)** — data-summary facts reach the report deterministically,
   but without the attributed-review → regeneration loop the report fails the four-section
   contract (every task 4/8). Verification's measured contribution = deliverable
   conformance.
3. **No grounded classes (ablate)** — restores the pre-intervention evidence
   (dup/multi-recipient/max only): `missing_fields` honest-halts (ungrounded figure
   refused); `empty_category` is a **silent error** (success claimed with a wrong count,
   caught only by the oracle); `sales_growth` carves a conservative halt whose persisted
   artifacts were already 8/8.
4. **Full system ×3** — no failure in 30/30 task-runs; the single-run archive result is
   not stochastic.

## CODE-GENERATION REGISTRY — 5 scenarios, executable oracles (2026-09-07)

A **second goal family** to break the one-family limitation: 5 SOFTWARE_ENGINEERING
scenarios whose oracles are *executable* — `py_compile` + a subprocess import of the
produced `module.py` + deterministic hidden cases (expected values computed from a
reference implementation) + `final_report.md` existence. No LLM, no rubric in the oracle.
Raw driver in `docs/evidence/registry_final/` (`run_family2.py`, `run_f2_all.sh`,
`f2_full.log`, `f2_out/results_f2_<rep>.json`); aggregate in
`WHITE_PAPER_BENCHMARK_RESULTS.json` → `code_registry`.

**Scenario family (8 steps: plan → review-plan → list → write → code-review → report →
attributed review → verify).** The write step uses the executor channel
`qwen2.5-coder:14b`; the review gate targets the code-review product (function/signature/
edge cases/test count) rather than report prose, and the report render is deterministic
(grounding evidence + artifacts, LLM re-writes disabled) so the judged output is not
fence-wrapped.

| Task | Function | Wall med (s) | Oracle | pass@3 |
|---|---|---|---|---|
| mergesort | `sort_nums` | 66.2 | 5/5 | 3/3 |
| dedupe_pairs | `dedupe_pairs` | 63.8 | 5/5 | 3/3 |
| payroll_net | `net_pay` | 63.8 | 5/5 | 3/3 |
| topk_words | `top_k_words` | 62.7 | 5/5 | 3/3 |
| budget_overspend | `find_overspending` | 60.5 | 5/5 | 3/3 |

| Metric | Value |
|---|---|
| Runs (3 independent reps × 5 tasks, single-shot) | **15/15 (100%)** |
| Hidden executable checks | **75/75** |
| System steps completed | 8/8 on all 15 |
| Wall mean / median | 74.3 s / 63.6 s (min 56.8 s, max 235.2 s) |
| VRAM swaps mean | 6.9 (gemma4 ×36, qwen2.5-coder ×67) |
| Internal gate ∧ external oracle mismatch | 0 runs |
| Financial cost | $0.00 (100% local) |

**Honest negative that shaped the result:** the first payload — `payroll_net` **without** an
injected requirement spec — invented tax brackets from memory, failed the oracle **0/9**
hidden cases, and the pipeline reported an honest failure (no false "success"). Injecting
the exact spec (the code-family equivalent of gold facts) yields the 15/15 above, mirroring
the requirement-grounding finding of EXP-3 in a second family.

**Outlier note (kept for honesty):** one run (r3/payroll_net) took 235.2 s with 19 VRAM
swaps vs a 6-swap typical — matching a transient Ollama slowdown (model reloads during
frame hiccups); it still converged to PASS (oracle 5/5, sys True, 8/8 steps), so only wall
and swap stats are affected, not the verdict.
