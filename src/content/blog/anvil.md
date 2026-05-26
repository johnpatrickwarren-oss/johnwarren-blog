---
title: 'Anvil: the deploy gate run backward'
description: 'How the DeploySignal engine adapts to verdict chaos experiments — same Ville-bounded portfolio, same audit substrate, inverted operational question.'
pubDate: 'May 28 2026'
---

The [DeploySignal post](/blog/deploysignal/) framed the engine as answering one operational question: should this canary proceed, extend, or rollback? The [Tessera post](/blog/tessera/) showed the same engine running continuously across N shards in a cluster.

This is the third operational direction. Anvil runs the same engine against chaos experiments — and the polarity of "fire = bad" gets inverted along the way.

This is the fourth post in the series, and the shortest. The engine is already explained in the DeploySignal post; the methodology context is already established. The thing distinctive about Anvil is the *operational inversion*, and that's what this post is about.

## The chaos-verdict gap

Chaos-engineering platforms — Gremlin, Chaos Mesh, AWS FIS, Litmus — are good at injecting faults. They schedule the experiment, run it against the target, and produce a structured record of what happened. What they don't produce is a structured *verdict*: did the system behave acceptably under the injected fault?

The verdict is left to an operator eyeballing dashboards. Did the latency spike recover in time? Did the dependent service degrade gracefully? Did anything *unexpected* happen during the fault window? The pass/fail call gets written into a postmortem document by hand, often days later, often by someone who wasn't watching the run live.

This is the same gap deploy-gate tooling closed for deploys, run in the opposite direction. A deploy gate asks: "did the canary degrade compared to the baseline? If yes, rollback." A chaos experiment asks: "did the system degrade *only in the way I expected* during the injected fault? If yes, the experiment passed; if it degraded in an unexpected way, the experiment failed and revealed a real weakness."

The structural similarity is the load-bearing observation. Both are verdict problems against an anytime-valid statistical portfolio with formal false-positive guarantees. The difference is the operator's prior: in a deploy, you're hoping the canary looks like the baseline; in a chaos experiment, you're hoping for a specific, declared deviation.

[Anvil](https://github.com/johnpatrickwarren-oss/deploysignal/tree/main/engine/o0/anvil) is the packaging that makes the engine run in that second direction.

## The expected_failure_pattern contract

The key data structure is an `ExpectedFailurePattern` declared at experiment-start. The operator commits, before the fault is injected, to what the fault should do.

```
ExpectedFailurePattern {
  kind:             "latency_injection",
  affected_signals: ["p99_latency"],
  magnitude:        0.30,            // expected to rise by ≤30%
  magnitude_unit:   "fractional",
  recovery_seconds: 60,              // expected to recover within 60s post-fault
  suppress_families: ["A_p99_latency"],
}
```

The contract does two things at once. First, it gives the verdict a hypothesis to test against — "did p99 rise by ≤30% within the fault window and recover within 60s of fault end?" Without a pre-declared expectation, the verdict has nothing to verify against; it's just measurement.

Second, it declares which detector families the *expected* fault would otherwise fire. In the latency-injection example, Family A's per-signal sequential test on `p99_latency` is going to fire when latency spikes — that's the fault doing its job. Suppressing `A_p99_latency` for the duration of the fault window means the verdict doesn't read "rollback" simply because the operator's intentional perturbation worked.

The verdict vocabulary maps from DeploySignal's native one:

| DS verdict | Anvil verdict |
|---|---|
| `proceed` | `experiment_passed` |
| `rollback` | `experiment_failed_unexpectedly` |
| `extend` | `experiment_still_running` |

The translation happens at the adapter boundary; the engine emits its native vocabulary, and the adapter rewrites it per `DeployContext.strategy === 'chaos_experiment'`. The Ville-bound on the α-participating portfolio holds exactly as it does in the canary direction. No detector math changes. Anvil is packaging, contracts, and vocabulary — not new statistics.

## Family suppression and the unexpected blast

The structural point of suppression is to preserve the engine's ability to fire on what the operator *didn't* expect, while staying quiet about what they did.

In the latency-injection example: Family A on `p99_latency` is suppressed. Family A on every other signal — `downstream_err`, `tool_success_rate`, `eval_score` — is still running. Family C (joint vector) is still running. If the latency injection causes an *additional* downstream error spike, Family A on `downstream_err` fires. If the latency injection causes p99 and error rate to move in an unusually correlated way (a joint-distribution shift that no single-signal test would catch), Family C fires.

Either of those firings produces `experiment_failed_unexpectedly` — not because the operator's intended fault didn't happen, but because the system revealed a coupling the operator hadn't accounted for. That is the chaos engineer's actual signal of interest. The whole point of injecting faults is to find out what the system does *other* than the intended thing.

The Anvil v1 reference profile enables Family A (per-signal mean shift) and Family C (joint-vector distance) only; Families B, D, and E are off. Family B's structural rules don't transfer to generic chaos targets — they were tuned for LLM-inference signatures. Families D and E need more calibration history than a typical chaos experiment provides at v1. The α allocation is 70/30 between A and C, total budget 1e-3 — same anytime-valid guarantee as in the deploy-gate direction.

## The four adapters

Anvil ships four chaos-platform adapters under `engine/o0/anvil/`:

```
engine/o0/anvil/
├── gremlin.ts
├── chaos-mesh.ts
├── chaos-mesh-translate.ts
├── aws-fis.ts
├── litmus.ts
├── suppression.ts
├── types.ts
└── index.ts
```

Each adapter implements two methods: `fetchDeployContext(experiment_ref)` (the standard orchestration-adapter contract) and `fetchExpectedFailurePattern(experiment_ref)` (the chaos-specific extension that pulls the experiment's declared parameters from the platform's API and shapes them into the `ExpectedFailurePattern` schema).

At v1, these ship as typed stubs with provenance docstrings citing the upstream platform's experiment-definition API. The contracts are real; the end-to-end wire-up to each platform's API surface is deferred to follow-on cycles. The bet being made: get the contract shape right, then wire each platform to it. Validating that bet requires operator partnership or substantive further build-out — neither of which is in scope for the v1 positioning play.

This is the honest framing of where Anvil sits today. The engine is real. The audit substrate is real. The verdict translation is real. The four platform adapters are typed stubs. The reference profile (`profiles/anvil-chaos-experiment.yaml`) is real. The end-to-end demonstration of a single platform integration end-to-end is a downstream cycle.

## The DS-Anvil composition

Anvil isn't a single component; it's a composition. DS-Anvil draws on three:

1. The DeploySignal engine at the verdict layer (Ville-bounded portfolio + audit substrate)
2. [Tessera](/blog/tessera/) at the per-shard observation layer
3. The chaos-platform adapter family (Gremlin / Chaos Mesh / AWS FIS / Litmus)

Tessera matters for chaos in a specific way. A typical chaos experiment targets a topology subset: pod-kill on shard-04, network-partition on rack-2, latency-injection on a tenant subset. The fault is, by construction, topology-localized. Tessera's per-shard residual semantics and topology-aware grouping already handle exactly this — and Tessera's bi-directional integration with DeploySignal (`engine/ds-integration/` HTTP contract) means the chaos event flows into Tessera's freeze-hook naturally: Tessera knows to suppress per-shard alerts on shards inside the declared fault window.

Without Tessera, a cluster-scope chaos experiment generates noise on every shard the operator didn't target. With Tessera, the verdict correctly says "shard-04 degraded as expected; the rest of rack-2 stayed within their hierarchical baselines."

That composition is what makes DS-Anvil a coherent thing rather than three repos you'd have to manually integrate. The pieces share the same statistical substrate and the same contracts — wiring them up by hand would duplicate work the contracts already do.

## Close

Anvil currently lands as positioning + contracts + adapter stubs + reference profile. The engine substrate is real; the verdict translation is real; the bundle composition is real. The four end-to-end platform integrations are not.

The four-product picture comes into focus here. DeploySignal answers "should this deploy proceed?" Anvil answers "did this chaos experiment fail in the way I said it would?" Tessera answers "is this shard behaving normally?" Cairn — coming up next — answers "what caused this incident?" Four operational questions, one statistical engine, one audit substrate, one methodology.

The methodology angle on Anvil is briefer than on the prior two posts because the disciplines have been told. The PRD lives in `coordination/PRD-29-anvil.md`. The spec lives in `coordination/Q29-ANVIL-CHAOS-VERDICT-SPEC.md`. The anti-scope is enumerated explicitly (no per-experiment detector retraining, no chaos-platform-side authoring UX, no live multi-tenant production chaos at v1, no fifth platform yet). The audit-record extension is replay-clean. The Ville-bound holds. The standard pattern.

The next post will be about Cairn, which closes the lifecycle loop in the original direction: when a regression escapes all three other systems and lands in production, Cairn ranks candidate cause-events against incident onset under a Bayesian alignment model. Different math again, same substrate, same methodology.
