# Adam Prism: A Sovereignty-First Architecture for Verified, Deterministic LLM Agents

**Paper-only repository.** This repository contains the white paper, its LaTeX/markdown
sources, and the measured benchmark data that back its claims. It intentionally contains
**no source code** — the Adam Prism implementation remains in a separate, private
repository.

**DOI:** [10.5281/zenodo.22638788](https://doi.org/10.5281/zenodo.22638788)
([Zenodo record](https://zenodo.org/records/22638788))
[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.22638788.svg)](https://doi.org/10.5281/zenodo.22638788)

## Contents

```
paper/
  WHITE_PAPER_DRAFT.pdf      compiled paper (recommended for reading)
  WHITE_PAPER_DRAFT.tex      LaTeX source (arXiv/build compatible)
  WHITE_PAPER_DRAFT.md       markdown source (source of truth for content)
data/
  WHITE_PAPER_BENCHMARK_RESULTS.json   machine-readable aggregates (§5.5–§5.7)
  SYSTEM_TRACE_DATA_AUDIT.md           trace audit of the measurement pipeline
  f2_summary.json                      second-goal-family summary (15/15, 75/75)
  results_f2_r1/2/3.json               per-replicate raw task results
```

## Rebuild the PDF

```bash
pdflatex paper/WHITE_PAPER_DRAFT.tex   # run twice for references
```

Requires only a standard TeX Live 2023+ install (no `minted`, no `-shell-escape`).

## License

The paper, sources, and data in this repository are licensed under
**CC BY 4.0** (see `LICENSE`). This permits reuse with attribution. The Adam Prism
**implementation** is NOT included here and is governed by its own license.

## Citation

See `CITATION.cff`. Preferred BibTeX:

```bibtex
@misc{othman2026adamprism,
  author = {Othman, Mohamed},
  title  = {Adam Prism: A Sovereignty-First Architecture for
            Verified, Deterministic LLM Agents},
  year   = {2026},
  note   = {Independent Researcher, Cairo, Egypt}
}
```

## Reproducibility

All evaluation numbers in §§5.5–5.7 are reproducible offline:

- **§5.5 System-Efficiency:** two artifact goals, two-role pipeline (Mode A/B),
  per-channel token instrumentation. See `WHITE_PAPER_BENCHMARK_RESULTS.json` →
  `system_efficiency`.
- **§5.6 Audit Registry:** 10 CSV data-verification scenarios, independent
  deterministic oracles, verbatim match on confirmed-patterns lines. Aggregate in
  `WHITE_PAPER_BENCHMARK_RESULTS.json` → `audit_registry`.
- **§5.7 Code Generation:** 5 scenarios x 3 replicates = 15 runs, 75/75 hidden
  executable checks. See `f2_summary.json` and `results_f2_r{1,2,3}.json`.

No API calls are required to re-verify any aggregate; every check listed above is
deterministic and runs locally.

## Data provenance

`SYSTEM_TRACE_DATA_AUDIT.md` records the audit trail of how the measurement trace was
collected, reviewed, and fixed, including the honest negatives reported in the paper
(empty-tool-call degeneration on τ-bench; the specification-less pilot that failed 0/9
oracle checks). Negative results are retained deliberately.