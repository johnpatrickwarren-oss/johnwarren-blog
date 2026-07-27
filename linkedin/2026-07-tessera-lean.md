# LinkedIn post — Tessera / SDC at fleet scale (2026-07)

**Visual to attach:** `2026-07-tessera-lean-diagram.png` — "Where the alarm budget holds":
the two fleet-heterogeneity axes (persistent quirk share × noise-scale spread), the green
metered-gate region, the measured breach brackets, the e-BH failure point, and the
fleet-size cascade annotation.

---

On a 72,000-GPU training cluster, the failure that costs the most is the one that never pages: a GPU computing wrong answers while every health check reads green.

TL;DR: Silent data corruption raises no hardware alarm, and hand-tuned dashboard thresholds can't track what "healthy" looks like across a changing fleet. Tessera is our statistical monitoring tier with a defined false-alarm budget. Testing it against the most realistic simulated cluster we could build showed that a guarantee holding everywhere is unrealistic. So we measured where the edges are and built the system to know when it crosses them. Full write-up: https://johnpwarren.dev/blog/tessera-lean/

The gap, from the SRE seat. The standard tier, NVIDIA's DCGM, alarms on hardware-reported fault syndromes: a double-bit memory error, an XID event from the driver. Those alarms are nearly false-positive-free because they are not inferences; the hardware itself is reporting. Silent data corruption produces no syndrome. Meta's Llama 3 team logged 466 job interruptions in 54 days on a 16,000-GPU job; six were silent data corruption. A corrupted-but-running GPU also passes the diagnostic ladder — that is the "silent." So the behavioral gray zone lands on dashboards, where an operator hand-tunes a threshold per signal. On a fleet where healthy spans SKUs, firmware revisions, rack thermals, and a job mix that changes weekly, a tuned threshold is a snapshot of last month's fleet.

What Tessera puts in the gap. Controlled probe workloads run on randomly chosen groups of GPUs at the same moment, so every unit in a group is doing identical work and comparability is manufactured by the design rather than assumed from telemetry. Each unit is scored by its rank among its peers; ranks accumulate into e-values (evidence scores that average at most 1 on a healthy unit, so evidence cannot inflate on its own); a fleet-level selection step called e-BH turns the evidence into cordon decisions with a bound on the false-discovery rate, meaning a cap on the fraction of pulled machines that are actually healthy. It does not replace DCGM: Tessera's confirmation step invokes DCGM's own diagnostics. It is the tier for the machines the diagnostics keep passing.

Then we ran the guarantee against the most realistic cluster we can manage in simulation. clustersynth models the topology, rack thermal structure, firmware mixes, diurnal load, and 14 distinct kinds of drift, over 60-day telemetry streams. Per-round calibration came out exact: false-page rate 0.0100 against a 0.01 target, at 10,000 and at 100,000 simulated GPUs. The guarantee we wanted — an FDR bound that holds anywhere, on any fleet, at any horizon — died in three measurements:

Persistent benign quirks. One rack runs slightly hot, permanently. A healthy GPU on it keeps losing its rank lottery for the same harmless reason, the evidence compounds, and by probe round 320 the healthy fleet had paged 3.3× its budget. Every per-round number stayed exact the whole time.

The noise channel. Units that are persistently noisier at the same average defeat the selection step itself: 14.8 false cordons per run from an all-healthy fleet at the widest noise spread we tested.

Scale. Doubling the simulated fleet twice moved false cordons from 0 to 3 to 26.5 per run, because the noise factor is shared within racks. Superlinear. At 72,000 GPUs, "we're far from the selection threshold" is not an argument.

Where it landed. The bound holds while two fleet-heterogeneity numbers stay in budget: persistent per-unit quirk share under about 4%, and noise-scale spread under about 0.15 (the measured breach brackets sit at 6.3–8.4% and 0.15–0.31). Both numbers are metered on live data by their own estimators, and the moment the fleet leaves budget the code revokes the FDR-bearing status of its outputs — the guarantee has a breaker, not a footnote. Per-round decisions keep a separate design-based guarantee at randomized-trial grade, because the scheduler physically performs the randomization. And the mathematics from the conditions to the conclusion is machine-checked in the Lean theorem prover; the proofs caught five wrong formal statements that 995,245 passing simulations had validated, which is its own story: https://johnpwarren.dev/blog/tessera-lean/

A hand-tuned threshold cannot be patched into this. It has no error-rate semantics: it says "this level," never "at most this fraction of pages is wrong." It has no breaker: when the fleet drifts past whatever it was tuned on, it keeps firing, or keeps silent, with unchanged confidence. And the failure modes above are invisible to tuning: every per-signal marginal stayed clean while the evidence compounded across weeks. There is no level to set when every level looks right.

Tessera is my system, so I am grading my own homework. Every fleet number here is from simulation — as realistic as we could make it, not a production fleet — and the superlinear cascade is measured only to about 4,000 units; where it sets in at 10,000-plus is the current top open item.

If you run fleet health at scale: what catches the GPU that passes every diagnostic — and for the pages that tier sends, what fraction being wrong did you sign up for?
