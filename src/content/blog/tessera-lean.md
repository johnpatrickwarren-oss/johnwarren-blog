---
title: 'Tessera: the proofs found five bugs the simulations passed'
description: 'We machine-checked the statistical guarantee chain behind Tessera in Lean. No new mathematics came out. Five wrong formal statements did.'
pubDate: 'Jul 27 2026'
heroImage: '/og/og_tessera_lean.png'
---

The [previous Tessera post](/blog/tessera/) ended on e-BH, the false-discovery-rate procedure that keeps a thousand concurrent shard verdicts from drowning an operator in false alarms. This post is about what happened when we machine-checked that machinery in a theorem prover: the whole chain, from the randomized experiment design at the bottom to the FDR bound at the top.

The result was not new mathematics. Every theorem in the chain was already known. What the prover produced instead was an audit finding: five of our formal statements of that known mathematics were wrong, and the numerical validation we ran on each of them had passed every one.

Tessera is my system; the proofs verify statements I have a stake in. The formal statements were written by Claude sessions transcribing our research notes into Lean, and the proofs that later exposed them were also done with Claude: the same kind of agent wrote the bugs and found them. Every fleet-scale number in the underlying research comes from a simulation substrate, and the proofs do not change that. After searching arXiv, Mathlib, Isabelle's Archive of Formal Proofs, and the adjacent formalization projects I could find, I found no prior machine-checked formalization of this chain or of its components — conformal prediction validity, e-value calibration, e-BH — in any proof assistant.

## The chain

Tessera's strongest guarantee rests on an active-canary design: run a fixed, versioned probe workload on a randomly chosen block of GPUs at the same time, and score each unit by its rank among its peers. The chain from that design to the FDR bound has five links:

| link | statement | where it comes from |
|---|---|---|
| randomization | the healthy block members' scores are exchangeable | the scheduler physically randomizes placement |
| conformal rank | the randomized rank of each member is exactly uniform on [0,1] | classical conformal prediction |
| calibration | an averaging transform turns that uniform rank into an e-value | e-value theory (Vovk, Wang, Ramdas and others) |
| combination | mixtures and minima of e-values stay e-values | elementary closure properties |
| e-BH | selecting on those e-values controls the false discovery rate, under arbitrary dependence between them | Wang and Ramdas, 2022 |

An e-value is a nonnegative score whose average is at most 1 when nothing is wrong. That single property is what makes evidence from many units combinable without lying about the error rate, and the "arbitrary dependence" clause in the last row is why the approach fits a GPU fleet, where nothing is independent of anything.

All five links are now proved in [Lean 4](https://lean-lang.org/) against Mathlib, in about 1,745 lines, with zero `sorry` placeholders — the keyword Lean uses for "trust me for now." The top-level theorem composes them: given exchangeable block scores and an independent uniform tiebreaker, the calibrated conformal rank is an e-value, and feeding such e-values to e-BH bounds the false discovery rate at the target level. There is no informal step left between the randomization and the bound.

The prover also formalized a result that cuts against the system. Per-round validity does not survive accumulation: when units carry persistent quirks, multiplying their per-round e-values across rounds inflates the null mean strictly above 1, no matter how exact each round is. That limitation was discovered by simulation earlier in the program; it is now a machine-checked theorem sitting next to the guarantee it constrains.

## Five wrong statements, one blind spot

Before any statement was written into Lean, our house rule required validating it numerically against the shipped implementation. The rule was followed. The e-BH threshold lemma was checked against 995,245 selections from the production engine across five adversarial input families: zero violations, and a worst-case slack of exactly 0.0, meaning the bound is attained. Rank uniformity was checked exhaustively over every permutation of blocks of size 3, 4, and 5, with the mean of the rank landing on 0.500000 and its second moment on 0.333333, the exact moments of a uniform draw. By every empirical standard the statements were solid.

Then we started proving, and five of them fell over.

| statement | what was wrong | the counterexample |
|---|---|---|
| countable mixture of e-values | missing "the weights are summable" | constant weights of 1 make the hypothesis vacuously true and admit a "mixture" with null mean 2 |
| calibration theorem | missing "the rank is measurable" | without it the conclusion's integral is not even defined |
| super-uniformity definition | quantified over every real α instead of α ≥ 0 | a probability can never be ≤ −1, so nothing satisfies the definition |
| rank super-uniformity | no independence hypothesis at all | couple the tiebreaker adversarially to the scores and a quarter-level event has probability one half |
| the accumulation setup | the conditional mean was a placeholder constant; the i.i.d. assumption lived in a comment typed `True` | three downstream theorems were formally about the placeholder, and said nothing |

The mixture bug is a collision between paper mathematics and formal mathematics. On paper, a divergent sum is undefined, so "the weights sum to at most 1" quietly excludes it. In Mathlib, an infinite sum is a total function that returns 0 when the series diverges — a deliberate convention that keeps the library usable. Under that convention, weights of all 1s "sum" to 0, satisfy the hypothesis, and break the theorem. Our intended weights were geometric and summable, so every simulation drew from the safe case and the gap was invisible.

The missing independence hypothesis is the one with real statistical content. The randomized conformal rank uses a uniform tiebreaker, and the statement asserted super-uniformity given only the tiebreaker's marginal distribution. But marginals are cheap: you can build a tiebreaker that is perfectly uniform on its own and adversarially coupled to the scores, squeezing each rank into the low half of its cell. With one peer, the probability of landing at or below one quarter becomes one half, double what the guarantee permits. Independence is where the randomization earns its keep, and the formal statement had left it out entirely — while a `True`-typed placeholder in a neighboring theorem made the file look fully hypothesized.

The validation missed all five the same way. Numerical validation runs against the shipped implementation, which instantiates the objects you intend: summable weights, measurable ranks, thresholds between 0 and 1, a genuinely independent random number generator. There is no way to "run" the formal text itself. So the formal statements had two guardians — simulation, which was blind to them by construction, and the type checker, which verifies that a statement is well-formed, not that it is true or even satisfiable. The scaffold's own header contained the sentence "a wrong statement fails loudly at build." All five of these elaborated cleanly. The only tool that reads the statement you actually wrote, rather than the one you meant, is the proof obligation.

Simulation checks your intent. Proving checks your text. The guarantee you publish is the text.

## Is this new mathematics?

No.

The theorems are known. e-BH under arbitrary dependence is Wang and Ramdas. Exact rank uniformity under exchangeability is classical conformal prediction. Calibration of super-uniform ranks into e-values is established e-value theory. The accumulation failure is the tower property plus Jensen's inequality, which is an exercise, not a discovery. A statistician will recognize every result in the repository.

What appears to be new is the formalization. I could find no prior machine-checked treatment of any of these components, let alone the composition. The nearest neighbors are real but distinct: a Lean 4 formalization of statistical learning theory (concentration inequalities, not testing), a 2026 change-point paper with its core approximation theorems in Lean (no e-values, no FDR), and a line of work that uses conformal prediction as a tool *for* verifying autonomous systems, which is the reverse direction despite the colliding vocabulary. Formalizing known mathematics is its own recognized activity — nobody called the Lean proof of Fermat's Last Theorem new number theory — and its value showed up here exactly where the formalization literature says it does: in the statements, not the theorems.

Two small artifacts did come out of the gap between our needs and Mathlib's coverage: a product rule for Lebesgue integrals over finite product measures that Mathlib only carried in a more restricted form, and a fully worked, tie-robust proof of rank uniformity that handles duplicate scores without any continuity assumption. Both are candidates for upstreaming. Neither is a theorem a probabilist would be surprised by.

## What the guarantee is now

Does Tessera still have an FDR guarantee? The unconditional version — valid for any fleet, any heterogeneity, any horizon — is dead, and machine-checked dead; that is what the accumulation theorem says. What survives is conditional, and the conditions changed character over the course of this program: from silent to named, from named to measured, from measured to enforced. Per-round selections carry the design-based guarantee at the same evidentiary standard as a randomized trial, because the scheduler physically performs the randomization. Accumulated selections carry the guarantee when measured heterogeneity sits inside a budget with two components — a location term and a noise-scale term, each with its own estimator — and the current build revokes the guarantee class at runtime when the measurements leave that budget. The mathematics from the conditions to the conclusion is machine-checked; the conditions themselves are empirical facts about a fleet, and no theorem prover can close an empirical premise. It can only make it visible.

Formal verification turned out to be less about certifying that the system is right and more about finding the exact sentence we are entitled to say: we do not have an unconditional false-discovery guarantee, because no one does; we have a machine-checked conditional one, a meter on every condition, and a breaker that trips when the meter says the guarantee has left the building.

The five wrong statements are public in the [repository history](https://github.com/johnpatrickwarren-oss/tessera), alongside the proofs that replaced them. If you maintain a system whose value rests on a mathematical claim, the uncomfortable question this exercise leaves you with is the one it left me with: your tests check what your code does. What checks what your claim says?
