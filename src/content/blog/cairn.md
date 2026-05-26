---
title: 'Cairn: ranked attribution, not root cause'
description: 'How Cairn ranks candidate cause-events against incident onset under a Bayesian alignment model — and the disciplines that distinguish attribution from causation.'
pubDate: 'May 29 2026'
---

The first four posts in this series walked forward through the lifecycle: [DeploySignal](/blog/deploysignal/) at promotion, [Anvil](/blog/anvil/) for chaos experiments, [Tessera](/blog/tessera/) for steady-state observation. Each runs the engine in a different operational direction.

Cairn answers the question that comes when all three were insufficient: the regression escaped, landed in production, and now the operator has to figure out what caused it.

The current state is dashboards, a postmortem document, and judgment. Operators reconstruct the timeline by hand, decide what caused it by experience, and write the call into the postmortem doc. The decision is judgment-loaded — when multiple things happened in the same window, the "which one was it" question doesn't have an obvious answer, and the narrative-first approach tends to commit to a cause before alternatives are weighed.

[Cairn](https://github.com/johnpatrickwarren-oss/cairn) answers a narrower question: given an incident definition (onset time + affected signals + optional engine-inferred onset distribution) and a set of candidate cause-events, rank them by timing alignment, per-kind prior, and evidence quality. The output is a ranked candidate list with cited evidence — not a narrative, not a "root cause."

The honesty discipline lands here in the opening, because it's the most important thing about Cairn: this is ranked alignment, not root cause. The discipline boundary is the product distinction.

## The postmortem-attribution gap

Deploy-gate tooling closed one dashboard-eyeballing gap (DeploySignal makes the verdict structured and audit-clean). Tessera closed another (steady-state shard behavior gets a verdict surface). Anvil closed a third (chaos experiments get a verdict, not an operator's qualitative pass/fail).

Postmortem attribution is the fourth dashboard-eyeballing gap. Today's pattern: an incident fires. The operator pulls up Grafana, the deploy dashboard, the change-log, the chaos history, and the on-call notes. They look for things that happened "around the same time as the incident." They notice three or four candidates. They pick the one that feels most likely — usually the deploy, because deploys are the most legible thing in the operator's mental model. The rest get written off in the postmortem with a sentence or two.

The failure mode is specific. Timing alignment is hard to do honestly without explicit math. Per-kind temporal scales are different — a deploy correlates with incidents tightly; an environment change can propagate over hours; a shard event is sharply localized. The narrative-first approach commits to a cause before weighing alternatives. And the evidence — the DS audit record, the Tessera VerdictGroup, the Anvil experiment definition — is often invisible at postmortem time because nobody pulls it into a single view.

Cairn's role isn't to author the narrative. The narrative is still human work. Cairn surfaces a ranked-and-cited candidate list with the math made explicit, *before* the narrative gets written. The human still does the synthesis. They just do it with a defensible ranking rather than a vibe.

## The Bayesian alignment math

The scoring model:

```
s(c) = K(Δt, σ_kind) × π(kind) × e(c)
posterior(c) = s(c) / Σ s(c')
```

Three components, each doing distinct work.

**K — Gaussian timestamp-alignment kernel.** For each candidate, compute the time delta between the candidate's timestamp and the incident onset. Map it through a Gaussian kernel with a per-cause-kind bandwidth:

| Cause kind | Kernel σ |
|---|---|
| deploy | 30 min |
| chaos experiment | 5 min |
| dependency | 2 hr |
| env change | 6 hr |
| shard event | 15 min |
| generic | 1 hr |

These aren't arbitrary. A deploy has a known timestamp and most deploy-induced regressions surface within the canary window — thirty minutes is the empirically reasonable σ. A chaos experiment is bounded by its fault window — five minutes σ matches the typical experiment duration. An environment change (config rollout, feature flag flip) can propagate over hours — six hours σ. The per-kind shape is what lets the math weight a deploy sixty minutes before incident as plausible and a chaos experiment sixty minutes before incident as implausible-or-suppressed.

**π — per-kind prior.** The operator's base-rate belief that this kind of event is the typical cause for their system. A team that ships ten times a day has a high deploy prior; a team that runs continuous chaos has a high chaos prior. Defaults exist; operator-tunable per profile.

**e — evidence quality boost.** The most distinctive component. When a candidate has associated DS audit evidence, that evidence shifts the score:

- DS verdict on the candidate was `proceed` (clean engine output): downweight to `e = 0.5`. The engine emitted clean — that's *negative evidence* against this candidate being the cause.
- DS verdict was `extend` (engine wanted more time): upweight to `e = 1.5`. The engine was uncertain.
- DS verdict was `rollback` and the operator overrode to `extend` anyway: upweight to `e = 2.0`. The engine said no, the human said yes, and the regression happened.

The third case is interesting. It's the most common "we thought we could get away with it" failure mode. Cairn's evidence boost mechanically captures the fact that a rollback-overridden deploy carries the strongest prior toward being the cause.

**Engine-inferred onset preference.** When DS provides an engine-inferred onset distribution — a Page-CUSUM fire-tick plus a confidence band — it supersedes the operator-supplied point onset. The kernel σ combines the engine's onset uncertainty with the per-kind bandwidth via quadrature. The operator's point estimate is preserved as a fallback; when the engine knows more precisely when the regression started, the engine's estimate wins.

**Mechanistic-inconsistency suppression.** Candidates whose timestamp is *after* the incident onset (plus a grace window) are excluded from the posterior with `suppression_reason: 'post_incident_timestamp'`. A thing that happened after the incident can't have caused the incident; the math shouldn't pretend otherwise.

**The worked example.** From Cairn's canonical demo:

| Candidate | Posterior | Why |
|---|---|---|
| deploy | 80.7% | Tight timing alignment + DS `extend`-overridden evidence (e = 2.0) |
| env_change | 14.7% | Within env kernel σ but weaker timing fit |
| shard event | 4.6% | Small kernel σ; misaligned timestamp |
| chaos experiment | suppressed | 90-min lag vs 5-min chaos kernel σ — kernel underflow |

The chaos suppression is the math at work. A chaos experiment that ran ninety minutes before incident onset is, statistically, no longer plausible as a cause under the five-minute chaos kernel σ — the probability density at 18σ from the mean is effectively zero. Suppression with a stated reason (`kernel_underflow`) is more honest than ranking it at "0.00001%" and forcing the operator to scroll past it.

## The honesty disciplines

Cairn's load-bearing differentiator from generic RCA tools is four explicit non-claims.

**Ranked alignment-based attribution, not causal inference.** The output language is "ranked attribution of timing-consistent candidates," never "root cause." Pearl-style counterfactual causal inference requires a known causal graph: a model of which events can cause which other events, with intervention semantics. Cairn does not have this. It ranks correlation-of-timing under a known set of candidates. Mislabeling the output as "causal" would be a honesty-breach. The distinction matters because postmortem culture has been corrupted by tools that claim to find "root cause" when they're actually pattern-matching on logs.

**Cairn does not author the postmortem narrative.** The output is structured ranked data plus cited evidence. The narrative — the chain of mechanisms, the lessons learned, the action items — is still human work. Cairn's role is to make the candidate ranking and evidence pulling deterministic, so the narrative author can focus on the parts that require human judgment.

**One incident at a time at v1.** Multi-incident batch RCA and cross-incident pattern detection are future work. Cairn at v1 ranks candidates for a single incident definition. The reason for the boundary is honest: cross-incident pattern detection introduces statistical questions (correlated false positives, prior contamination across incidents) that the v1 math doesn't address.

**No live production telemetry at v1.** Cairn ships against synthetic fixtures plus the audit JSONL produced by existing DS / Tessera / Anvil demos. Operating on live production data requires data-sovereignty and isolation work that isn't in scope for the v1 release.

The pattern across the four boundaries is the same as DS's "the gate emits a verdict, not a root cause" and Tessera's "shard behavior, not hardware diagnosis": the engine deliberately doesn't claim more than the math supports. The discipline of refusing to overclaim is what makes the underclaim trustworthy.

This is the part that's hardest to ship in a postmortem tool because operators *want* a root cause. They want the postmortem doc to say "deployment X caused incident Y" with confidence. Cairn instead says "deployment X has 80.7% posterior weight under this alignment model; here is the cited evidence; the rest is your call." It's less satisfying to the asker. It's more defensible to the auditor.

## The four-product lifecycle

With Cairn in place, the four operational questions are:

- DeploySignal: should we promote this deploy?
- Anvil: did this chaos experiment fail in the way we declared it would?
- Tessera: are these shards behaving normally during steady state?
- Cairn: what's the ranked attribution for this incident?

One statistical substrate, four operational questions, four contracts at the boundaries.

The audit-stream consumption pattern is the point. Cairn reads:

| Source | What Cairn reads |
|---|---|
| DS audit JSONL | per-deploy verdict + α-budget consumption + per-cell baseline ref |
| Tessera VerdictGroup feed | per-shard observations + freeze-hook events |
| Anvil ExpectedFailurePattern | chaos-experiment definitions + fault windows |
| Generic external events | incident-mgmt webhook payloads, env-change feeds, operator-supplied JSON |

No separate observability layer required. The other three products already emit structured audit records as a byproduct of operating; Cairn is the consumer of those records at postmortem time. The contracts at the boundaries — DS audit JSONL schema, Tessera VerdictGroup type, Anvil ExpectedFailurePattern type — are typed and replay-clean. Given the same inputs, Cairn produces the same ranked attribution, byte-identical. The CLI ships with a `--check` flag that re-runs against a saved expected output and compares.

The composition pays back at postmortem time. When the operator asks "what caused incident Y?" the data is already there: DS verdicts, Tessera observations, Anvil experiment definitions, operator-supplied environment changes. Cairn surfaces it ranked. Without the audit substrate, the postmortem author would be reconstructing each of those sources by hand, hours after the fact, often days later. With it, the data flows through the bundle's existing contracts and lands in a defensible ranking.

## Methodology

Cairn was built using Anchor from round 1, same pattern as Tessera. The methodology angle is brief because the disciplines have been told across the other posts in this series.

One concrete methodology note worth surfacing. Cairn originally landed inside the DeploySignal repo at `engine/cairn/*` via DS PR #21, merged on 2026-05-21. It was extracted to its own sibling repo a few days later for architectural consistency with the rest of the bundle.

The extraction was itself an architectural-reality discovery in the same shape as Tessera's R61. The discovery: a postmortem-attribution layer with its own input contracts and its own math doesn't sit comfortably inside the deploy-gate codebase. The right answer was a sibling, not a subdirectory. Anchor's discipline made the extraction ship cleanly — the contract surfaces between Cairn and DS were already typed and audit-clean, so the extraction was structural rather than rewrite work. The work that had been done inside `engine/cairn/*` traveled to the new repo intact.

That pattern — methodology designed for clean handoffs makes structural extractions cheap — is a derivative benefit of audit-substrate continuity that doesn't show up until you do an extraction. Worth naming because it's not obvious from the methodology articles alone.

## Where the series goes from here

This closes the project deep-dive series. Five posts, five projects, one methodology:

- [Anchor](/blog/three-roads-to-the-same-harness/) — methodology essay
- [DeploySignal](/blog/deploysignal/) — product + methodology emergence
- [Tessera](/blog/tessera/) — product + methodology applied from round 1
- [Anvil](/blog/anvil/) — product + operational inversion
- Cairn — product + lifecycle closure

The substrate is mostly built out. What's next is a different direction: AI infrastructure capacity planning, specifically runway reporting for service owners, planners, and finance teams. Different problem domain (forecasting demand vs. detecting deviation), different stakeholders (FP&A and infra leadership vs. SRE and oncall), and different statistical work (calibrated supplier-slip uncertainty rather than anytime-valid detection). The methodology travels; the substrate doesn't.

I'll write about that when there's enough built to deep-dive on. The deep-dive shape works when the project has a real artifact behind it. The next posts in this series will get to the new work as the work gets to a state worth describing.
