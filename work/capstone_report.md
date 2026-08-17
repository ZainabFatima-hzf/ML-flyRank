# Capstone Report — Growth / Recovery / Momentum Prediction

- **Author:** Zainab Fatima
- **Lane:** Growth / Recovery / Momentum Prediction (freestyle)
- **Repo:** https://github.com/ZainabFatima-hzf/ML-flyRank
- **Date:** August 2026

## 0. Abstract

Can a page's prior 28-day search and engagement signals flag which pages are likely to decline
over the *next* 28 days, well enough to beat a transparent hand-built rule? Using FlyRank's
production search warehouse (March 2026, ~9.8M daily rows, 331K content items, client-grouped
validation), a Logistic Regression trained on seven prior-window features achieved a mean
precision@50 of 0.712 (±0.125) across five client-grouped splits — clearly ahead of a hand-built
CTR-vs-position rule's 0.456 (±0.206), which itself performed below the observed base rate
(0.516). This is a decision-support ranking output for a weekly, human-reviewed content review
queue — not a causal or general claim, and not validated beyond the one month and client set
tested here.

## 1. Problem framing

**Decision supported:** a FlyRank content/SEO lead choosing which pages, out of a much larger
inventory, to review for a refresh this week.

**Unit of analysis:** one content page (`content_hash_id`), scored at a fixed decision date.

**Output:** a ranked, reason-coded shortlist — not a single score in isolation.

**Action a human takes:** review the top of the list first; each row carries a reason code (e.g.
"weak position," "declining volume") mapped to a concrete next step (rewrite title, flag for
refresh, deprioritize).

**Cost of a wrong call:** a false decline flag wastes reviewer/writer time; a missed decline lets
real traffic loss compound unnoticed. The project treats missed declines as the costlier error and
optimizes for precision within a fixed review budget (precision@K) rather than plain accuracy.

**Why ML, not a fixed rule:** tested directly, not assumed. A hand-built rule combining the same
kind of signals FlyRank's own refresh/CTR-fix flags rely on scored *below* the base rate on
held-out data — no single signal or simple threshold combination cleanly separated declining from
non-declining pages (the strongest individual correlation checked was -0.164). A model that
weighs several weak signals together earned a real, repeatable improvement instead.

## 2. Data safety

**Source:** `FlyRank/internship-warehouse` (Hugging Face), `fact_content_daily_performance` joined
to `dim_content` and `dim_clients`. Development window: March 2026 (9,841,378 rows, 331,437
content items, 55 clients that month). The panel's final month
(`fact_content_daily_performance_sample.parquet`, June 2026) was never touched — reserved as a
sealed holdout.

**Deliberately excluded, and why:**
- `trend_direction` / `trend_pct` — confirmed (correlation ≈ 1.0) to be a same-window snapshot
  comparison, not a forward-looking outcome. Reference-only, never a feature.
- FlyRank's own product-decision fields (health score, priority score, refresh flags) — never
  pulled into the project, to avoid the model simply copying an existing decision.
- `days_since_update` — knowable for only ~12% of rows at the decision date used, and reversed
  direction on that thin slice. Excluded on evidence, not assumption.

**Leakage risks considered:** pseudonymous IDs (`content_hash_id`, `client_hash_id`) used only for
joining and for grouped-split boundaries, never as features. No client names, URLs, or private
queries appear anywhere in this repo.

## 3. Baseline

A transparent rule: flag a page if it's visible and ranking well (position 1-20, ≥100 prior
impressions) but has low click-through (<2%); score = prior impressions among qualifying pages.
Built only after testing its underlying assumptions directly — a staleness-based gate was tested
first and found **OPPOSITE** of FlyRank's own refresh-flag assumption (decline rate fell, not
rose, with page age) and dropped before it reached the final rule.

**Same-split result:** mean precision@50 = 0.456 (±0.206), mean precision@200 = 0.454 (±0.200),
against an observed base rate of 0.516 — the rule underperformed random selection on average.

## 4. Model / analysis

**Method:** Logistic Regression — the task is a yes/no outcome with an observed label, evaluated
by ranking, which maps directly to this method first. A Random Forest was trained on the identical
split specifically to test whether complexity earned its keep.

**Features (7, all knowable before the decision date):** `prior_28d_impressions`,
`prior_28d_clicks`, `ctr`, `prior_28d_avg_position`, `prior_28d_sessions`,
`ga4_available_in_prior_window`, `content_age_days`.

**Target (proxy, stated plainly):** `declined_next_28d` = 1 if total impressions over the 28 days
after the decision date are lower than the 28 days before it, 0 otherwise — a genuine
forward-window comparison with an enforced gap between feature and label windows.

## 5. Evaluation

**Split:** client-grouped (every page from one client stays entirely in train or entirely in
test), reported as mean ± std across 5 different grouped splits — a single split was found to be
unreliable, with the test-set base rate swinging from 0.387 to 0.631 across seeds (only 9 clients
per test fold).

| Method | Base rate | Precision@50 | Precision@200 |
|---|---|---|---|
| Week-4 rule | 0.516 ± 0.104 | 0.456 ± 0.206 | 0.454 ± 0.200 |
| Random Forest | 0.516 ± 0.104 | 0.676 ± 0.155 | 0.617 ± 0.128 |
| **Logistic Regression** | 0.516 ± 0.104 | **0.712 ± 0.125** | **0.674 ± 0.147** |

**Errors:** the model's absolute discrimination is modest (ROC-AUC ≈ 0.56-0.59) even though its
top-of-list ranking performs well — a tool for building a short, prioritized list, not for
confidently judging any single random page. Several of the top-ranked pages showed CTR at or near
0.00% on tens or hundreds of thousands of impressions — plausible as a real content problem, but
equally consistent with a broken tracking pixel, and flagged as needing a human GSC check before
acting.

## 6. Interpretation

Logistic Regression's largest standardized coefficients were `prior_28d_clicks` (-0.51),
`prior_28d_sessions` (+0.28), and `prior_28d_impressions` (+0.25) — volume and engagement signals
dominate, not CTR alone (`ctr`'s coefficient was the smallest of the seven, +0.015), despite CTR
being the feature that survived Week 4's isolated signal check. Random Forest's permutation
importance instead leaned heavily on `content_age_days` (0.091, roughly 4x the next feature) — a
direct train-with/train-without test confirmed this is a real but modest signal (AUC drop of only
0.035), and cross-checking against client-level averages suggests it functions more as a
between-client cohort pattern (older client portfolios decline more, on average) than a clean
per-page decay clock. A genuine negative result worth stating: the CTR-vs-position signal that
looked CONFIRMED in isolation did not end up as the dominant driver of the model that actually
beat the baseline.

## 7. Recommendation

Ranked queue reason codes and mapped actions, from the held-out test-client scoring run:

| Reason code | Share | Action |
|---|---|---|
| `weak_position` | 46.1% | Deprioritize unless business-critical; consider consolidation |
| `declining_volume` | 45.4% | Flag for content refresh (add depth, update examples) |
| `low_engagement` | 5.8% | Expand page depth; check for a better-matching competitor page |
| `established_cohort_risk` | 2.5% | Route to a portfolio-level review, not a single-page edit |
| `low_ctr_well_ranked` | 0.2% | Rewrite title/meta description; check search-intent match |

**Confidence and limits:** a weekly, human-reviewed shortlist tool — never an autonomous
publishing or auto-editing system, never a basis for client billing or writer performance
judgments, and not validated outside the one month and ~43-55 client slice tested here.

## 8. Reproducibility

**Repo:** https://github.com/ZainabFatima-hzf/ML-flyRank — notebooks under `work/notebooks/`
(`w01` through `w07`, plus `capstone.ipynb`). Random seed for all grouped-split evaluation: `42`
(primary), cross-checked against seeds `0, 1, 2, 99` for stability. Environment: Python 3,
`duckdb`, `scikit-learn`, `pandas` (see each notebook's setup cell for the exact `pip install`).
Re-running Weeks 3-7 requires a Hugging Face `HF_TOKEN` (gated dataset access, instant approval)
stored as a Colab secret — never committed to the repo. The sealed June 2026 holdout month was
never opened during this project; the cell/logic that would build a sealed-evaluation frame does
not yet exist and is flagged as future work, not claimed as already done.

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset — [flyrank.ai](https://flyrank.ai).

---

**Claims checklist:** all results above are reported as observed, measured, and decision-support
— no causal claims are made without an experiment, and no claim extends beyond the one month and
client slice actually validated. Every precision@K and accuracy-style number is reported next to
its base rate.
