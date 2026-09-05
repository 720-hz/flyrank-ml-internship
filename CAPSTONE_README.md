# Content Review Prioritization — a ranking model for FlyRank search data

**Live paper:** https://720-hz.github.io/flyrank-ml-internship/
**Repo:** https://github.com/720-hz/flyrank-ml-internship

> This file is the FL-09 project README — what the capstone *is*, how to run it yourself, and
> what it actually showed. The root [`README.md`](README.md) documents the internship template
> repo itself (how the assignments and notebooks are organized); this one documents the finished
> product.

---

## What it does, and for whom

A content strategist or SEO editor reviewing pages for an agency has far more pages than review
time. This project ranks a client's content pages by *decline risk*, so a reviewer with a capped
weekly budget (this project illustrates with 20 or 50 reviews/week) knows which pages to look at
first — instead of triaging by gut feel or working oldest-first.

It compares two approaches on the same held-out data:

- **A transparent baseline rule** — three trailing-90-day numbers multiplied together
  (visibility × reachability × CTR gap vs. peers). No training, fully explainable.
- **A Random Forest model** trained on a leakage-checked, safe feature set.

Output is a five-action queue (`refresh_and_review_ctr`, `refresh_content`, `review_ctr`,
`monitor`, `no_action_insufficient_data`) — every page gets a fixed reason code, never a bare
score. See the paper's [Ranked recommendations](https://720-hz.github.io/flyrank-ml-internship/#recommendations)
section for the full breakdown.

**Not for:** client-facing claims, automated publishing/pausing decisions, or anything
presented as validated production infrastructure. See [Limitations](#limitations) below — this
is an internship-scale demonstration project.

---

## Setup (from a clean clone)

```bash
git clone https://github.com/720-hz/flyrank-ml-internship.git
cd flyrank-ml-internship
pip install -r requirements.txt
jupyter notebook "work/notebooks/Week 8/capstone.ipynb"
```

Then **Run All**. No credentials, no gated access, no downloads beyond `pip install` — the
notebook reads `data/raw/content_refresh_anonymized.csv`, which ships in the repo (an anonymized,
~30,000-row sample, no client-identifying data). Runtime is well under a minute on a laptop.

If you don't want to install anything locally, open the notebook directly in Colab:

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/720-hz/flyrank-ml-internship/blob/main/work/notebooks/Week%208/capstone.ipynb?flush_cache=true)

---

## Usage example

The core scoring functions are plain pandas/sklearn and easy to reuse on your own frame with the
same columns. From the capstone notebook:

```python
# baseline: transparent, no training
test_df["baseline_score"] = baseline_score(test_df)

# model: Random Forest on the safe, leakage-checked feature set
model_metrics, rf_safe = fit_and_score(train_df, test_df, SAFE_NUMERIC)

# top 20 pages to review first, by the model's score
top_20 = test_df.assign(score=rf_safe.predict_proba(engineer(test_df, SAFE_NUMERIC))[:, 1]) \
                 .nlargest(20, "score")
```

The full, runnable version — including the client-grouped split and feature engineering it
depends on — is in `work/notebooks/Week 8/capstone.ipynb`, sections 1–4.

---

## Architecture

```
data/raw/content_refresh_anonymized.csv
            │
            ▼
   feature engineering (engineer())
   - safe numeric cols only (position, age, word count, search volume, cpc, ...)
   - impressions/clicks/CTR family EXCLUDED (window-overlap leakage, r=0.98 with label)
   - missingness flags + one-hot categoricals
            │
            ├──────────────┐
            ▼              ▼
   baseline rule      Random Forest
   (no training,      (leakage-checked
    3 factors)         safe features)
            │              │
            └──────┬───────┘
                   ▼
   client-grouped 80/20 split evaluation
   (zero client overlap train/test — see Methodology)
                   │
                   ▼
   5-action ranked queue + reason codes
   (refresh_and_review_ctr / refresh_content / review_ctr / monitor / no_action_insufficient_data)
                   │
                   ▼
   deployed paper (docs/index.html) + this repo
```

Full reasoning for each box — why these features were excluded, why client-grouped over random
split, why Random Forest over the earlier Logistic Regression comparison — is in the paper's
[Methodology](https://720-hz.github.io/flyrank-ml-internship/#methodology) section.

---

## Eval results (v2 — safe, leakage-checked feature set)

Same held-out, client-grouped test split (2,325 pages across 6 unseen clients, base rate 0.391)
for both:

| Metric | Baseline rule | Model (safe features) |
|---|---:|---:|
| Precision@20 | 0.70 | 0.75 |
| Precision@50 | 0.64 | 0.72 |
| Precision@100 | 0.65 | 0.66 |
| ROC-AUC | 0.596 | 0.708 |
| Average precision | 0.476 | 0.550 |

**The honest read:** the baseline rule is a genuinely strong top-of-list ranker on its own — the
model's real advantage is breadth (ROC-AUC +0.112, average precision +0.074) across the *whole*
ranked list, not a single blowout at the top. Precision@100 barely moved (0.65 → 0.66). A prior,
naive random-row split on the same data inflated precision@20 to 0.95 — the client-grouped number
above (0.70 for the baseline) is the one that actually generalizes; see the paper's
[Results](https://720-hz.github.io/flyrank-ml-internship/#results) section for the full validation
story.

---

## Limitations

- **The label is a backward-looking proxy**, not a future-observed outcome — it answers "was this
  page already declining," not "will it decline next month."
- **No time-based holdout.** Validation tests generalization to unseen *clients*, never to
  *future* data.
- **No revenue data.** The "value-at-stake" framing in the recommendations is a directional proxy
  from CPC and estimated recoverable clicks, not a dollar forecast.
- **A residual, disclosed leakage risk.** `impressions_90d`-family features structurally overlap
  the label's own defining window (r = 0.98) and are excluded from the model for exactly that
  reason.
- **The `no_position_data` subgroup (4.0% of pages) is a measurement gap**, not a clean bill of
  health — almost certainly a near-zero-traffic artifact, not evidence of health.
- **No causal claims, anywhere.** This is association/prediction on cross-sectional traffic data —
  never a claim about what caused a change or what Google's algorithm does.
- **An internship-scale demonstration project**, not a validated production system. Nothing here
  should be presented to a client, or used for pricing, staffing, or contractual claims, without
  new, purpose-built validation.

Full list, with more detail on each point, is in the paper's
[Limitations](https://720-hz.github.io/flyrank-ml-internship/#limitations) section.

---

## Built with AI

This project was built with Claude (Anthropic) as a research and coding partner throughout. I did
the research myself, worked through multiple candidate approaches at each stage — feature sets,
model choices, validation design — and only settled on the final plan after iterating through
those alternatives with Claude, rather than taking the first suggestion. Claude helped draft
notebook cells, pipeline code, and the prose for the deployed paper and this README, but the
decisions — the lane, the label definition, which features to exclude, the validation design —
were mine. I double-checked and validated the outputs myself before shipping, including re-running
the pipeline and cross-checking the headline numbers in the paper against the committed metrics
receipts (`work/outputs/action_playbook_metrics.json`) — the notebook's own Section 6 re-verifies
this on every run.

---

## Full submission package (FL-10 index)

Everything for this track, in one place:

- **This README** — what it does, setup, usage, architecture, eval results, limitations, AI use.
- **Live paper:** https://720-hz.github.io/flyrank-ml-internship/
- **Repo:** https://github.com/720-hz/flyrank-ml-internship
- **Live demo video** (FL-09, 3–5 min, live end-to-end run): https://youtu.be/PRKaZWDSsMA
- **[Retrospective](RETROSPECTIVE.md)** — Week-1-me vs. now, what changed, what's next.
- **[Build-in-public post](BUILD_IN_PUBLIC_POST.md)** — the decision + limitation story, drafted here and posted publicly from this copy.
- **Personal site:** https://720hz.is-a.dev/

---

## Reproducibility

Notebooks, in order: [Week 1 — question](work/notebooks/Week%201/w01_research_question.ipynb) ·
[Week 2 — task framing](work/notebooks/Week%202/w02_ml_task_framing.ipynb) ·
[Week 3 — data contract](work/notebooks/Week%203/w03_data_contract.ipynb) ·
[Week 4 — baseline & signal tests](work/notebooks/Week%204/w04_baseline_score.ipynb) ·
[Week 5 — model](work/notebooks/Week%205/w05_model.ipynb) ·
[Week 6 — validation & leakage audit](work/notebooks/Week%206/w06_validation_audit.ipynb) ·
[Week 7 — action playbook](work/notebooks/Week%207/w07_action_playbook.ipynb) ·
[Week 8 — capstone synthesis](work/notebooks/Week%208/capstone.ipynb) (mirrors this README and
the deployed paper).

Fixed random seed (`42`) throughout every split and model fit.

## Data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai).
