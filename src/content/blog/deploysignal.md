---
title: 'DeploySignal: a statistical deploy gate, and the methodology that fell out of building one'
description: 'What DeploySignal does, how it works, and how building it produced the Anchor methodology and the substrate for Tessera.'
pubDate: 'May 26 2026'
heroImage: '/og/og_deploysignal.png'
---

The standard deploy-gate question is also the hardest one in production AI inference: given a deploy and live telemetry, should we proceed, extend, or rollback?

The tools that answer it today — Spinnaker Kayenta, Argo Rollouts, Flagger, Harness Continuous Verification — operate by comparing summary statistics between a canary population and a stable one. They work. They fail in a specific way on AI-inference workloads: regressions in this domain are often multi-modal and slow-developing. A refusal rate creeps up by 6%. A tool-call success rate dips on one cohort. Latency tails widen without the median moving. Each signal stays within its own threshold; the joint pattern is what's broken. By the time the dashboard alerts fire, the regression has shipped at scale and the rollback is expensive.

[DeploySignal](https://github.com/johnpatrickwarren-oss/deploysignal) answers a narrower version of the same question with statistical rigor: given a stream of multi-signal telemetry from a deploy and its baseline, with formal anytime-valid guarantees on the false-positive rate, fire a verdict. Five detector families run in parallel; each contributes evidence at a different mathematical layer; portfolio fusion produces the decision.

This post is about what DeploySignal is technically, and about what four weeks of building it as a five-role multi-agent project produced as a side effect: the methodology I later published as [Anchor](/blog/three-roads-to-the-same-harness/), and the engine substrate that [Tessera](https://github.com/johnpatrickwarren-oss/tessera) is now built on top of.

## The technical core

### The deploy-gate problem

A deploy goes out. The dashboard shows green on all the obvious aggregates: p50 latency, error rate, success rate. Over the next several hours, the long-tail behavior degrades — refusal rate climbs gradually, tool-call coherence weakens on a specific tenant tier, a periodic oscillation appears in cache pressure that wasn't there before. None of these signals exceed their own thresholds. The compound pattern is the regression.

The single-statistic deploy gate is structurally incapable of catching this. It compares one summary to another, computes a p-value or t-statistic, and decides. The class of failure where each signal looks fine in isolation but the joint distribution has shifted is exactly the failure mode that summary-statistic gates miss — by construction.

The narrower question DeploySignal answers: given a continuous telemetry stream and a healthy-baseline reference, can we fire a verdict that holds up under formal anytime-valid guarantees, while catching multi-modal slow-developing regressions that summary-statistic gates miss?

### Anytime-valid α and why it matters

Most deploy-gate tools have a fixed α threshold (typically 0.05) and an implicit "check at the end of the canary window" semantics. If you peek mid-window — and operators always peek mid-window — you inflate the false-positive rate. If you don't peek, you can't adapt the canary duration based on what the signal is doing.

DeploySignal's detectors are *anytime-valid*. The wealth statistic for each family is a supermartingale under the null hypothesis (deploy is healthy). By Ville's inequality, the probability that the statistic ever crosses a Ville bound during the entire monitoring window is at most α. Operators can read the wealth statistic at every tick without consuming additional α from the budget. The bound holds regardless of stopping rule.

Concretely: an α budget is allocated per family at compile time. Family A gets 40%, Family C gets 20%, Families D and E get 10% each. Family B is rule-based and consumes no α. The total stays within the global budget. Every tick, the audit record tracks α consumption per family.

This is the difference between "you can look once, at the end" and "you can look continuously, and the worst case is bounded." Operationally, the second is what operators actually want — they want to extend the canary window when the wealth statistic is drifting suggestively but hasn't crossed, and short-circuit when it has. Anytime-valid is the math that lets you do that without lying about your false-positive rate.

### The five detector families

The portfolio is five families running in parallel every tick. Each family contributes evidence under a different statistical model. Any one firing triggers rollback. When multiple families fire on the same event, you get multi-family corroboration — independent statistical tests agreeing.

| Family | Mathematical anchor | Catches | α-budget |
|---|---|---|---|
| A | Page-CUSUM mixture-supermartingale + mSPRT | Per-signal slow drift | 40% |
| B | Hand-tuned structural rules | Known regression archetypes | 0% (rule-based) |
| C | Hotelling T² + Sequential MMD betting e-process | Joint-distribution shift | 20% |
| D | Spectral / autocorrelation + BOCPD | Oscillation patterns | 10% |
| E | Weighted-conformal Mahalanobis novelty | Unknown-unknowns | 10% |

**Family A — Per-signal sequential tests.** A statistical accumulator that keeps a running tally of log-likelihood-ratio evidence against the healthy hypothesis. Each tick adds evidence; when the accumulator crosses a Ville bound, it fires. Family A catches signals that *creep* — slow persistent drifts on a single signal that stay below any tick-by-tick threshold but accumulate enough evidence over time to cross the cumulative bound. Six SLIs watched: p99 latency, time-to-first-token, eval score, tool success rate, downstream errors, cost per request. Largest single family in the portfolio.

<figure class="family-figure">
<svg viewBox="0 0 400 280" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Family A visual intuition">
<!-- Top panel: signal with noise + drift -->
<text x="10" y="16" font-size="11" fill="#1c1a17" opacity="0.7" font-weight="600">signal value</text>
<line x1="10" y1="20" x2="390" y2="20" stroke="#1c1a17" opacity="0.18" stroke-width="0.5"/>
<!-- Baseline line -->
<line x1="30" y1="80" x2="390" y2="80" stroke="#1c1a17" opacity="0.35" stroke-dasharray="2,2" stroke-width="0.5"/>
<text x="32" y="76" font-size="9" fill="#1c1a17" opacity="0.55">baseline</text>
<!-- Threshold line (dashed red) -->
<line x1="30" y1="40" x2="390" y2="40" stroke="#8a3324" stroke-dasharray="3,3" stroke-width="0.8"/>
<text x="32" y="36" font-size="9" fill="#8a3324">ratio threshold (cascade)</text>
<!-- Signal line - noisy around baseline, slowly drifting up but stays below threshold -->
<polyline points="30,82 50,78 70,81 90,75 110,79 130,74 150,78 170,72 190,75 210,70 230,73 250,68 270,71 290,66 310,69 330,64 350,67 370,62 390,65" fill="none" stroke="#3d5b3f" stroke-width="1.5"/>
<!-- Bottom panel: CUSUM statistic -->
<text x="10" y="160" font-size="11" fill="#1c1a17" opacity="0.7" font-weight="600">CUSUM S_n</text>
<line x1="10" y1="164" x2="390" y2="164" stroke="#1c1a17" opacity="0.18" stroke-width="0.5"/>
<!-- Zero line -->
<line x1="30" y1="250" x2="390" y2="250" stroke="#1c1a17" opacity="0.35" stroke-width="0.5"/>
<text x="32" y="248" font-size="9" fill="#1c1a17" opacity="0.55">0</text>
<!-- Ville threshold line -->
<line x1="30" y1="180" x2="390" y2="180" stroke="#8a3324" stroke-dasharray="3,3" stroke-width="0.8"/>
<text x="32" y="176" font-size="9" fill="#8a3324">h = -log(α) threshold</text>
<!-- CUSUM curve - accumulating, crossing threshold around x=290 -->
<polyline points="30,250 50,248 70,245 90,240 110,234 130,227 150,219 170,210 190,199 210,187 230,174 250,180 250,180" fill="none" stroke="#3d5b3f" stroke-width="1.5"/>
<polyline points="30,250 50,248 70,245 90,240 110,234 130,227 150,219 170,210 190,199 210,187 230,174 250,159 270,142 290,125" fill="none" stroke="#3d5b3f" stroke-width="1.5"/>
<!-- Fire marker -->
<circle cx="290" cy="125" r="4" fill="#8a3324" stroke="#f3efe6" stroke-width="1.5"/>
<text x="300" y="120" font-size="10" fill="#8a3324" font-weight="600">FIRE</text>
<line x1="290" y1="128" x2="290" y2="160" stroke="#8a3324" stroke-width="0.5" stroke-dasharray="2,2"/>
<text x="200" y="275" font-size="9" fill="#1c1a17" opacity="0.55" text-anchor="middle">time →</text>
</svg>
<figcaption>Signal (top) stays below any single-tick threshold. Cumulative CUSUM (bottom) crosses Ville's bound and fires.</figcaption>
</figure>

**Family B — Structural signatures.** Hand-tuned rules encoding the kind of pattern an experienced SRE would recognize at a glance. The canonical rule is *slowbleed*: when 4 or more of 9 monitored signals drift in the same direction simultaneously, fire. Each signal's drift alone is harmless; the synchronized pattern is the architectural signature. Family B is the expert-system layer beneath the statistical ones — rule-based, no α spent, because these are hand-tuned heuristics rather than statistical tests.

<figure class="family-figure">
<svg viewBox="0 0 400 280" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Family B visual intuition">
<!-- Title -->
<text x="10" y="16" font-size="11" fill="#1c1a17" opacity="0.7" font-weight="600">9 monitored signals — slowbleed rule: "4+ drift together"</text>
<!-- Bars for 9 signals, each with slight upward or downward drift -->
<!-- Each bar: vertical gray ruler showing baseline, then colored bar showing drift -->
<g transform="translate(30, 40)">
<!-- p99_latency: drifting up (matches rule) -->
<line x1="0" y1="10" x2="340" y2="10" stroke="#1c1a17" opacity="0.18"/>
<text x="-5" y="14" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="end">p99_latency</text>
<rect x="200" y="4" width="35" height="12" fill="#3d5b3f" opacity="0.8"/>
<text x="237" y="14" font-size="9" fill="#3d5b3f" font-weight="600">↑</text>
<!-- ttft: drifting up -->
<line x1="0" y1="35" x2="340" y2="35" stroke="#1c1a17" opacity="0.18"/>
<text x="-5" y="39" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="end">ttft</text>
<rect x="200" y="29" width="30" height="12" fill="#3d5b3f" opacity="0.8"/>
<text x="232" y="39" font-size="9" fill="#3d5b3f" font-weight="600">↑</text>
<!-- tokens_turn: drifting up -->
<line x1="0" y1="60" x2="340" y2="60" stroke="#1c1a17" opacity="0.18"/>
<text x="-5" y="64" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="end">tokens_turn</text>
<rect x="200" y="54" width="32" height="12" fill="#3d5b3f" opacity="0.8"/>
<text x="234" y="64" font-size="9" fill="#3d5b3f" font-weight="600">↑</text>
<!-- cost_req: drifting up -->
<line x1="0" y1="85" x2="340" y2="85" stroke="#1c1a17" opacity="0.18"/>
<text x="-5" y="89" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="end">cost_req</text>
<rect x="200" y="79" width="28" height="12" fill="#3d5b3f" opacity="0.8"/>
<text x="230" y="89" font-size="9" fill="#3d5b3f" font-weight="600">↑</text>
<!-- kv_cache: flat -->
<line x1="0" y1="110" x2="340" y2="110" stroke="#1c1a17" opacity="0.18"/>
<text x="-5" y="114" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="end">kv_cache</text>
<rect x="198" y="104" width="4" height="12" fill="#1c1a17" opacity="0.35" opacity="0.6"/>
<text x="210" y="114" font-size="9" fill="#1c1a17" opacity="0.35">—</text>
<!-- downstream_err: flat -->
<line x1="0" y1="135" x2="340" y2="135" stroke="#1c1a17" opacity="0.18"/>
<text x="-5" y="139" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="end">downstream_err</text>
<rect x="198" y="129" width="4" height="12" fill="#1c1a17" opacity="0.35" opacity="0.6"/>
<text x="210" y="139" font-size="9" fill="#1c1a17" opacity="0.35">—</text>
<!-- mfu: flat -->
<line x1="0" y1="160" x2="340" y2="160" stroke="#1c1a17" opacity="0.18"/>
<text x="-5" y="164" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="end">mfu</text>
<rect x="196" y="154" width="6" height="12" fill="#1c1a17" opacity="0.35" opacity="0.6"/>
<text x="210" y="164" font-size="9" fill="#1c1a17" opacity="0.35">—</text>
<!-- hbm_spill: flat -->
<line x1="0" y1="185" x2="340" y2="185" stroke="#1c1a17" opacity="0.18"/>
<text x="-5" y="189" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="end">hbm_spill</text>
<rect x="198" y="179" width="4" height="12" fill="#1c1a17" opacity="0.35" opacity="0.6"/>
<text x="210" y="189" font-size="9" fill="#1c1a17" opacity="0.35">—</text>
<!-- collective_ops: flat -->
<line x1="0" y1="210" x2="340" y2="210" stroke="#1c1a17" opacity="0.18"/>
<text x="-5" y="214" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="end">collective_ops</text>
<rect x="196" y="204" width="6" height="12" fill="#1c1a17" opacity="0.35" opacity="0.6"/>
<text x="210" y="214" font-size="9" fill="#1c1a17" opacity="0.35">—</text>
<!-- Vertical baseline reference line -->
<line x1="200" y1="0" x2="200" y2="220" stroke="#1c1a17" opacity="0.6" stroke-width="0.5"/>
<text x="200" y="236" font-size="9" fill="#1c1a17" opacity="0.6" text-anchor="middle">baseline</text>
<!-- Rule annotation -->
<g transform="translate(-10, 255)">
<rect x="0" y="0" width="350" height="20" fill="#1c1a17" opacity="0.06" rx="3"/>
<text x="175" y="14" font-size="11" fill="#3d5b3f" text-anchor="middle" font-weight="600">4 of 9 drifting in same direction → slowbleed FIRES</text>
</g>
</g>
</svg>
<figcaption>Each signal's drift alone is harmless. A <em>pattern</em> of multiple signals drifting together is the architectural signature the rule recognizes.</figcaption>
</figure>

**Family C — Hotelling T² + Sequential MMD.** Multivariate distance from the baseline distribution. Each signal can stay within its own noise band, but if the joint vector sits off the correlation structure that healthy signals normally move along, Family C fires. This is the family that threshold-based systems categorically miss — the regression where each axis looks fine in isolation but the signals are moving wrong relative to each other. Uses Ledoit-Wolf shrinkage on the baseline covariance matrix for numerical stability; Wilson-Hilferty χ² quantile for the threshold. The Sequential MMD component runs as a betting e-process against random Fourier features.

<figure class="family-figure">
<svg viewBox="0 0 400 280" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Family C visual intuition">
<!-- Title -->
<text x="10" y="16" font-size="11" fill="#1c1a17" opacity="0.7" font-weight="600">two-signal joint view (actual engine uses 11-dim)</text>
<!-- Plot area -->
<g transform="translate(60, 40)">
<!-- Axes -->
<line x1="0" y1="0" x2="0" y2="220" stroke="#1c1a17" opacity="0.8" stroke-width="1"/>
<line x1="0" y1="220" x2="320" y2="220" stroke="#1c1a17" opacity="0.8" stroke-width="1"/>
<!-- Axis labels -->
<text x="-15" y="115" font-size="10" fill="#1c1a17" opacity="0.7" transform="rotate(-90, -15, 115)">signal 2</text>
<text x="160" y="245" font-size="10" fill="#1c1a17" opacity="0.7" text-anchor="middle">signal 1</text>
<!-- Univariate thresholds (dashed) -->
<line x1="250" y1="0" x2="250" y2="220" stroke="#8a3324" stroke-dasharray="3,3" stroke-width="0.6" opacity="0.5"/>
<line x1="0" y1="30" x2="320" y2="30" stroke="#8a3324" stroke-dasharray="3,3" stroke-width="0.6" opacity="0.5"/>
<text x="256" y="14" font-size="9" fill="#8a3324" opacity="0.8">thr 1</text>
<text x="260" y="28" font-size="9" fill="#8a3324" opacity="0.8">thr 2 ↓</text>
<!-- Mahalanobis ellipse (tilted correlation) -->
<ellipse cx="130" cy="130" rx="90" ry="35" transform="rotate(-30 130 130)" fill="none" stroke="#3d5b3f" stroke-width="1.5" stroke-dasharray="4,2"/>
<text x="15" y="198" font-size="10" fill="#3d5b3f" font-weight="600">Mahalanobis</text>
<text x="15" y="210" font-size="10" fill="#3d5b3f" font-weight="600">ellipse (T²)</text>
<!-- Baseline healthy points (inside ellipse) -->
<circle cx="100" cy="145" r="2.5" fill="#3d5b3f" opacity="0.6"/>
<circle cx="115" cy="140" r="2.5" fill="#3d5b3f" opacity="0.6"/>
<circle cx="130" cy="135" r="2.5" fill="#3d5b3f" opacity="0.6"/>
<circle cx="145" cy="125" r="2.5" fill="#3d5b3f" opacity="0.6"/>
<circle cx="160" cy="120" r="2.5" fill="#3d5b3f" opacity="0.6"/>
<circle cx="95" cy="152" r="2.5" fill="#3d5b3f" opacity="0.6"/>
<circle cx="120" cy="130" r="2.5" fill="#3d5b3f" opacity="0.6"/>
<circle cx="135" cy="145" r="2.5" fill="#3d5b3f" opacity="0.6"/>
<circle cx="108" cy="135" r="2.5" fill="#3d5b3f" opacity="0.6"/>
<circle cx="150" cy="135" r="2.5" fill="#3d5b3f" opacity="0.6"/>
<circle cx="125" cy="150" r="2.5" fill="#3d5b3f" opacity="0.6"/>
<!-- Drift point: inside both univariate thresholds but OUTSIDE ellipse -->
<circle cx="180" cy="80" r="5" fill="#8a3324" stroke="#f3efe6" stroke-width="1.5"/>
<text x="190" y="77" font-size="11" fill="#8a3324" font-weight="600">drifted</text>
<text x="190" y="89" font-size="11" fill="#8a3324" font-weight="600">point</text>
<!-- Annotations -->
<g transform="translate(0, 260)">
<text x="0" y="0" font-size="10" fill="#1c1a17" opacity="0.8">
<tspan fill="#1c1a17" opacity="0.7">• signal 1 alone:</tspan>
<tspan fill="#3d5b3f" font-weight="600"> within threshold</tspan>
<tspan fill="#1c1a17" opacity="0.7">  • signal 2 alone:</tspan>
<tspan fill="#3d5b3f" font-weight="600"> within threshold</tspan>
</text>
<text x="0" y="14" font-size="10" fill="#3d5b3f" font-weight="600">• joint position outside Mahalanobis ellipse → T² FIRES</text>
</g>
</g>
</svg>
<figcaption>Baseline healthy points form a correlated cloud (ellipse). New deploy's point looks "fine" on each axis individually but sits <em>off</em> the correlation structure.</figcaption>
</figure>

**Family D — Spectral / ACF oscillation.** Looks for periodic patterns in signals that shouldn't have them. When a signal starts oscillating — cache flapping, feedback loop, resource cycling — autocorrelation develops a peak at the oscillation lag. Family D sees the peak and fires. A `kv_cache` oscillating between 60% and 90% every 3 ticks has normal mean and normal variance; its time structure is what's wrong. Amplitude-based detectors miss this entirely.

<figure class="family-figure">
<svg viewBox="0 0 400 280" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Family D visual intuition">
<!-- Top panel: time series with oscillation -->
<text x="10" y="16" font-size="11" fill="#1c1a17" opacity="0.7" font-weight="600">signal value</text>
<line x1="10" y1="20" x2="390" y2="20" stroke="#1c1a17" opacity="0.18"/>
<!-- Sinusoidal oscillation (period 3) -->
<line x1="30" y1="70" x2="390" y2="70" stroke="#1c1a17" opacity="0.35" stroke-width="0.5"/>
<text x="32" y="66" font-size="9" fill="#1c1a17" opacity="0.55">baseline</text>
<polyline points="30,70 45,45 60,95 75,50 90,90 105,45 120,95 135,45 150,90 165,50 180,95 195,45 210,90 225,50 240,95 255,45 270,90 285,50 300,95 315,45 330,90 345,50 360,95 375,45 390,90"
fill="none" stroke="#3d5b3f" stroke-width="1.5"/>
<text x="200" y="115" font-size="9" fill="#1c1a17" opacity="0.55" text-anchor="middle">time (periodic oscillation at period ≈ 3)</text>
<!-- Bottom panel: ACF -->
<text x="10" y="145" font-size="11" fill="#1c1a17" opacity="0.7" font-weight="600">autocorrelation (ACF)</text>
<line x1="10" y1="149" x2="390" y2="149" stroke="#1c1a17" opacity="0.18"/>
<!-- ACF axis -->
<line x1="30" y1="160" x2="30" y2="260" stroke="#1c1a17" opacity="0.8"/>
<line x1="30" y1="240" x2="390" y2="240" stroke="#1c1a17" opacity="0.8"/>
<text x="20" y="165" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="end">+1</text>
<text x="20" y="245" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="end">0</text>
<!-- Significance bound (dashed red) -->
<line x1="30" y1="195" x2="390" y2="195" stroke="#8a3324" stroke-dasharray="3,3" stroke-width="0.6"/>
<text x="34" y="192" font-size="9" fill="#8a3324">significance bound</text>
<!-- ACF bars - peak at lag 3 -->
<rect x="40" y="160" width="20" height="80" fill="#3d5b3f"/>
<rect x="70" y="220" width="20" height="20" fill="#3d5b3f" opacity="0.5"/>
<rect x="100" y="215" width="20" height="25" fill="#3d5b3f" opacity="0.5"/>
<rect x="130" y="175" width="20" height="65" fill="#3d5b3f" stroke="#8a3324" stroke-width="1.5"/>
<rect x="160" y="222" width="20" height="18" fill="#3d5b3f" opacity="0.5"/>
<rect x="190" y="218" width="20" height="22" fill="#3d5b3f" opacity="0.5"/>
<rect x="220" y="183" width="20" height="57" fill="#3d5b3f" stroke="#8a3324" stroke-width="1.5"/>
<rect x="250" y="224" width="20" height="16" fill="#3d5b3f" opacity="0.5"/>
<rect x="280" y="220" width="20" height="20" fill="#3d5b3f" opacity="0.5"/>
<rect x="310" y="190" width="20" height="50" fill="#3d5b3f" stroke="#8a3324" stroke-width="1.5"/>
<!-- Lag labels -->
<text x="50" y="254" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="middle">0</text>
<text x="80" y="254" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="middle">1</text>
<text x="110" y="254" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="middle">2</text>
<text x="140" y="254" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="middle">3</text>
<text x="170" y="254" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="middle">4</text>
<text x="200" y="254" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="middle">5</text>
<text x="230" y="254" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="middle">6</text>
<text x="260" y="254" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="middle">7</text>
<text x="290" y="254" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="middle">8</text>
<text x="320" y="254" font-size="9" fill="#1c1a17" opacity="0.7" text-anchor="middle">9</text>
<text x="200" y="272" font-size="10" fill="#1c1a17" opacity="0.7" text-anchor="middle">lag (ticks)</text>
<!-- Annotation on lag 3 peak -->
<text x="145" y="172" font-size="10" fill="#8a3324" font-weight="600">peak at lag 3</text>
<text x="175" y="184" font-size="10" fill="#8a3324">→ periodic pattern</text>
</svg>
<figcaption>Signal oscillates (top). Autocorrelation at lag 3 is unusually high (bottom). Peak = "pattern repeats every 3 ticks" = oscillation detected.</figcaption>
</figure>

**Family E — Weighted-conformal Mahalanobis novelty.** A nonparametric unknown-unknowns detector. Instead of predicting what failure looks like, Family E compares the incoming deploy vector against a bank of ~16K known-healthy samples. If the new vector is more anomalous than α_E of the calibration samples, it's flagged — without having to know in advance what went wrong. The other four families assume a model of what degradation looks like; Family E doesn't.

<figure class="family-figure">
<svg viewBox="0 0 400 280" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Family E visual intuition">
<!-- Title -->
<text x="10" y="16" font-size="11" fill="#1c1a17" opacity="0.7" font-weight="600">distance to held-out healthy baseline samples</text>
<!-- Plot area -->
<g transform="translate(60, 40)">
<!-- Axes -->
<line x1="0" y1="0" x2="0" y2="200" stroke="#1c1a17" opacity="0.8"/>
<line x1="0" y1="200" x2="320" y2="200" stroke="#1c1a17" opacity="0.8"/>
<text x="-15" y="100" font-size="10" fill="#1c1a17" opacity="0.7" transform="rotate(-90, -15, 100)">signal 2</text>
<text x="160" y="225" font-size="10" fill="#1c1a17" opacity="0.7" text-anchor="middle">signal 1</text>
<!-- Healthy baseline cluster (N points) -->
<!-- Scatter of healthy samples -->
<circle cx="110" cy="120" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<circle cx="125" cy="115" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<circle cx="140" cy="110" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<circle cx="100" cy="135" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<circle cx="130" cy="120" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<circle cx="115" cy="125" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<circle cx="145" cy="115" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<circle cx="120" cy="105" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<circle cx="135" cy="125" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<circle cx="105" cy="110" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<circle cx="150" cy="120" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<circle cx="110" cy="130" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<circle cx="140" cy="130" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<circle cx="120" cy="135" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<circle cx="130" cy="130" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<circle cx="155" cy="125" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<circle cx="95" cy="125" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<circle cx="108" cy="118" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<circle cx="137" cy="117" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<circle cx="123" cy="125" r="2.5" fill="#3d5b3f" opacity="0.5"/>
<!-- Conformal ellipse (aggregate-fallback quantile) -->
<ellipse cx="126" cy="120" rx="50" ry="30" fill="none" stroke="#3d5b3f" stroke-width="1.5" stroke-dasharray="4,2"/>
<text x="185" y="108" font-size="10" fill="#3d5b3f" font-weight="600">conformal α=1e-4</text>
<text x="185" y="120" font-size="10" fill="#3d5b3f" font-weight="600">quantile boundary</text>
<!-- Novel point: far from all baseline samples -->
<circle cx="240" cy="55" r="5" fill="#8a3324" stroke="#f3efe6" stroke-width="1.5"/>
<text x="250" y="55" font-size="11" fill="#8a3324" font-weight="600">novel</text>
<text x="250" y="67" font-size="11" fill="#8a3324" font-weight="600">deploy</text>
<!-- Dashed distance line -->
<line x1="140" y1="110" x2="240" y2="55" stroke="#8a3324" stroke-dasharray="2,2" stroke-width="0.8" opacity="0.6"/>
<text x="180" y="80" font-size="9" fill="#8a3324" opacity="0.7" font-style="italic">Mahalanobis distance</text>
<!-- Label for baseline cluster -->
<text x="80" y="155" font-size="10" fill="#3d5b3f" font-weight="600">~16K held-out</text>
<text x="80" y="167" font-size="10" fill="#3d5b3f" font-weight="600">healthy samples</text>
<!-- Annotations -->
<g transform="translate(0, 240)">
<text x="0" y="0" font-size="10" fill="#1c1a17" opacity="0.8">
<tspan fill="#1c1a17" opacity="0.7">• no model of what "failure" looks like required</tspan>
</text>
<text x="0" y="14" font-size="10" fill="#3d5b3f" font-weight="600">• deploy's joint vector is too far from ANY healthy sample → FIRE</text>
</g>
</g>
</svg>
<figcaption>Compare incoming deploy (red) to a bank of known-healthy samples (pink). If it's farther from every healthy sample than a calibration threshold, it's novel — flag it.</figcaption>
</figure>

The design choice across the five is redundancy on purpose. Single-family detectors fail predictably on patterns outside their detection scope. A five-family portfolio with disjoint coverage catches more — and when multiple families fire on the same event, the agreement is itself diagnostic information for the verdict layer.

### The calibration compiler

`tools/calibrate.ts` takes a healthy-baseline trace and produces a `CompiledConfig` — per-(hour-of-day × day-of-week × tenant-tier) cells, each with its own statistical scaffolding: mean vectors, covariance matrices (Ledoit-Wolf for the parametric path; MCD / MRCD for the robust path), Cholesky factors for fast quadratic-form evaluation, AR(1) phi coefficients for Family A's drift model, mixture-supermartingale priors, betting-e-process baseline pools for Sequential MMD, conformal calibration scores for Family E.

The compile is deterministic. Same input baseline → same compiled config → same fire decisions on replay. This is the property that makes the system trustworthy as a production deploy gate: when an alert fires, the audit substrate emits a verdict that includes the compiled-config version. Anyone with the same compiled config and the same metric stream can re-derive the same decision exactly.

The tradeoff is heavy upfront, cheap at runtime. The compile takes minutes; runtime is arithmetic against precomputed structures. No matrix factorization at runtime, no threshold recalibration. Per-tick gate-evaluation latency on the full five-family portfolio:

| Scenario | Median | p99 | Max |
|---|---|---|---|
| Healthy path (full evaluation) | 29.8 μs | 62.8 μs | 0.194 ms |
| Regression path (C+E co-fire at t=11) | 27.8 μs | 60.8 μs | 0.167 ms |

For context, a typical LLM token-generation step is 10-100 ms. DeploySignal adds well under 1% overhead on a typical inference-request budget.

### The audit substrate

Every detector evaluation emits a structured `DetectorVerdict` record. Schema sketch:

```
DetectorVerdict {
  cell_key:           "hour-14_day-tue_tier-bronze",
  baseline_version:   "compiled-config-v4-fusion-novelty",
  schema_continuity:  true,
  family:             "C",
  alpha_consumed:     0.0024,
  alpha_remaining:    0.0476,
  fire:               true,
  fire_reason:        "hotelling_t2_exceeded_ville_bound",
  timestamp:          1714512000.123,
}
```

These records are replay-clean: same compiled config + same metric stream produces the same verdict. Post-incident reconstruction is exact, not approximate. When an SRE asks "why did the gate fire at 14:23?", the audit log answers with: which family fired, which cell, how much α was consumed, what the fire reason was, what compiled config was in use. The verdict can be re-derived from first principles.

This is the property that turns DeploySignal from a black-box decision system into something an operator can trust enough to ship behind. The audit record is the contract between the statistical engine and the human who has to defend the rollback decision in the postmortem.

## The methodology that emerged

### Four weeks, four roles, 250 coordination files

I didn't set out to invent a methodology. I set out to build DeploySignal. The methodology emerged because the project was statistically nontrivial — the math anchors are multi-paper; the calibration choices are load-bearing in subtle ways; the test coverage has to be rigorous enough that a wrong assumption about how a Cholesky factor is computed turns into a CRITICAL bug that only surfaces under perturbation. Single-pass agent generation produced code that compiled, passed surface-level tests, and would have shipped wrong.

So I structured the work as multi-role: one chat for Architect (the specs), one for Implementer (the code), one for Reviewer (audit), one for TPM (routing between the others), one for the PM role I held myself. Each role got per-chat project instructions pinning its identity. Coordination state lived in files in a `coordination/` directory rather than in chat history.

By around the second week, the directory had 100+ files and patterns started to repeat. The Architect was making the same kinds of attribution mistakes across different topics. The Implementer was committing similar omissions on different specs. The Reviewer was catching the same classes of bug across rounds. Each failure suggested a discipline that, if codified, would have prevented it.

I started writing those disciplines down. By the end of the project, there were about a dozen named disciplines being applied at specific moments in the workflow. The project's velocity improved measurably — not because the model had gotten better, but because the disciplines were preventing the failure modes that had cost time earlier.

The four weeks produced 250 coordination files, 970 tests, and a reference implementation that passes a reviewer-verified right-reasons audit on 6 canned demos including a reconstruction of a publicly-disclosed AI inference regression. It also produced, as a side effect, a methodology.

### What was specifically learned

Three concrete examples — each one a specific moment where a failure produced a discipline that I later named in Anchor.

**The Topic 52 phantom chain → memorial accretion.** During an investigation of detector behavior on one of the synthetic regressions, an architectural artifact misattributed firing-IDs between the TPR-sweep and FPR-sweep outputs. The Architect built a hypothesis tree on the wrong premise. The chain ran for 7 artifacts before the misattribution was caught via wrapper-bypass log diff. Wall-clock cost: 2-3 days.

The recognition was the moment that mattered. I had a clear memory of similar misattributions happening at least three times earlier in the project — each time resolved as a one-off, nobody had written it down, and so each subsequent occurrence cost the same time to diagnose from scratch. Memorial accretion became a discipline at that point because the same class of failure kept happening and the catalog wasn't growing to match. From then on, every failure that cost more than half a day got memorialized: what happened, what discipline would have prevented it, where the discipline applies, how its violations and confirmations would be tracked.

**The σ² compile-time underflow → the cold-eye Reviewer.** 60 of 66 statistical cells were affected by a precision bug in how σ² was computed for bounded-probability features like `tool_success_rate` and `refusal_rate`. The bug never surfaced as a test failure during initial implementation — every test the Implementer wrote against the suspect surface passed, because the test fixtures inherited the same precision assumption as the code they were testing.

The Reviewer caught it during an audit aimed at a different question. The recognition: the Implementer cannot audit its own surface effectively, because the tests it writes inherit the assumptions of the code it wrote. The Reviewer's value is not "second opinion" — it's "different starting context." A Reviewer reading the diff cold, with no exposure to the spec rationale or the Implementer's session history, will notice things the Implementer had no reason to notice. That's the cold-eye Reviewer discipline.

**The wrong-premise architect chain → pre-emit grilling.** A spec emitted with a factual claim that wasn't independently verified ("MCD on the clean fixture produces zero contamination flags"). The Implementer built against the spec; the claim was empirically wrong; the round produced wrong work that the Reviewer caught at audit. Wall-clock cost: 4 days.

The recognition: the most expensive class of error in multi-role work isn't the implementation bug — it's the spec-emit-time error that the Implementer absorbs as input. By the time the audit catches it, multiple downstream artifacts have been built on the wrong premise. Pre-emit grilling — adversarial self-review of an artifact *before* it forwards to the next role — became a discipline because spec-emit-time was where the costliest errors were being made unchecked.

None of these disciplines were known in advance. All of them were learned by failing in a specific way, naming the failure mode, and then naming the discipline that would have prevented it.

### The methodology turning around

By the time DeploySignal closed, there were about twenty named disciplines and a clear sense of which ones were load-bearing. I wrote them up as Anchor — disciplines, templates, worked example — and published it as a standalone repo. I hadn't set out to build a methodology pack; I'd just noticed patterns that seemed like they'd be useful to anyone running multi-role agent work, and writing them up as a pack was the cheapest way to share them.

Then I started Tessera using Anchor from round 1.

The shift was meaningful. DeploySignal's first 20 rounds had multiple expensive failures of types I now had explicit disciplines for — misattribution chains, self-confirming tests, spec-emit-time errors absorbed as input. Tessera's first 20 rounds had zero of those failure types. Not because the project was easier — Tessera vendors the DS engine for a different operational scope (per-shard observation of a running cluster), with its own complexity around topology adapters and bi-directional integration contracts — but because the methodology was applied from the start instead of distilled along the way.

Two concrete transfers:

- The audit-trail file discipline transferred directly. Tessera's `coordination/` directory mirrors DeploySignal's structure: Q-specs, reviewer reports, investigations, dispositions. The shape was already designed.
- Memorial accretion transferred *with the memorials*. Tessera started with about 15 inherited disciplines from DeploySignal that had been paid for in failure. The cross-project memorial file (`~/.claude/CROSS-PROJECT-MEMORIAL.md`) is real cross-project capital — disciplines learned on project N applying automatically to project N+1.

The three products together form a lifecycle:

> **DeploySignal catches before promotion. Tessera observes during steady state. Cairn attributes when something escapes both.**

DeploySignal is the pre-promotion gate. Tessera is the steady-state per-shard observer, vendoring the same statistical engine for cluster-scope problems. Cairn is the postmortem attribution layer for incidents that escape both — ranks candidate cause-events against incident onset, closes the loop. Each product applies the methodology to a different operational question; the methodology composes across the three.

## Close

DeploySignal is a reference implementation. It isn't packaged for production deployment as-is — the engine is shipped as runtime-exercised TypeScript modules with a deterministic test substrate, and integration with a specific deployment platform (Argo Rollouts, Flagger, custom Kubernetes operators) is wrap-the-engine work that comes downstream. What it is, instead, is a statistically rigorous answer to a narrowly defined question with provenance guarantees that let an operator replay a verdict and reconstruct the decision exactly.

It is also the project I built before I knew what I was building. The early weeks produced code I'm proud of. The later weeks produced code that's better than what I would have written without the disciplines that emerged early on. Building something hard reveals the structure of the work in a way that nothing else does. The methodology that emerged exists because the project was statistically demanding enough that single-pass generation didn't work — and now the methodology applies to other projects because the same constraints hold.

The next post will be about Tessera: what changes when the operational scope is "100-10,000 GPU shards in a running cluster" rather than "one deploy decision," how the same statistical engine recomposes around per-shard observation, and what 67 rounds of Anchor-methodology development looks like when the disciplines are applied from round 1 rather than distilled along the way.
