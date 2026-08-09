
# Capstone Report — Structured Content Archetype Clustering

- **Author:** Caesar Ghazi
- **Lane:** Structured Content Archetype Clustering
- **Repo:** https://github.com/CaesarGhazi/Flyrank
- **Date:** August 2026

## 0. Abstract

FlyRank's own March 2026 research paper clusters content into five "archetypes" using k-means but admits the naming was heuristic, with three of five segments sharing the identical label "Rising Stars" — this study asks whether a smaller, more disciplined clustering pass can produce archetypes worth trusting, and whether clustering beats a simple human-written rule for prioritizing content review. Using one honest month (March 2026) of real FlyRank warehouse data — 9,841,378 daily rows aggregated to 331,437 content-level records — a KMeans model (k=2, chosen by silhouette score) was validated on a grouped-by-client split and compared against a rule-based baseline built from a confirmed CTR-vs-position signal, the same signal behind FlyRank's own CTR-fix flag. The model separated a small (~1%) high-traffic cluster from a dominant long-tail cluster (held-out silhouette 0.907), while the baseline flagged 25.56% of content as needing a CTR fix, with the two methods agreeing in direction across both archetypes rather than contradicting each other. The output is a ranked action playbook — protect, fix CTR, review-for-prune, or monitor — meant to help a content team decide where to spend limited review time this cycle. All findings are observed and directional decision-support, not predictions of Google's ranking behavior or proof of causal impact.

## 1. Problem framing

**Unit of analysis:** one piece of content, per client, aggregated over March 2026 (client + content, summed/averaged across the daily grain).

**Output:** a cluster assignment (archetype) plus a rule-based action label (`protect`, `fix_ctr`, `review_for_prune`, `monitor`).

**Decision this supports:** which content a team reviews first, instead of treating an inventory of hundreds of thousands of pages as uniformly uncertain.

**Who acts on it:** SEO specialists and content managers deciding where limited editing time goes this cycle.

**Cost of a wrong call:** time spent fixing pages that didn't need it, or a real opportunity — a well-ranked page quietly losing clicks — going unreviewed for another month.

**Why ML helps here:** a fixed rule alone can flag a specific, narrow condition (page-1 rank + weak CTR), but it can't describe the overall shape of a content portfolio — clustering reveals structure (a tiny high-traffic segment vs. a massive long tail) that a single if-statement doesn't surface. Neither method replaces the other; they answer different questions.

## 2. Data safety

**Data used:** `fact_content_daily_performance` (month=2026-03) joined to `dim_content` for `content_type`, from the FlyRank Hugging Face warehouse release.

**Deliberately excluded:**

-   AI-referral columns (`ai_chatgpt`, `ai_claude`, etc.) — confirmed under 0.1% row coverage this month, too sparse to use.
-   Product action flags and date fields (`is_published`, `is_deleted`, `last_optimized_date`, `optimization_eligible_date`) — risk of encoding decisions made _after_ seeing performance (circularity).
-   Any April/June data, and the sealed `fact_content_daily_performance_sample.parquet` final-month table — kept out entirely to avoid future-window leakage.

**Leakage risks considered:** ran a deliberate leakage experiment adding an April-derived `future_clicks` column to the clustering input — the held-out silhouette score did _not_ improve (0.705 with leak vs. 0.720 honest, in an earlier k=4 pass), an honest negative result reported as-is. `client_hash_id` and `content_hash_id` are used only for grouping/joins, never as model features.

**Confirmed:** no client names, real domains, or identifying details appear anywhere in `work/` — all reporting uses aggregate counts and hashed IDs only, and hashed IDs never appear in the deployed paper.

## 3. Baseline

**The rule:** score content 80/`fix_ctr` if it ranks on page 1 (position ≤ 10) and CTR falls below a fixed 0.5% threshold — a cutoff derived from the confirmed CTR-vs-position signal check, not picked arbitrarily. Otherwise, score 20/`monitor`. One reason code throughout: `low_ctr_high_position`.

**Why it's a fair comparison:** built from the same March 2026 content-level data, same features, and evaluated on the same held-out split as the clustering model.

**Numbers:** 25.56% of content flagged `fix_ctr` overall (12,735 of 49,823 held-out rows); flag rate was consistent across both clusters (25.5% in Cluster 0, 29.4% in Cluster 1), showing the rule doesn't just concentrate in one archetype.

## 4. Model / analysis

**Method:** KMeans clustering, k=2 (chosen via silhouette sweep across k=2–7).

**Why it fits:** the lane is unsupervised by nature — there's no ground-truth "archetype" label, only behavioral similarity to group by.

**Features:** `gsc_impressions`, `gsc_clicks`, `gsc_avg_position`, `ga4_engaged_sessions` — all observed, same-window metrics, standardized before clustering.

**Left out on purpose:** `content_type` (explored, not included — numeric behavioral signals drove the final separation), AI-referral columns, and any date/flag field (see Section 2).

**Target/proxy:** none — clustering is unsupervised; the closest comparison point is the baseline's rule output, used only for evaluation, never as a training target.

## 5. Evaluation

**Split:** grouped-by-client (`GroupShuffleSplit`), not random row split — chosen because content from the same client shares scale/behavior, so a random split risks leaking client identity and overstating generalization.

**Metrics vs. baseline, same split:**

| Method |Metric  |Value  |
|--|--|--|
| Baseline (rule) | `fix_ctr` flag rate |25.56%|
| Model (KMeans, k=2) | Silhouette — naive split |0.911|
| Model (KMeans, k=2) | Silhouette — grouped split (honest) |0.907|

**Base rate context:** the baseline's 25.56% flag rate is the relevant "positive rate" here — since this is a rule, not a classifier with a train/test accuracy, the honest comparison is that the grouped split changed the silhouette score by only −0.004 versus the naive split, meaning client identity was not secretly inflating cluster separability.

**Error analysis:** the model can't distinguish _why_ long-tail content underperforms — a zero-impression page and a page with modest but intent-mismatched traffic land in the same cluster despite needing different fixes. That's a real limitation of k=2's coarseness, not a validation failure.

## 6. Interpretation

**Cluster 0 (long tail, ~99%, n≈278,705):** avg. 577 impressions, 1.4 clicks, position 16.3, 0.04 engaged sessions — low visibility, low position, minimal engagement. Ordinary, unremarkable content.

**Cluster 1 (high performers, ~1%, n≈2,909):** avg. 25,653 impressions, 93.8 clicks, position 11.6, 3.22 engaged sessions — a small set of pages driving most portfolio traffic.

**What drove the split:** scale (`gsc_impressions`, `gsc_clicks`) dominates — Cluster 1's impressions are ~44x Cluster 0's; position and engagement differ less dramatically in relative terms.

**Negative result worth keeping:** a signal check found that higher-volume content shows _lower_ average engagement than lower-volume content — the opposite of the "prioritize by traffic" assumption behind quick-win logic. This OPPOSITE verdict is a legitimate, well-evidenced finding, not a failed hypothesis to hide.

## 7. Recommendation

Ranked action queue: **(1) protect + fix CTR** — high-traffic content also underperforming on clicks, highest-value fix. **(2) fix CTR** — long-tail content ranking well but underconverting. **(3) protect** — high-traffic content with no red flags. **(4) review for prune** — zero-impression content, human check required before any action. **(5) monitor** — the long-tail majority, no urgent signal.

**How an editor uses it tomorrow:** start at rank 1, check real impression volume and SERP context before acting, and never auto-prune or auto-rewrite based on the score alone.

**Confidence and limits stated explicitly:** this is one month, one snapshot — it says nothing about April or any future period, and the rule's threshold was tuned on this month's distribution specifically.

## 8. Reproducibility
**To re-run from a fresh clone:**

```bash
git clone https://github.com/CaesarGhazi/Flyrank.git
cd Flyrank
pip install -r requirements.txt
# Open work/notebooks/capstone.ipynb in Colab or Jupyter, set HF_TOKEN, Run All
```

**Random seed:** `random_state=42` used throughout (KMeans, GroupShuffleSplit, np.random.default_rng).

**Sealed/holdout evaluation:** the grouped-by-client test split is built in `work/notebooks/w05_model.ipynb` and `work/notebooks/capstone.ipynb` (see `GroupShuffleSplit` cell); resulting silhouette metrics are committed in `work/outputs/capstone_metrics.json` and `work/outputs/w04_baseline_score_metrics.json`.

## 9. Acknowledgments & data credit

Built on the [FlyRank ML Internship dataset](https://flyrank.ai) — real production search and analytics data, released for hands-on learning.

---

> **Claims checklist before submitting:** all language above is observed/measured/directional/decision-support. No causal claims. No claim of predicting Google's algorithm. No client-identifying details anywhere. Numbers here trace to `work/outputs/*.json` — confirm they match a fresh notebook re-run before final submission.

