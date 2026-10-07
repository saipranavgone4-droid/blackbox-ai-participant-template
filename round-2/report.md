# round-2 — Investigate

**Team:** BB-026
**Queries used:** 158 / budget *(fill in your round budget)*

## What we concluded

1. **The system is deterministic.** 20 groups of identical inputs (including 3 repeated across rounds) always returned identical scores. Every difference we saw is caused by an input, not noise.
2. **Decision = a gate on `tenure_years`.** Tenure 10 gives exactly **0.0320 and DECLINE** in two different contexts (R1-37, R1-58). Everything else we varied, even at extremes, only lowered the score and still APPROVED.
3. **At tenure 10 the other inputs stop mattering.** R1-37 and R1-58 differ in `requested_zone` (0 vs 30) and `escorts` (0 vs 1) yet return the identical 0.0320. At tenure 20, the same zone change moves the logit by +0.376 (R1-34 -> R1-47). So the low-tenure branch behaves like a floor or gate that overrides the rest, not an additive penalty.
4. **Six inputs are tuned to a target value** and the score falls on both sides: `badge_age_days` ~24.6-24.7, `history_score` ~349-350, `requested_zone` ~30-31, `linked_badges` ~16.4-16.5, `recent_denials` ~0.4-0.5, `tenure_years` ~22. Ceiling reached: 0.9947.
5. **Three inputs behave as if silently dropped:** `anomaly_ratio`, `clearance_level`, `escorts`. 36 single-field comparisons (6, 13, 17) all gave an exactly unchanged score.
6. **Inputs are combined, not independent.** The same change has a different size in different contexts: `linked_badges` 16.5 -> 20 costs 0.136 logit in R1 contexts but 0.661 in R2 contexts (R1-34->R1-21 vs R2-22->R2-40). Where effects did agree across contexts, they agreed best on the **logit** scale (`recent_denials` 0 -> 2: -0.362 vs -0.366; tenure 10 -> 20: +7.48 vs +7.86), so the score looks like a sigmoid of a latent value.

## How we got there

1. **Fixed the baseline noise question first.** Before reading any effect, we grouped identical inputs. All 20 repeat groups matched exactly, so single comparisons are trustworthy.
2. **Found every pair of queries that differ in exactly one field (586 pairs).** This turns the log into free controlled experiments and shows which fields ever change the score (`experiments/explore.py`).
3. **Checked inert fields on those pairs.** No pair involving anomaly_ratio, clearance_level or escorts changed the score at 4 decimals, even though the same fields sat in contexts with scores from 0.82 to 0.99.
4. **Looked for the same change in different contexts** (`matched_pairs*.py`). This is the cheapest test for combination: if inputs were independent, the effect would not depend on context. Two results mattered: denials behaves consistently on the logit scale, but linked_badges 16.5 -> 20 is 5x larger near the optimum, and zone 0 -> 30 disappears at tenure 10.
5. **Tried to fit global models** (additive in score, additive in logit, multiplicative, Euclidean-distance, with free targets, weights and exponent). None reproduced the log: RMSE 0.0015-0.0025 against a 0.00003 rounding floor, and fitted targets were far from the observed peaks. We did not adopt any of them (see ruled out).
6. **Wrote down discriminating queries for what is left** (`experiments/discriminating_queries.md`), designed so each outcome supports exactly one explanation.

## What we ruled out

- **Measurement noise.** 20 repeat groups, zero disagreements.
- **`anomaly_ratio`, `clearance_level`, `escorts` as score drivers at the values tried** (anomaly 0.5-1.0, clearance 50-100, escorts 0-4.5). Pairs such as R1-9/R1-11, R1-14/R1-15, R1-12/R1-14 and R2-42/R2-43 are identical to 4 decimals.
- **Any non-tenure input causing a decline.** Badge age 75, history 642, zone 52, denials 3.0 and a combined far-off point (R1-75, 0.8213) all stayed APPROVE.
- **Fully additive scoring in score units.** The same perturbation gives very different score changes by context (denials 0 -> 1: -0.0055 vs -0.0007).
- **Fully additive scoring even in logit units.** linked_badges 16.5 -> 20 differs 5x between contexts, and zone's effect vanishes at tenure 10.
- **The tenure drop as a smooth penalty added to the others.** If it were, zone would still shift the tenure-10 score; instead both tenure-10 queries return exactly 0.0320.
- **One simple global formula** (additive, multiplicative, Euclidean, with or without a logistic): all fit poorly, so we make no claim about the exact formula.
- **"More is worse" or "more is better" for denials, history_score and badge age.** All three are peaked.
- **`site` as a decision input.** A-D all approve; D costs ~0.007.

## What we are still unsure about

- **Gate or penalty?** We have two queries at tenure 10 and both give 0.0320. We do not know if *any* other field can move that value (experiment E1).
- **R2-47 puzzle.** Tenure 6.829 scored 0.1859, higher than tenure 10 (0.0320) even though every other field was far from optimum. Either tenure is not monotone below the cutoff, or another field in that query lifted the score (E1, E4).
- **Where the tenure cutoff is, and whether it moves with context** (between 10 and ~20; E2, E3).
- **Are the three "dropped" fields truly dropped?** Maybe they matter only at low tenure, at extreme values, or jointly (for example clearance below zone). Not tested (E6).
- **Which input amplifies the linked_badges penalty** near the optimum (E5).
- **A possible kink at tenure 22.** R2-55 (21.895) scored 0.0014 below R2-54 (22), yet R1 tenure 20 vs 22 differed only 0.0001. Could be a transform or bucketing; one data point (E7).
- **Exact formula.** We only know the logit scale is the better one.
- **Missing log rows.** R1-1..R1-8 are not in our export.

No new queries were spent writing this report; all claims come from re-analysing the 150 logged rows. Items marked E1-E7 are not yet run.# round-2 — Investigate


