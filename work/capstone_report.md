# Capstone Report — Growth / Recovery / Momentum Prediction (Freestyle)

- **Author:** Ziad Khaled
- **Lane:** Freestyle — Growth / Recovery / Momentum Prediction (content-decline review prioritization)
- **Repo:** https://github.com/720-hz/flyrank-ml-internship
- **Date:** September 2026

> This file mirrors the deployed paper (`docs/index.html`, live at
> `https://720-hz.github.io/flyrank-ml-internship/`). Sections 0 and 9 are the paper's required
> bookends; sections 1-8 are the working notes behind it.

## 0. Abstract

With a fixed weekly review capacity, which content pages should a strategist look at first — and
can a model beat a transparent rule at ranking them? Using an anonymized sample of 30,000 content
items across 32 pseudonymous FlyRank clients (one trailing 90-day search-performance snapshot per
page), I compared a transparent CTR-vs-position baseline rule against a Random Forest trained on a
leakage-checked feature set, evaluated on a client-grouped holdout with zero clients shared between
train and test. The model lifts precision among the top 20 flagged pages from 0.70 to 0.75, and
whole-list ranking quality (ROC-AUC) from 0.596 to 0.708 — a real, moderate improvement over an
already-competitive rule, not a dramatic reinvention of it. This is a decision-support ranking for a
content reviewer's queue: directional and observed, never a causal claim, and never a substitute for
a human's final call on any one page.

## 1. Problem framing

**Unit of analysis:** one content item, for one client, as of one trailing 90-day snapshot (later
extended, for exploratory work only, to one content item x one report date in the daily warehouse
panel). **Output:** a priority score and rank per page, plus a fixed reason code and one of five
actions. **Action a human takes:** a content strategist opens the top of the queue for their
client(s), reads the reason code and archetype, and decides what to actually do. **Cost of a wrong
call:** a false positive burns one of a reviewer's limited weekly slots on a page that didn't need
it (bounded, recoverable); a false negative lets a page with real demand keep losing visibility
unnoticed until the next cycle (compounding). **Why ML helps at all:** 54.2% of pages are trending
down and 80.9% of those still carry real demand — at 20 reviews/week that's a 658-week backlog, so
*which* pages get reviewed first is the whole decision. A first pass correlating eight candidate
signals against the outcome found every one near zero (max |r| ≈ 0.05) — the shape of a problem
where combining many weak signals beats hand-writing one rule, which is exactly what gets tested
empirically in Sections 3-5 rather than assumed.

## 2. Data safety

**Used:** `data/raw/content_refresh_anonymized.csv` (ships in the repo, no credentials needed) —
30,000 content items, 32 pseudonymous clients. `client_id` / `content_id` are opaque hashes used
only for grouping and joins, never as model features. **Deliberately excluded:** FlyRank
product-decision fields (`health_score`, `priority_score`, `action_type` — not shipped, and would
be circular if they were); raw client names, domains, URLs, titles, or search queries (never
shipped, and excluded from every output). **Label-derived fields, never used as features:**
`trend_direction` and `trend_pct` (the label's own source columns) are used only to build
`is_declining_label`, never as model inputs. **Leakage risk found and handled:** `impressions_90d`
and its trailing-window relatives correlate at r = 0.98 with the label's own defining columns — a
structural window overlap, not a copy — and are excluded from the model's feature set for that
reason (Section 4, Section 5). Confirmed: no client-identifying strings appear anywhere in `work/`
or `docs/` (checked directly — see Reproducibility).

## 3. Baseline

A transparent CTR-vs-position rule, chosen only after testing it against a rejected alternative.
**Staleness** (days since last update — the assumption behind refresh-tier flags) tested **MIXED**:
decline rate rises through the 91-180 day bucket, then drops in the 181-365 bucket, with too few
pages in the oldest buckets to trust. **CTR vs. position tier** tested **CONFIRMED**: mean CTR falls
cleanly from 2.71% at top-3 to 0.15% at deep positions. The rule is built on the confirmed signal
only: `score = visible × reachable × ctr_gap × log1p(impressions_90d)`, no fitted weights, tier-mean
CTR fit on training rows only so the baseline itself is scored out-of-sample. On the held-out,
client-grouped test split: precision@20 = 0.70, @50 = 0.64, @100 = 0.65, ROC-AUC = 0.596 (test base
rate 0.391) — a genuinely strong top-of-list ranker for three multiplied numbers.

## 4. Model / analysis

**Method:** Random Forest, chosen after an earlier full-feature comparison found Logistic
Regression competitive with it (and ahead on some metrics) — Random Forest is used here because it
produced the strongest result on the final, leakage-checked feature set, not because it dominated
every comparison. **Feature list (safe, leakage-checked):** `avg_position`, `content_age_days`,
`days_since_last_update`, `word_count`, `char_count`, `search_volume`, `competition`, `cpc`, plus
content-type/intent/provider metadata and missingness flags. **Left out on purpose:**
`impressions_90d`, `clicks_90d`, `ctr`, `engagement_rate`, `scroll_rate`, `ai_traffic_pct`, and
every other trailing-90-day engagement column — all structurally overlap the label's own 60-day
defining window (r = 0.98 for `impressions_90d` specifically). **Target/proxy, in one sentence:**
`is_declining_label = (trend_direction == "down")`, a backward-looking bucket of a 60-day trailing
impressions comparison — an observed proxy, not a true future-observed outcome (see Limitations in
the paper).

## 5. Evaluation

**Split:** 80/20, **client-grouped** — 6 of 32 clients held out entirely, zero client overlap
between train and test. Tested directly against a naive random row-level split on the same data: the
random split let 31 of 32 clients appear in both sides and inflated precision@20 to 0.95 versus the
honest 0.70 for the identical baseline rule — the model was partly memorizing per-client base rates,
not generalizing. **Metrics, model vs. baseline, same split:**

| Metric | Baseline | Model |
|---|---:|---:|
| Precision@20 | 0.70 | 0.75 |
| Precision@50 | 0.64 | 0.72 |
| Precision@100 | 0.65 | 0.66 |
| ROC-AUC | 0.596 | 0.708 |
| Average precision | 0.476 | 0.550 |

**Error shape:** the model's most confident wrong calls (from the earlier full-feature model,
`w05_model.ipynb`) all shared 1-3 impressions in the whole 90-day window — near-zero-traffic pages
where the label's percentage-based logic reacts to noise, and the model's "not declining" call is
arguably the more defensible one.

## 6. Interpretation

Two independent leakage/interpretation checks agree on the top drivers of decline risk:
`avg_position` and `content_age_days` dominate the safe-feature model, alongside the recomputed
`no_position_data` flag — plausible signal (where a page sits and how old it is), not a
suspiciously-perfect single-feature giveaway. The clearest negative result: staleness alone does
not predict decline monotonically (Section 3) — "this page is old, refresh it" is not a supported
rule in this data, and it is never used as a standalone trigger anywhere in the recommendation
queue.

## 7. Recommendation

Five actions, from two signals kept separate: `refresh_and_review_ctr` (5,092 pages, highest
priority — both signals agree), `refresh_content` (2,408, model risk without a CTR symptom yet),
`review_ctr` (11,966, the largest bucket — a metadata/title problem, cheaper to fix than a rewrite),
`monitor` (9,329, neither signal fired), `no_action_insufficient_data` (1,205, a tracking gap —
route to whoever owns tagging, not to a content reviewer). A FlyRank editor opens the top of
`refresh_and_review_ctr` for their client(s) first, confirms the flagged decline is real (not a
near-zero-traffic artifact), and works down. **Confidence and limits:** precision@50 ≈ 0.72 on
unseen clients is decision-support for prioritization, not a guarantee for any single page — and
nothing in this pipeline should trigger an automated action (see the paper's "never automated"
list).

## 8. Reproducibility

```
git clone https://github.com/720-hz/flyrank-ml-internship.git
cd flyrank-ml-internship && pip install -r requirements.txt
jupyter notebook "work/notebooks/Week 8/capstone.ipynb"
```

Fixed `random_state = 42` throughout every split and model fit. The held-out, client-grouped
validation is re-derived directly from the public CSV in `work/notebooks/Week 8/capstone.ipynb`
(no hand-typed numbers) and matches the committed receipts in
`work/outputs/action_playbook_metrics.json` to 3 decimal places — checked programmatically in the
notebook itself, not asserted. Figures are committed under `work/figures/`. The exploratory
forward-label warehouse work (`w03_data_contract.ipynb`) needs a requested Hugging Face token for
gated access and is not required to reproduce this report's headline numbers.

## 9. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai). Thanks to the internship's skills
library and mentorship framework for the leakage-audit and honest-claims discipline this report
tries to hold itself to.

---

> **Claims checklist before submitting:** observed / measured / directional / decision-support
> language everywhere · no causal claims without an experiment or causal design · no "predicted
> Google's algorithm" · no client-identifying details · numbers in this report match a fresh
> re-run (confirmed programmatically in `work/notebooks/Week 8/capstone.ipynb`, Section 6).
