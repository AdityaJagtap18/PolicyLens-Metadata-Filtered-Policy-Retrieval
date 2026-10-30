# PolicyLens — Metadata-Filtered Policy Retrieval

> A Forward Deployed Engineering (FDE) proof of concept. This repository contains both the **pre-transformation** (before) and **post-transformation** (after) states of the solution, so the impact of the work can be seen side by side.

**PolicyLens** is a small retrieval benchmark that shows how metadata filters (country, product, effective date) make a policy search return the *correct* answer, not just a similar-sounding one.

---

## 1. Overview

| Item | Details |
|------|---------|
| **POC Name** | PolicyLens — Metadata-Filtered Policy Retrieval |
| **Scenario / Customer** | DecisionStream AI, an illustrative insurer with a claims-policy library (synthetic data) |
| **Owner** | adiraj2503@gmail.com |
| **Status** | Completed |
| **Program** | Birlasoft FORGE FDE Academy · Sprint 1 · Day 12 Lab |

**One-line summary:** Adding metadata filters to a policy index lifts retrieval correctness from about 3/10 to about 9/10 — far more than changing the embedding model, chunker or search mode.

---

## 2. Problem Statement

A claims assistant searches 20 policy documents that contain the same clause with different answers (older versions, other countries, other products). Plain similarity search finds well-written text, but often the wrong policy.

- Old policy versions compete with the current one (e.g. motor total-loss threshold: 70% / 65% / 60%).
- Policies from other jurisdictions (UK, Ireland, EU) look nearly identical to the right one.
- Historical decisions must use the rules in force on the decision date, not today's rules.

---

## 3. Pre-Transformation (Before)

Documents are chunked, embedded and searched with **no metadata filters** — every chunk in the corpus is a candidate.

| Aspect | Current State |
|--------|---------------|
| Process | Embed text and search by similarity only |
| Model / Tooling | Chunker, embedding model and retrieval mode tuned as the only levers |
| Cost | Not measured |
| Latency / Performance | Not measured |
| Accuracy / Quality | ~2.7 / 10 questions correct (`--filters none`) |
| Pain points | Wrong country, outdated version or wrong product returned with high confidence |

**Location in repo:** [Pre-Transformation/](Pre-Transformation/)

---

## 4. Post-Transformation (After)

The index carries metadata (`jurisdiction`, `policy_type`, `effective_from`, `effective_to`, `superseded`) and every query applies a filter **before** similarity ranking. The lab also runs against real Azure OpenAI embeddings and Azure AI Search.

| Aspect | New State |
|--------|-----------|
| Process | Filter by country + product + in-force date, then rank by similarity |
| Model / Tooling | Python stdlib only; optional Azure OpenAI `text-embedding-3-*` and Azure AI Search (REST) |
| Cost | Embedding prices modelled per 1M tokens; not otherwise measured |
| Latency / Performance | Not measured |
| Accuracy / Quality | ~9.0 / 10 questions correct (`--filters full`) |
| Benefits | Correct, date-aware answers; null-safe `effective_to` handling; `vectorFilterMode` comparison; per-mode score cutoffs |

**Location in repo:** [Post-Transformation/](Post-Transformation/)

---

## 5. Before vs. After

| Metric | Pre-Transformation | Post-Transformation | Change |
|--------|--------------------|---------------------|--------|
| Avg. correctness (no filter → full filter) | 2.72 / 10 | 9.00 / 10 | +6.28 questions |
| Jurisdiction-only filter | — | 4.72 / 10 | +2.00 vs. none |
| Best non-filter lever (retrieval mode swing) | 0.56 | — | small |
| Embedding model swing (small vs. large) | 0.07 | — | negligible |

Detailed results: [Post-Transformation/out/index_report.md](Post-Transformation/out/index_report.md), with per-question data in `per_query.csv` and `results.csv`.

---

## 6. Approach

1. **Discover** – Identify why similar-looking policies give wrong answers.
2. **Baseline** – Run the grid with `--filters none`.
3. **Evaluate** – Vary one lever at a time: filters, retrieval mode, embedding model, chunker.
4. **Build** – Add jurisdiction, product and effective-date filters; add live Azure layers.
5. **Measure** – Compare average correctness per lever over 10 questions.
6. **Hand off** – Report and a "How to Read the Results" guide.

### Decision Criteria

| Criterion | Weight | Why |
|---|---|---|
| Correctness (right country, product, date) | Highest | A confident wrong answer is worse than no answer |
| Reproducibility | High | Offline mode needs no keys or installs |
| Cost / re-embedding impact | Medium | Changing metadata design later forces a re-index |

---

## 7. Repository Structure

```
Assignment2/
├── README.md
├── Pre-Transformation/      # "Before" state
└── Post-Transformation/     # "After" state
    ├── run_lab.py           # main command
    ├── index_lab/           # corpus, indexing, Azure embed/search, env
    ├── out/                 # reports and result CSVs
    ├── .env.example         # settings for live mode
    ├── project_analysis.md  # detailed project analysis
    └── How_to_Read_the_Results.docx
```

---

## 8. Getting Started

### Prerequisites

- Python 3.7+ (stdlib only — no `pip install` needed)
- For live mode only: an Azure OpenAI resource and an Azure AI Search service with keys

### Setup

```powershell
cd Post-Transformation
copy .env.example .env   # live mode only; fill in endpoints and keys
```

### Run

```powershell
python run_lab.py                       # full offline grid
python run_lab.py --filters none        # baseline
python run_lab.py --filters full        # with metadata filters
python run_lab.py --show Q06            # inspect one question
python run_lab.py --live search --provision   # live Azure AI Search
```

---

## 9. Results & Learnings

- **Key result:** Metadata filtering improves results by about 6 questions; every other lever improves them by less than 1.
- **What worked:** Filtering on country, product and effective date *before* similarity ranking.
- **What didn't:** Q08 (full paraphrase) fails in every setup; hybrid search is not always better than vector alone.
- **Recommendations:** Design filterable fields up front, handle `effective_to = null` explicitly, prefer `preFilter`, and set separate negative-query cutoffs for BM25, vector and hybrid scores.

> Caveats: Offline embeddings are simulated; the corpus is small (20 documents, 45 chunks) and synthetic; cost and latency were not measured; no automated tests exist.

---

## 10. Next Steps

- [ ] Fix the Q08 judge defect (expected phrase split across a line break)
- [ ] Re-run the grid with real `text-embedding-3-large` embeddings
- [ ] Test `postFilter` vs. `preFilter` on a much larger index
- [ ] Add automated tests

---

## 11. Contact

Owner · adiraj2503@gmail.com
