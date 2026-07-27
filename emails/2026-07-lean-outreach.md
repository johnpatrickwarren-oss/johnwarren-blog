# Outreach drafts — Lean formalization (2026-07)

Two emails. Ramdas gets the statistics; Degenne gets the formalization. Both link the
public repo (verified public). Send after the blog post is live so [blog] resolves.

---

## Email 1 — Aaditya Ramdas (CMU)

**Subject:** e-BH machine-checked in Lean, inside a deployed e-value system

Prof. Ramdas,

I wrote to you earlier this year about Tessera, a GPU-fleet fault detector built on
e-values and e-BH. Following up with two results I think are closer to your interests
than anything in that first note.

First: we machine-checked e-BH in Lean 4 with Mathlib, including FDR control under
arbitrary dependence, as the final link of a fully formalized chain that starts at the
experiment design: randomized placement, exchangeability, exact conformal rank
uniformity (tie-robust, no continuity assumption), calibration to e-values, then your
procedure. The development is about 1,700 lines, sorry-free, and public:
https://github.com/johnpatrickwarren-oss/tessera (lean/). After searching arXiv,
Mathlib, and Isabelle's AFP, I believe it is the first formalization of e-BH, and of
conformal validity, in any proof assistant. Corrections welcome if you know of prior
work.

Second, and more likely to be useful to your group: the same program measured two ways
the guarantee degrades in deployment, both with the per-round construction exact. With
persistent unit heterogeneity, accumulated per-unit e-processes drift; the inflation
E[M_T] = E[g^T] is now a machine-checked theorem in the same repository, and the
operational failure is first-passage (paging rate), which the mean bound badly
mispredicts. Separately, under persistent noise-scale heterogeneity we measured e-BH
itself producing false selections from all-healthy fleets: a persistently noisy unit
concentrates its inflation and crosses the selection threshold alone, and because the
scale factor is shared within racks, false selections grew superlinearly with fleet
size in our experiments (0, then 3, then 26.5 per run as N doubled twice). The
dependence-robustness theorem is intact; the inputs stop being e-values in a correlated
way. All of this is on a simulation substrate, stated as such in the reports
(research/ in the same repo).

One sentence on a side effect you may enjoy: proving the chain exposed five defects in
our own formal statements that 995,245 numerical checks had passed, including an
independence hypothesis whose absence has a clean adversarial-coupling counterexample.

No reply expected. If any of it is useful to you or a student, it's public.

John Warren

---

## Email 2 — Rémy Degenne (Inria Lille)

**Subject:** Conformal validity + e-BH formalized in Lean; two possible Mathlib pieces

Dr. Degenne,

Your Markov-kernel and disintegration work in Mathlib turned out to be exactly the
machinery I needed, so this is partly a thank-you note. I formalized the validity chain
of a deployed statistical system (GPU-fleet fault detection): exchangeability plus an
independent jitter gives exact tie-robust conformal rank uniformity, calibration gives
e-values, and e-BH gives FDR control under arbitrary dependence. The conditional-i.i.d.
accumulation model in the second half is built directly on `Kernel` and `Measure.compProd`.
About 1,700 lines, Lean 4.32.1, sorry-free, public:
https://github.com/johnpatrickwarren-oss/tessera (lean/). I could find no prior
formalization of conformal prediction or e-BH in any proof assistant; given your FORMAL
project's aims this seemed worth flagging to you specifically.

Two pieces may be worth upstreaming, and I'd value your judgment on whether they fit:
a lintegral product rule over finite product measures (Mathlib carries the Bochner
version, integrability-gated; the lintegral version is side-condition-free), and the
tie-robust rank uniformity proof, whose deterministic core is a clean Finset argument
(tie-class blocks tile [0, K+1) and unit clamps telescope).

The part of the exercise your community may find most useful as a case study: proving
exposed five defects in formal statements that extensive numerical validation had
passed, each a known transcription trap (junk-value tsum hypotheses, ambient
measurability, quantifier domains, `True`-typed placeholder hypotheses, placeholder
definitions that hollow out downstream theorems). Write-up: [blog].

Happy to open Mathlib PRs for either piece if they look wanted.

John Warren
