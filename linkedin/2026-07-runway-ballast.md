<!--
Paste into LinkedIn's article editor by hand; headings→Heading 2, bold survives paste. Upload the diagram from linkedin/2026-07-runway-ballast-diagram.png where marked.
-->

Companies buying AI compute keep fighting the same two arguments: "when do we run out?" and "who's hoarding?"

**TL;DR:** I built two open-source tools to replace those arguments with evidence. Runway answers the first question, Ballast answers the second, and as of this month they talk to each other. Below: what each one does, what's validated on real data, and what's still simulation-only.

## The runway problem

"When do we run out?" sounds like a question with a date for an answer. It deserves a date with error bars. Runway is a vendor-neutral, open-source planning tool for GenAI compute capacity. It forecasts your capacity runway (the date demand crosses what you've deployed), shows what's slipping, and ranks which intervention extends that date cheapest: buy more, defer a launch, fix the workload growing fastest. The forecast is honest about uncertainty. Instead of one confident line, you get a band showing the range the data actually supports, so "we're fine until spring" arrives with its caveats attached.

A disclosure that applies to everything here: I built both tools, so this is a maker's account. Which is why the thing I'd point a skeptic at is the validation study. In July 2026 I ran Runway against public data, pre-registered: the pass/fail criteria were frozen before any data was fetched, the forecasting equivalent of calling your shot. Three of the four checks passed. The growth-model check failed, tripped by a 2026 break in token volumes (the raw unit of AI usage) that departed from the trend it assumed. The miss is published alongside the hits, and it's the reason to believe the hits. A forecaster that has never admitted a miss has never been tested anywhere it could have one.

## The hoarding problem

Ballast exists because a capacity escalation reached the CEO, who asked to see the requesting team's math. There was math, but nothing convincing enough to carry an escalation of that size. The usual fix for GPU fights is centralizing the pool under one owner. That buys control over who decides. It says nothing about whether anyone asked honestly.

Ballast is the missing record. Think of an insurer's claims history: it compares what each team reserved against what it actually used, cycle after cycle, and judges the gap against a reference class — teams with workloads like yours. The blending is empirical Bayes: your own track record, mixed with the group average so one weird month doesn't define you. The score measures calibration — do your predictions match what actually happens — rather than distance from some utilization cutoff. And a scope line I hold firmly: Ballast scores reservations; it never decides who gets capacity. Allocation stays wherever it lives in your org today.

The peers-like-you framing has teeth. Across 200 simulated runs per scenario, chronic padders were flagged every time, honest teams with spiky demand never, and — the part I like most — cautious teams reserving about 3.3 times what they forecast needing were also never flagged. A crude "use it or lose it" utilization audit would punish exactly those careful teams first. Ballast asks a different question: is your behavior explainable among teams like yours? Cautious and consistent passes. Padding doesn't.

Under the hood, each team's history feeds an anytime-valid e-process: a running tally of evidence you can check at any time without the peeking problem, so a slow, steady padder is eventually caught even when no single month looks damning. And the incentive result comes with its caveat in the same breath: in simulation only, with no real organization having run this loop yet, scoring teams this way produced a −71.6% reduction in padding (0.527 to 0.150) while utilization, the share of reserved capacity actually used, rose from 0.753 to 0.850 instead of sandbagging.

## A forecast is only as good as the asks feeding it

Runway plans against demand as stated. If the stated demand is padded, the plan inherits the padding: the run-out date is wrong, the ranked fixes are wrong, and money moves on the strength of asks nobody vetted. The new integration closes that hole. Ballast hands Runway each team's calibration score, and Runway re-runs the plan as if demand were honest, scaling each scored team's ask by its measured track record. You get both plans side by side, and the gap between them is the price of padding, in dates and dollars.

One design decision worth naming in a single sentence: the cross-team audit view lives on the executive dashboard, with the people who hold the forecast provider accountable, because the forecast provider is also the likeliest padder — while every team still sees its own score, since the incentive only works when the scored party can see the scoreboard.

![Who's hoarding? → Ballast; When do we run out? → Runway](2026-07-runway-ballast-diagram.png)

*Two questions, two tools, one plan: Runway forecasts the run-out date; Ballast scores whether the asks feeding it are honest.*

## Where the evidence stands

Plainly: the scoring core is validated on real data; the incentive claims are not. On a real Alibaba GPU cluster trace covering 439 organizations, Ballast's peer-blended score cut prediction error on held-out data (data the model never saw) by 53.2% versus using the group average, and its uncertainty ranges achieved coverage of 0.802 against a nominal 0.80 — its 80% confidence ranges came true 80.2% of the time. The same checks, code imported unmodified, then reproduced on a Microsoft Azure trace: different cloud, different resource type, different year. What remains simulation-only is everything about behavior change, including that −71.6% padding number. Those runs model organizations; they don't observe one. And Ballast's first version failed its real-data test outright. That failure is published, left standing, and is what forced the redesign that passed.

## Your turn

If you manage a shared compute pool: when someone asks for double what they used last quarter, what evidence do you ask for? And if you've settled the hoarding argument some other way, I'd genuinely like to hear how.

Full write-ups: johnpwarren.dev/blog/ballast/ · johnpwarren.dev/blog/runway/ · johnpwarren.dev/blog/runway-validation/ · github.com/johnpatrickwarren-oss/ballast · github.com/johnpatrickwarren-oss/runway
