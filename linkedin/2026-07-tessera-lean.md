# LinkedIn post — Tessera Lean formalization (2026-07)

**Visual to attach:** one simple diagram, two panels. Left panel, "what simulation checks":
the chain randomization → rank → e-value → e-BH with green checks, caption "995,245 runs,
0 violations." Right panel, "what the prover reads": the same chain with the five formal
statements above it, three marked with red X, caption "5 statements wrong." (SVG request
same style as the runway/ballast diagram.)

---

A GPU-fleet monitor pages a human on statistical evidence. Ours now carries a machine-checked bound on how often that page is false.

TL;DR: We proved the false-discovery-rate guarantee behind Tessera (our GPU-fleet fault detector) in the Lean theorem prover, end to end — from the randomized probe design to the alarm bound. No new math came out. What came out: five wrong formal statements that 995,245 passing simulations had validated as fine, and a sharper statement of what a fleet's alarm budget can actually promise. Full write-up: https://johnpwarren.dev/blog/tessera-lean/.

The chain we proved runs from experiment design to error control: randomized probe placement gives exchangeability (any ordering of healthy peers is equally likely), which makes each unit's rank among its peers exactly uniform, which a calibration step turns into an e-value (a score that averages at most 1 when nothing is wrong), which the e-BH procedure turns into a false-discovery-rate bound that holds under arbitrary dependence. Every theorem in that chain was already known. As far as I could find, none had ever been machine-checked, in any proof assistant.

Why did five wrong statements survive a million checks?

One example. A mixture theorem needed its weights to be summable. On paper you'd never state that; a divergent sum is undefined, so "weights sum to at most 1" excludes it silently. In Lean's math library, an infinite sum is a total function that returns 0 when the series diverges. Under that convention, weights of all 1s "sum" to zero, satisfy the hypothesis, and admit a fake e-value with average 2. Our real weights were geometric and safe, so every one of our simulations drew from the safe case. The gap was invisible by construction.

Another: a statement about our randomized tiebreaker required only that it be uniformly distributed. Marginals are cheap. A tiebreaker can be perfectly uniform on its own and adversarially correlated with the scores, and then a threshold that should fire 25% of the time fires 50% of the time. The independence assumption existed in our heads and in a code comment typed `True`. The prover does not read comments.

The pattern behind all five: simulation checks your intent, because it runs the objects you actually built. Proving checks your text, because it reads the statement you actually wrote. The guarantee you publish is the text.

Tessera is my system, so I am grading my own homework here. The original formal statements were written by AI transcribing our research notes, and the proofs that caught them were AI-assisted too — the same kind of agent wrote the bugs and found them. The fleet-scale numbers behind the system come from a simulation substrate, not production hardware. The prover also machine-checked a limitation: our per-round validity does not survive naive accumulation across rounds. The end state is a conditional guarantee with every condition named, measured by a runtime monitor, and revoked in code when measurements leave budget.

If you maintain a system whose value rests on a mathematical claim, here's the question it left me with, for anyone who has faced it: your tests check what your code does — what checks what your claim says?

Full write-up: https://johnpwarren.dev/blog/tessera-lean/
