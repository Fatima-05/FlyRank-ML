# Capstone Report —

* Author: Fatima-Tu-Zahra
* Lane: Refresh / Content Opportunity Scoring
* Repo: https://github.com/Fatima-05/FlyRank-ML
* Date: 20 September 2026

Copy source: repo template → filled for the measured capstone run.

---

## 0. Abstract

Small content teams need a short list of which pages to refresh first.
This project uses anonymized page-level search and content signals from the FlyRank ML Internship dataset.
A hand-written baseline rule is compared to tree-based models on the same client-holdout split, using Precision@50.
On this run, random forest reaches 0.62 Precision@50 versus 0.50 for the baseline (decision tree 0.42).
The output is a ranked action queue with scores, actions, and reason codes for human review, not auto-publish.

---

## 1. Problem framing

**Decision supported:** which pages should enter the next refresh queue first.

**Unit of analysis:** one content item (page) after aggregation.

**Output:** ranked queue with score, suggested action, and reason codes.

**Action a human takes:** open the top of the queue, review the flagged pages, refresh or monitor within a weekly capacity limit.

**Cost of a wrong call:** wasted editor time on low-value pages, or missing pages that still have demand and are aging/under-clicking.

**Why data/ML helps:** hand review does not scale across tens of thousands of pages; a transparent baseline plus a learned ranker can surface a short, comparable list on the same metric.

---

## 2. Data safety

**Used:**
- Starter page-level table `data/raw/content_refresh_anonymized.csv` (~30,000 rows) for the reproducible capstone comparison
- Internship warehouse access pattern (DuckDB / `hf://`) in earlier weekly notebooks for feature exploration

**Deliberately excluded from features and public outputs:**
- Client names, domains, URLs
- Private queries
- Credentials / raw private exports
- Label-derived shortcuts used as features (e.g. putting `trend_direction` into the model feature matrix or into the baseline score)

**Leakage risks considered:**
- `trend_direction` / decline labels are evaluation targets, not model features
- Pseudonymous IDs (`client_id`, `content_id`) used for grouping/splitting and display only, never as predictive features
- Final month treated as sealed in the broader track practice; this notebook comparison uses a client-holdout split on the starter table

**Confirm:** no client-identifying details in `work/` public artifacts used for the paper.

---

## 3. Baseline

**Rule:** hand-written score using only non-label signals:
- impressions / demand
- content age
- low CTR on visible pages
- striking-distance-ish position
- thin word count

**Why fair:** does not use `trend_direction` (the proxy label). Compared on the same holdout rows and the same metric as the models.

**Numbers (Precision@50, same split):**
- baseline_rules: **0.50**

---

## 4. Model / analysis

**Method:** decision tree and random forest classifiers producing a ranking score (`predict_proba` for the positive class), then sorting pages by that score.

**Why it fits the lane:** Refresh / Content Opportunity Scoring needs a prioritised list, not only a global accuracy number.

**Features used:**
`impressions_90d`, `clicks_90d`, `ctr`, `content_age_days`, `word_count`, `avg_position`, `search_volume` (whichever exist in the table; missing filled with 0)

**Left out on purpose:**
- `trend_direction` as a feature
- client/content IDs as features
- any future-window outcome fields

**Target / proxy (one sentence):** `is_declining = 1` when `trend_direction` is down — a proxy for evaluation, not confirmed editorial ground truth.

---

## 5. Evaluation

**Split:** `GroupShuffleSplit` on `client_id` (25% test, `random_state=42`) so test clients are held out from training.

**Why:** reduces the “same client pages in train and test” leak pattern.

**Metric:** Precision@50 — share of the top 50 ranked test pages that match the proxy label. Chosen because a small team only works a short list.

**Base rate:** test declining rate ≈ **0.517** (majority-class baseline context for reading Precision@50).

**Results on the same split:**

| model           | Precision@50 |
|-----------------|--------------|
| baseline_rules  | 0.50         |
| decision_tree   | 0.42         |
| random_forest   | 0.62         |

**Lift:** random forest ≈ **1.24×** baseline on this run.

**Error sketch:** the baseline sits near the base rate; the forest improves top-50 concentration of declining pages but still leaves false prioritisation risk — hence human review remains required.

---

## 6. Interpretation

The forest ranks higher when combinations of visibility, aging, weak CTR, and position-like cues line up.

Top-queue reason codes observed in the sample include `high_impressions`, `low_ctr`, `aging`, and `striking_distance`.

**Surprise / negative note:** decision tree underperformed the hand-written baseline on Precision@50 in this run (0.42 vs 0.50). That is reported honestly; the forest still leads.

This is measured ranking quality on a proxy label, not proof of which pages would gain traffic after a refresh.

---

## 7. Recommendation

**How an editor would use this tomorrow:**
1. Sort by model score descending  
2. Work the top of the queue first  
3. Read reason codes to see why a page was flagged  
4. Cap capacity (example: 8–10 pages per cycle)  
5. Final decision stays human — queue is a reviewer aid  

**Confidence:** moderate for offline prioritisation on this dataset and split.

**Limits:** no live adoption study, no causal refresh experiment, proxy label ≠ editorial truth.

---

## 8. Reproducibility

From a fresh clone of the repo:

```bash
# example Colab-style path
git clone https://github.com/Fatima-05/FlyRank-ML.git
cd FlyRank-ML
# open and Run-all: work/notebooks/capstone.ipynb
```

**Seeds:** `random_state=42` for split and models.

**Key outputs written by the notebook:**
- `work/outputs/capstone_precision_at_50.csv`
- `work/outputs/capstone_top10_queue.csv`
- `work/outputs/capstone_precision_at_50.png`

**Environment:** standard Colab / `scikit-learn` + `pandas` + `matplotlib` (see repo `requirements.txt` if present).

---

## 9. Acknowledgments & data credit

Built on the FlyRank ML Internship dataset.  
https://flyrank.ai

