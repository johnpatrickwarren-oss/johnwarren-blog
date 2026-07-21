---
title: 'Three passes, one fail: pre-registering the validation of a capacity planner'
description: 'We froze the endpoints, fetched public data, and published the miss along with the hits — what external validation of planning statistics actually looks like, and what a failed endpoint taught us that a passing one could not.'
pubDate: 'Jul 06 2026'
heroImage: '/og/og_runway.png'
---

[Runway](/blog/runway/) is a capacity-planning tool for AI compute whose pitch depends on statistics: bootstrap confidence bands on demand forecasts, empirical-Bayes slip priors on delivery schedules, Monte Carlo runway distributions. When a tool's value proposition is "calibrated uncertainty," the only honest question a buyer can ask is: *calibrated against what?*

For most planning tools the answer is a test suite, and this post starts with why that answer is worth less than it sounds. Runway had 527 passing tests and 95% coverage the day an adversarial audit found that its 80% confidence band actually covered about 54%, its "empirical-Bayes" estimator gave a single noisy observation 99.8% of the posterior weight while the UI claimed the value was "pulled toward the industry prior," and its backtest was quietly reading delivery dates recorded *after* the as-of cutoff. Every test was green. Not one test simulated a process where the true answer was known and checked the estimator against it. Tests verify that code runs; they don't verify that math is true.

The remediation story — a truth-known simulation harness, mutation-checked so every guarantee test provably fails when its fix is reverted — is a post of its own. This one is about what happened after the internal statistics were sound: testing them against reality, with the endpoints frozen in advance.

## The protocol: endpoints first, data second

The design borrows from clinical-trial pre-registration, for the same reason trials use it: when success criteria are negotiable after the results exist, they get negotiated. The rules are short:

1. **The pre-registration commits before any data is fetched.** Selection *rules*, not hand-picked series; metrics; pass/fail thresholds; naive baselines to beat. Once the commit lands, nothing moves.
2. **Raw data freezes with hashes.** Analysis reads frozen copies; run manifests record the engine's git SHA and the data hashes.
3. **Results are append-only.** A rerun is allowed only for a code defect, fixed test-first, with the defect named in the superseding run's manifest — and the old run stays.
4. **Every endpoint publishes, pass or fail.**

Two sub-studies ran under this protocol against public data. Sub-study B tested the empirical-Bayes slip estimator against [LBNL's Queued Up dataset](https://emp.lbl.gov/queues) — 3,367 completed power-infrastructure projects with both proposed and actual commercial-operation dates, which is a real, heavy-tailed slip distribution (median slip zero, ninetieth percentile nearly a year). Sub-study A tested demand forecasting and band calibration against OpenRouter's daily token-volume rankings — nineteen months of real GenAI demand.

## The results, all four of them

**B1 — does shrinkage beat raw group means on held-out data? Pass.** Total held-out squared error 4.4% below the raw-group-mean predictor, with the improvement concentrated exactly where the theory says: small noisy groups shrink hard, large clean ones barely move.

**B2 — do the predictive intervals calibrate? Pass.** 82.0% of 779 held-out project slips landed within the ±1.28σ band, against a nominal 80%.

**A2 — does the forecast beat naive baselines? Pass**, on the one eligible cell (the eligibility rules were frozen; per-model series were too young, so one cell is what the rules produced — the report says so rather than pretending otherwise).

**A1 — does the 80% band cover held-out demand? Fail. Zero of six months.**

The failure is the interesting part. The 2026 token-volume surge was a demand regime break; the pre-registered model-selection rule — pick the growth model with the best fit on the recent past — chose a linear model immediately before super-linear growth, and every held-out month landed above the band. A post-hoc analysis (clearly labeled, no verdicts attached) reran the same cell forcing each growth model: linear covered 0/6, piecewise 0/6, **exponential 5/6 with a quarter of the error**. The band mechanics were fine. The single-model selection rule was the failure — it rewarded fitting the recent past, which is precisely the wrong criterion at the moment a regime turns.

Two things came out of that failure that no passing endpoint could have produced. First, honest product language: Runway's confidence bands are now labeled *within-regime* uncertainty statements on every surface that draws them — they do not claim to anticipate regime changes, and the anomaly detector is documented as the compensating control that tells you when you've left the regime. Second, a design response: a confidence-weighted **ensemble** growth model — every member keeps a floor weight, replicates pool, and the forecast is the pooled median — whose validation armor includes a synthesized version of exactly this failure. On that synthetic regime break the ensemble's band covers 0.858 where the failed selection rule covers 0.163, and mutation checks confirm both the floor and the proportional weighting are load-bearing. Whether the ensemble beats the old rule *out of sample* is itself pre-registered: a forecast diary scores both rules side-by-side on data that did not exist at registration time, and the ensemble is promoted to default only if the diary says so.

## Replication, because one dataset is one dataset

The estimator family at the center of all this — the post-audit empirical-Bayes design — has now been tested against three independent real-world datasets: the LBNL power-project slips, per-tenant GPU reservations from [Alibaba's PAI cluster trace](https://github.com/alibaba/clusterdata/tree/master/cluster-trace-gpu-v2020), and per-subscription VM requests from [Microsoft's Azure public trace](https://github.com/Azure/AzurePublicDataset). Three domains, three clouds, one design, one question: do the ±1.28σ predictive intervals hold near their nominal 80%?

| Dataset | Domain | Held-out coverage (nominal 80%) |
|---|---|---|
| LBNL Queued Up | power-project slip | 0.820 |
| Alibaba PAI 2020 | GPU reservations | 0.802 |
| Azure VM 2019 | VM core requests | 0.787 |

The replications inherit their endpoints verbatim from the original registrations — that's what makes them replications — and their reports carry the construct caveats up front (the Azure trace only records lifetime-average utilization, so its headline error-reduction number is not comparable across studies; the pass is, the magnitude isn't).

## What this buys

A capacity planner is a machine for turning statistics into capital decisions, and the failure mode of the genre is confident numbers nobody ever checked. The claim this work supports is narrow and, I think, the right one: every statistical guarantee Runway states is either validated against truth-known simulation, validated against public real-world data under frozen endpoints, or labeled with exactly what it doesn't cover — and the one endpoint that failed is published next to the ones that passed, with the mechanism and the design response.

The [repository is now public](https://github.com/johnpatrickwarren-oss/runway) — the pre-registrations, the run manifests, the failed endpoint, the amendment notes, and the studies are all in the history where they can be checked rather than believed. If you plan GPU capacity for a living and want to see whether this survives contact with *your* demand series: that's the conversation I'm hoping this post starts.
