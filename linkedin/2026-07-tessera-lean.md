# LinkedIn — Tessera / SDC at fleet scale (2026-07)

Two artifacts. The FEED POST (≤3,000 chars) is the reach vehicle: attach
`2026-07-tessera-lean-promise.png`, put the blog link in the post text, drop
`2026-07-tessera-lean-diagram.png` (the edges map) in the first comment. The ARTICLE
(title: **A false-alarm budget for GPU fleets**) is the long piece below the divider;
use the promise image as its cover and embed the edges map at the "Where it landed"
section.

---

## FEED POST (2,096 chars — limit 3,000)

On a 72,000-GPU training cluster, a GPU can compute wrong answers for weeks while every health check reads green.

Silent data corruption raises no hardware alarm: NVIDIA's DCGM alarms on hardware-reported syndromes, and SDC by definition produces none. Meta's Llama 3 team logged 466 interruptions in 54 days; six were silent corruption that passed diagnostics. The behavioral gray zone lands on dashboards where operators hand-tune thresholds — and a tuned threshold is a snapshot of last month's fleet.

Tessera is my statistical tier for that gap. Randomized probe groups make GPUs directly comparable; ranks accumulate into e-values; a selection step (e-BH) issues cordons with a defined false-discovery budget: at most, say, 5 of 100 cordons wrong. Not a level you tune — an error rate you set, metered live, revoked in code when the fleet leaves the regime where the math holds.

Testing against the most realistic simulated cluster I could build (clustersynth: to 100,000 simulated GPUs, 60-day streams) killed the guarantee I wanted — one that holds anywhere, always:

• A rack that runs benignly hot: healthy GPUs paged 3.3× their budget by round 320, with every per-round number exact.
• Persistently noisier-but-healthy units: 14.8 false cordons per run.
• Doubling the fleet twice: false cordons went 0 → 3 → 26.5. Superlinear.

So I measured the edges, built estimators that meter them live, and machine-checked the math from conditions to conclusion in the Lean theorem prover — which found five wrong formal statements that numerical validation had already passed.

Measured record when the guarantee is live: 0 wrong cordons in 46 validation runs. All simulation, stated as such; the 10,000-plus cascade onset is the open item.

Hand-tuned thresholds can't be patched into this: no error-rate semantics, no breaker, and the failure modes above never move a per-signal marginal.

If you run fleet health at scale: what catches the GPU that passes every diagnostic — and what fraction of wrong pages did you sign up for?

Full write-up: https://johnpwarren.dev/blog/tessera-lean/

---

## ARTICLE — A false-alarm budget for GPU fleets

On a 72,000-GPU training cluster, a GPU can compute wrong answers for weeks while every health check reads green.

TL;DR: Silent data corruption raises no hardware alarm, and hand-tuned dashboard thresholds can't track what "healthy" looks like across a changing fleet. Tessera is my statistical monitoring tier with a defined false-alarm budget. The most realistic simulated cluster I could build showed a guarantee holding everywhere is unrealistic. So I measured the edges and built the system to know when it crosses them. Full write-up: https://johnpwarren.dev/blog/tessera-lean/

The gap, from the SRE seat. NVIDIA's DCGM, the standard tier, alarms on hardware-reported fault syndromes: a double-bit memory error, an XID event. They are nearly false-positive-free: the hardware is reporting, not inferring. Silent data corruption produces no syndrome. Meta's Llama 3 team logged 466 interruptions in 54 days on a 16,000-GPU job; six were silent data corruption. A corrupted-but-running GPU passes the diagnostic ladder — that is the "silent." The behavioral gray zone lands on dashboards, where operators hand-tune thresholds per signal. On a fleet where healthy spans SKUs, firmware, rack thermals, and a shifting job mix, a tuned threshold is a snapshot of last month's fleet.

An undetected unit's cost scales with fleet-hours. In training, a corrupting GPU poisons gradients; the damage surfaces later, if at all, as a loss anomaly, and the fix is rollback plus bisection. At $2 per GPU-hour, rolling a 72,000-GPU job back six hours is about $860,000 of recompute before the debugging starts. In inference there is no loss curve: the fleet serves some fraction of wrong answers, and nothing downstream knows.

What Tessera puts in the gap. Controlled probe workloads run on randomly chosen groups of GPUs at the same moment: every unit does identical work, so comparability is manufactured by the design rather than assumed from telemetry. Each unit is scored by its rank among peers; ranks accumulate into e-values (scores that average at most 1 on a healthy unit, so evidence cannot inflate on its own); a selection step called e-BH turns evidence into cordons with a bounded false-discovery rate: a cap on the fraction of pulled machines that are actually healthy. It does not replace DCGM: Tessera's confirmation step invokes DCGM's own diagnostics. It is the tier for the machines diagnostics keep passing.

The other defenses are screens and symptom watchers. Known-answer screens catch a defect only if it misbehaves during the test window: Meta's Fleetscanner reaches each machine every 45–60 days; Ripple squeezes tests between workloads; on GPUs the equivalent is a scheduled dcgmi diag sweep. Symptom watchers — NaN checks, loss-spike alerts, checkpoint replay — fire after the damage is in the model. Anomaly detectors over telemetry exist, with tuned sensitivities and no statement about how often they are wrong. As far as I could find, Tessera is the only system that turns a fleet's behavioral signals into statistical evidence with a defined false-discovery budget.

The budget is what turns alarms into decisions. Staffing: at a 5% budget, at most five of every hundred cordons waste a technician's visit, on average — a queue you can size; a threshold stream's wrongness rate is unknown. Automation: cordon, diagnostics, and RMA filing can be wired directly to discoveries because the wrong-action rate is capped; an unbounded alert stream always ends in a human. Sensitivity: detection reaches persistent shifts of about 1% in a probe metric without alert fatigue, because pushing sensitivity does not move the false-page rate; a threshold trades sensitivity against noise until operators de-tune it into silence. And ranking outlives the bound: even with the guarantee revoked, e-values still order every unit by accumulated evidence, so the dispatch queue stays principled. Ranking also collapses the bisection: when a job's loss anomaly fires, the evidence already points at the suspects, and cordoning the top of the list replaces days of drain-and-bisect — one skipped bisection covers years of the 0.005% probe tax.

The compute bill is smaller than it reads. Probes are the only GPU-side cost: three sub-second micro-workloads, about half a second per unit every 2–4 hours — a 0.005% duty cycle, the same class of test tax fleets already pay (Meta's Ripple runs 2.5 billion co-located executions a month). The statistics run on CPUs: a full detector pass over 72 hours of 1 Hz telemetry for 2,304 units (a 10 GB bundle) took 31 seconds on a Mac mini in 2.4 GB of memory, flat as the fleet grows.

Then I ran the guarantee against the most realistic cluster I can manage in simulation: clustersynth models topology, rack thermals, firmware mixes, diurnal load, and 14 kinds of drift over 60-day streams. Per-round calibration came out exact: false-page rate 0.0100 against a 0.01 target, at 10,000 and 100,000 simulated GPUs. The guarantee I wanted — an FDR bound that holds anywhere, on any fleet, at any horizon — died in three measurements:

Persistent benign quirks. One rack runs slightly hot, permanently. A healthy GPU on it keeps losing its rank lottery for the same harmless reason; the evidence compounds, and by round 320 the healthy fleet had paged 3.3× its budget. Every per-round number stayed exact the whole time.

The noise channel. Units that are persistently noisier at the same average defeat the selection step itself: 14.8 false cordons per run from an all-healthy fleet at the widest noise spread I tested.

Scale. Doubling the fleet twice moved false cordons from 0 to 3 to 26.5 per run, because the noise factor is shared within racks. Superlinear. At 72,000 GPUs, "we're far from the selection threshold" is not an argument.

Where it landed. The bound holds while fleet diversity stays inside a measured budget, on two axes. Persistent quirks: at most about 4% of a unit's behavioral variance may be its permanent signature (its rack, its silicon) rather than shared noise; in testing, the budget failed between 6.3% and 8.4%. Noise spread: a typical unit's noise level must sit within about ±16% of the fleet median; failures began between ±16% and ±36%, and the e-BH collapse above hit at about 1.8×. Both numbers are metered live, and the moment the fleet leaves budget the code revokes the FDR-bearing status of its outputs. Per-round decisions keep a design-based guarantee at randomized-trial grade: the scheduler physically performs the randomization. And the mathematics from the conditions to the conclusion is machine-checked in Lean, a proof assistant: a language whose compiler accepts a theorem only when every logical step checks, with Mathlib, its community-built mathematics library. I went that far because the alarm budget is a mathematical claim and tests cannot vet it: they exercise the code, never the claim. The proofs caught five wrong formal statements that numerical validation had already passed — its own story: https://johnpwarren.dev/blog/tessera-lean/

A hand-tuned threshold has none of this: no error-rate semantics, no breaker. When the fleet drifts past whatever it was tuned on, it keeps firing, or keeps silent, with unchanged confidence. And the failure modes above are invisible to tuning: every per-signal marginal stayed clean while evidence compounded across weeks; there was no signal to set a threshold on.

Tessera is my system, so I am grading my own homework. Every fleet number here is from simulation — as realistic as I could make it, not a production fleet — and the superlinear cascade is measured only to about 4,000 units; where it sets in at 10,000-plus is the top open item.

If you run fleet health at scale: what catches the GPU that passes every diagnostic — and for the pages that tier sends, what fraction being wrong did you sign up for?
