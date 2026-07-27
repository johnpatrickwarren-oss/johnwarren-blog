---
title: 'Tessera: a machine-checked false-alarm guarantee for GPU fleets'
description: 'Fleet monitors page people on statistical evidence. Tessera''s false-discovery-rate guarantee is now proved end to end in Lean — and the proofs found five wrong statements that 995,245 passing simulations missed.'
pubDate: 'Jul 27 2026'
heroImage: '/og/og_tessera_lean.png'
---

A thousand-shard GPU cluster produces a thousand concurrent health verdicts every tick, and every verdict can page a human. The incumbent tooling avoids false alarms by construction: DCGM-class health checks alarm on hardware-reported fault syndromes (a double-bit ECC error, an XID event), which are nearly false-positive-free because they are not inferences, and everything behavioral gets exported to dashboards where operators tune thresholds per signal. That design has a documented gap — Meta's Llama 3 run logged 466 interruptions in 54 days on monitored infrastructure, six of them silent data corruption, which produces no syndrome at all. Tessera's design goal has been the layer that approach leaves open: statistical verdicts on behavior, with a *defined* false-alarm budget — a false-discovery-rate bound (a cap on the fraction of its alarms that are wrong) holding across all verdicts at once, which is where the [previous post](/blog/tessera/) ended. This post is about what happened when I machine-checked that guarantee in a theorem prover (software that accepts a mathematical claim only when handed a complete formal proof it can verify step by step): the whole chain, from the randomized experiment design at the bottom to the FDR bound at the top.

The result was not new mathematics. Every theorem in the chain was already known. What the prover produced instead was an audit finding: five of my formal statements of that known mathematics were wrong, and the numerical validation I ran on each of them had passed every one.

Tessera is my system; the proofs verify statements I have a stake in. The formal statements were written by Claude sessions transcribing my research notes into Lean, and the proofs that later exposed them were also done with Claude: the same kind of agent wrote the bugs and found them. Every fleet-scale number in the underlying research comes from a simulation substrate, and the proofs do not change that. After searching arXiv, Mathlib, Isabelle's Archive of Formal Proofs, and the adjacent formalization projects I could find, I found no prior machine-checked formalization of this chain or of its components — conformal prediction validity, e-value calibration, e-BH — in any proof assistant.

## The chain

Tessera's strongest guarantee rests on an active-canary design: run a fixed, versioned probe workload on a randomly chosen block of GPUs at the same time, and score each unit by its rank among its peers. The chain from that design to the FDR bound has five links:

| link | statement | where it comes from |
|---|---|---|
| randomization | the healthy block members' scores are exchangeable (any ordering equally likely) | the scheduler physically randomizes placement |
| conformal rank | the randomized rank of each member is exactly uniform on [0,1] | classical conformal prediction |
| calibration | an averaging transform turns that uniform rank into an e-value | e-value theory (Vovk, Wang, Ramdas and others) |
| combination | mixtures and minima of e-values stay e-values | elementary closure properties |
| e-BH | selecting on those e-values controls the false discovery rate, under arbitrary dependence between them | Wang and Ramdas, 2022 |

An e-value is a nonnegative score whose average is at most 1 when nothing is wrong. That single property is what makes evidence from many units combinable without lying about the error rate, and the "arbitrary dependence" clause in the last row is why the approach fits a GPU fleet, where nothing is independent of anything. In pager terms, the chain says: which GPUs get probed together is a lottery; a healthy GPU's rank in its group is pure chance; chance can be banked as evidence across rounds; and the bank pays out cordons with a capped fraction of mistakes.

All five links are now proved in [Lean 4](https://lean-lang.org/), a proof assistant in which every proof is verified mechanically down to the axioms, against [Mathlib](https://leanprover-community.github.io/), its community-built mathematical library (over two million lines, including the measure theory and probability this work stands on). The development is about 1,745 lines, with zero `sorry` placeholders — the keyword Lean uses for "trust me for now." The top-level theorem composes them: given exchangeable block scores and an independent uniform tiebreaker, the calibrated conformal rank is an e-value, and feeding such e-values to e-BH bounds the false discovery rate at the target level. There is no informal step left between the randomization and the bound.

The prover also formalized a result that cuts against the system. Per-round validity does not survive accumulation: a GPU with a persistent quirk (a rack that runs hot, a unit with noisier power delivery) keeps losing its rank lottery for the same benign reason, and multiplying its per-round e-values across weeks pushes the healthy-case average (the null mean) strictly above 1, no matter how exact each round is. That is the slow-burn mechanism behind paging a healthy GPU three weeks into an accumulation window. It was discovered by simulation earlier in the program; it is now a machine-checked theorem sitting next to the guarantee it constrains, and it is why the deployed system meters fleet heterogeneity rather than assuming it away.

## Five wrong statements, one blind spot

Before any statement was written into Lean, my house rule required validating it numerically against the shipped implementation. The rule was followed. The e-BH threshold lemma was checked against 995,245 selections from the production engine across five adversarial input families: zero violations, and a worst-case slack of exactly 0.0, meaning the bound is attained. Rank uniformity was checked exhaustively over every permutation of blocks of size 3, 4, and 5, with the mean of the rank landing on 0.500000 and its second moment on 0.333333, the exact moments of a uniform draw. By every empirical standard the statements were solid.

Then I started proving, and five of them fell over.

| statement | what was wrong | the counterexample |
|---|---|---|
| countable mixture of e-values | missing "the weights are summable" | constant weights of 1 make the hypothesis vacuously true and admit a "mixture" with null mean 2 |
| calibration theorem | missing "the rank is measurable" | without it the conclusion's integral is not even defined |
| super-uniformity definition | quantified over every real α instead of α ≥ 0 | a probability can never be ≤ −1, so nothing satisfies the definition |
| rank super-uniformity | no independence hypothesis at all | couple the tiebreaker adversarially to the scores and a quarter-level event has probability one half |
| the accumulation setup | the conditional mean was a placeholder constant; the i.i.d. assumption lived in a comment typed `True` | three downstream theorems were formally about the placeholder, and said nothing |

The mixture bug is a collision between paper mathematics and formal mathematics. On paper, a divergent sum is undefined, so "the weights sum to at most 1" quietly excludes it. In Mathlib, an infinite sum is a total function that returns 0 when the series diverges — a deliberate convention that keeps the library usable. Under that convention, weights of all 1s "sum" to 0, satisfy the hypothesis, and break the theorem. In the fleet, those weights are how the accumulator prices "the fault may have started at any past round," and they are a config surface: the broken statement licenses a weight configuration that silently doubles the false-cordon rate while every dashboard reads healthy. The shipped weights were geometric and summable, so every simulation drew from the safe case and the gap was invisible.

The missing independence hypothesis is the operational one. The randomized conformal rank uses a uniform tiebreaker, and the statement asserted the alarm-rate property given only the tiebreaker's marginal distribution. But marginals are cheap: a tiebreaker can be perfectly uniform on its own and still coupled to the scores, squeezing each rank into the low half of its cell — and then a threshold meant to fire 25% of the time fires 50% of the time. That is a trap someone can build in an afternoon: seed the tiebreaker's RNG from the telemetry clock, or from anything downstream of the measurements, and you have constructed the counterexample in production. The proof forced independence into the statement, which turns it into a deployment requirement: the jitter's randomness must have no data path from the scores it breaks ties between.

The other three, briefly. The unsatisfiable definition is a spec bug with a familiar shape: a safety property written so that nothing can ever satisfy it, like an alert condition that can never be true — every claim depending on it was a claim about an empty set, and a runtime validity monitor built faithfully from that definition could never have certified any fleet. The placeholder statements were worse in a quieter way: they were the formal coverage of exactly the slow-burn risk above (the persistently hot rack accumulating false evidence), and they formally said nothing, while reading as if they said a lot. The measurability bug is the honest exception — a purely formal defect with no fleet consequence. Four of the five cash out as cluster behavior; one is bookkeeping.

The validation missed all five the same way. Numerical validation runs against the shipped implementation, which instantiates the objects you intend: summable weights, measurable ranks, thresholds between 0 and 1, a genuinely independent random number generator. There is no way to "run" the formal text itself. So the formal statements had two guardians — simulation, which was blind to them by construction, and the type checker, which verifies that a statement is well-formed, not that it is true or even satisfiable. The scaffold's own header contained the sentence "a wrong statement fails loudly at build." All five passed the type checker cleanly. The only tool that reads the statement you actually wrote, rather than the one you meant, is the proof obligation.

Simulation checks your intent. Proving checks your text. The guarantee you publish is the text.

## Is this new mathematics?

No.

The theorems are known. e-BH under arbitrary dependence is Wang and Ramdas. Exact rank uniformity under exchangeability is classical conformal prediction. Calibration of super-uniform ranks into e-values is established e-value theory. The accumulation failure is the tower property plus Jensen's inequality, which is an exercise, not a discovery. A statistician will recognize every result in the repository.

What appears to be new is the formalization. I could find no prior machine-checked treatment of any of these components, let alone the composition. The nearest neighbors are real but distinct: a Lean 4 formalization of statistical learning theory (concentration inequalities, not testing), a 2026 change-point paper with its core approximation theorems in Lean (no e-values, no FDR), and a line of work that uses conformal prediction as a tool *for* verifying autonomous systems, which is the reverse direction despite the colliding vocabulary. Formalizing known mathematics is its own recognized activity — nobody called the Lean proof of Fermat's Last Theorem new number theory — and its value showed up here exactly where the formalization literature says it does: in the statements, not the theorems.

Two small artifacts did come out of the gap between what the proofs needed and Mathlib's coverage: a product rule for Lebesgue integrals over finite product measures that Mathlib only carried in a more restricted form, and a fully worked, tie-robust proof of rank uniformity that handles duplicate scores without any continuity assumption. Both are candidates for upstreaming. Neither is a theorem a probabilist would be surprised by.

## What the guarantee is now

Does Tessera still have an FDR guarantee? The unconditional version — valid for any fleet, any heterogeneity, any horizon — is dead, and machine-checked dead; that is what the accumulation theorem says. What survives is conditional, and the conditions changed character over the course of this program: from silent to named, from named to measured, from measured to enforced. Per-round selections carry the design-based guarantee at the same evidentiary standard as a randomized trial, because the scheduler physically performs the randomization. Accumulated selections carry the guarantee when measured heterogeneity sits inside a budget with two components — a location term and a noise-scale term, each with its own estimator — and the current build revokes the guarantee class at runtime when the measurements leave that budget. The mathematics from the conditions to the conclusion is machine-checked; the conditions themselves are empirical facts about a fleet, and no theorem prover can close an empirical premise. It can only make it visible.

Formal verification turned out to be less about certifying that the system is right and more about finding the exact sentence I am entitled to say: Tessera does not have an unconditional false-discovery guarantee, because no one does; it has a machine-checked conditional one, a meter on every condition, and a breaker that trips when the meter says the guarantee has left the building.

The five wrong statements are public in the [repository history](https://github.com/johnpatrickwarren-oss/tessera), alongside the proofs that replaced them. A test suite exercises the code and never reads the published claim. The one tool that reads the claim is a proof obligation: the first time I pointed one at mine, five statements were wrong.
