# round-2 — Investigate

**Team:** BB-026
**Queries used:** 158 / budget *(fill in your round budget)*

## What we concluded

GK-05 behaves like a scoring function made of **independent "sweet-spot" terms plus one hard cliff on `tenure_years`**.

- **Decision:** the only thing we found that flips APPROVE to DECLINE is low `tenure_years`. With every other field fixed, tenure 22 or 20 gives 0.9885 / 0.9832 (APPROVE) and tenure 10 gives 0.0320 (DECLINE). The cutoff is between 10 and 20.
- **Score:** six features have a preferred value, and the score falls as you move away from it:

  | Feature | Preferred value |
  |---------|-----------------|
  | `badge_age_days` | ~24.6-24.7 |
  | `history_score` | ~349-350 |
  | `requested_zone` | ~30-31 |
  | `linked_badges` | ~16.4-16.5 |
  | `recent_denials` | ~0.4-0.5 |
  | `tenure_years` | ~22-22.5 |

- **Minor:** `site` is a small offset (A ~ B > C > D).
- **No effect found:** `anomaly_ratio`, `clearance_level`, `escorts`.
- **Combining deviations:** penalties stack. The score ceiling we reached is 0.9947 with all sweet spots set together.

Features that are far from their targets lower the score a lot without causing a decline (e.g. R1-75 scored 0.8213 and was still APPROVED), so the score is not the decision boundary by itself. Tenure is.

## How we got there

1. **Started from a poor baseline (R1-9 to R1-11).** Badge age 46.5, history 600, escorts 3, clearance 50, score 0.9410. This gave us a point with room to improve and let us see which fields moved the score.
2. **Toggled suspicious-sounding fields (R1-9 vs R1-11, R1-10).** Anomaly ratio 0.5 vs 1.0 gave an identical score, so we stopped expecting it to matter. Denials 0 -> 1 lowered it slightly (0.9410 -> 0.9355).
3. **Walked fields toward better scores one at a time (R1-12 to R1-36).** Dropping badge age to 18 gave the first big jump (0.9410 -> 0.9658). History 600 -> 300 and zone 50 -> 0 followed. Clearance, escorts and site changes did nothing visible, which classified them as inert or minor.
4. **Found the cliff (R1-37).** Tenure 20 -> 10 on an otherwise unchanged query gave 0.0320 and DECLINE. We replicated it at R1-58 in a different context (same score, same decision), confirming it was tenure and not luck.
5. **Mapped each sweet spot with sweeps.** Zone (R1-42 to R1-52), history (R1-66 to R1-72), denials (R1-78 to R1-81), site (R1-62 to R1-65), tenure (R1-54 to R1-57). Each sweep gave a rise then a fall, which is why we call them peaked rather than monotonic.
6. **Checked an extreme combination (R1-74, R1-75).** Several fields far off at once scored 0.8213 and still APPROVED, which showed penalties add up and that a low score alone does not cause a decline.
7. **Round 2: refined around the optimum.** Badge age 18.5 -> 24.7 (R2-4 to R2-18) raised the score 0.9915 -> 0.9945, and R2-16 (26 days) showed the peak. Small tweaks to linked badges, denials, and zone (R2-19 to R2-33) checked how flat the peak was. R2-34 to R2-41 re-tested single fields at the new optimum to confirm earlier effects held.
8. **Near-optimum fine tuning (R2-48 to R2-69).** Tiny changes (e.g. 24.595 vs 24.685, history 349 vs 349.585) moved scores by only 0.0001-0.0004, which is how we found the ceiling at 0.9947.
9. **Random probe (R2-47).** A far-off point with tenure 6.8 returned DECLINE at 0.1859, consistent with the tenure cliff.

## What we ruled out

- **`anomaly_ratio` as a driver.** 0.5 vs 1.0 gave identical scores at two different baselines (R1-9/R1-11: 0.9410; R2-40/R2-42 for 0.775 vs 1: 0.9894). Not tested below 0.5 apart from R2-47.
- **`clearance_level` as a driver.** 50 vs 100 gave identical scores twice (R1-14/R1-15: 0.9658; R1-22/R1-27: 0.9830). Values 99, 99.5, 100 also matched.
- **`escorts` as a driver.** 0, 1, 2, 3 and 4.5 gave the same scores at matched baselines (R1-12/R1-14, R1-53/R1-54, R2-42/R2-43).
- **"More is worse" for `recent_denials`.** Zero denials scored lower than 0.4-0.5 (R1-78 0.9901 vs R1-81 0.9915), so it is not monotonic.
- **"Higher `history_score` is better".** The score peaks at ~349-350 and falls both sides, steeply above (600: -0.017, 642: -0.029).
- **"Older badge is worse".** The score improves up to ~24.7 days before falling.
- **`site` as a decision driver.** Sites A to D all approved (R1-62 to R1-65, R2-44); D cost only ~0.007.
- **Any non-tenure feature causing a decline by itself.** Badge age 75, history 642, zone 52, denials 3.0 all lowered the score but stayed APPROVE.
- **A smooth, gradual path to decline.** Tenure 20-25 is smooth and high, while tenure 10 collapses to 0.032, so the behavior looks like a cliff.

## What we are still unsure about

- **Where the tenure cutoff is.** Only 10 (decline) and 20+ (approve) were tested. We never probed 11-19.
- **What happens below 10.** R2-47 (tenure 6.8) scored 0.1859, higher than the 0.0320 at tenure 10. Either other features in that query lifted the score, or tenure is non-monotonic below the cliff. We could not separate these since R2-47 changed everything at once.
- **Whether the cliff depends on other fields.** We only tested tenure 10 with otherwise good settings (R1-37, R1-58) and in R2-47. We did not test whether a very good profile can survive low tenure.
- **Whether "inert" fields are truly inert.** We found no effect in tested ranges, but extremes (0, max) were not all covered, and interactions with low tenure or odd site values were not tested.
- **Interactions in general.** Almost everything was one-factor-at-a-time. The stacking of penalties is an inference from a few multi-change queries, not a measured model.
- **Exact sweet-spot positions.** They are bounded by the values we sampled, especially `recent_denials` (nothing tested between 0 and 0.4) and `linked_badges`.
- **Missing early data.** R1-1 to R1-8 are not in our export, so we have not re-checked what those queries showed.
- **R1-74 vs R1-75.** Badge age 46.5 -> 46.7 raised the score +0.0008 on the far side of the peak. We have a single pair, so we can't tell if this is noise or a real local wiggle.
