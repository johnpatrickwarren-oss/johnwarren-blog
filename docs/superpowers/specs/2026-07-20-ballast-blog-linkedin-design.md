# Design: Ballast blog post + Runway/Ballast LinkedIn article

Date: 2026-07-20
Status: approved pending user review

## Goal

Two written deliverables serving portfolio/credibility and thought leadership:

1. **Blog post** — the missing Ballast post for johnpwarren.dev, with the Runway
   integration as the payoff. Runway already has two posts (`runway.md`, May 30 2026;
   `runway-validation.md`, Jul 06 2026) and is recapped, not re-explained.
2. **LinkedIn article** — layperson-friendly, covering both Runway and Ballast (neither
   has LinkedIn coverage), linking out to the blog and repos.

Both follow `/Users/johnwarren/Writings/WRITING-STYLE.md`: no "not just X, but Y"
antithesis, no "load-bearing"/"in the limit"/"epistemic status", flat declaratives,
precision over absolutes, em-dash caps, disclosure-when-conflicted before claims,
distilled closing kicker (blog) / question-invitation close (LinkedIn).

## Deliverable 1 — Blog post "Show me the math"

- **File:** `src/content/blog/ballast.md` (johnwarren-blog, Astro content collection).
- **Front-matter:** `title`, thesis-bearing `description` (1–3 sentences),
  `pubDate: 'Jul 20 2026'`, `heroImage` following the existing `/og/og_<name>.png`
  convention (build via `.og-build/` if that pipeline supports adding one; otherwise omit).
- **Length:** 1,200–1,500 words. Em-dashes ≤ ~13.
- **Series convention:** opening line places the post in the
  DeploySignal → Tessera → Anvil → Cairn → Runway lineage, as every post in the series does.

### Structure (word budgets)

1. **Cold open — the dispute** (~150 w). Capacity-allocation dispute escalates to senior
   leadership; leadership asks to see the requesting team's math; the team can't produce
   it. Centralizing the pool delivers control but does not produce the math. Ballast is
   the legibility layer that does.
2. **Lineage/disclosure paragraph** (~120 w). Ballast is a sibling of
   DeploySignal/Tessera/Runway, built with Anchor methodology. It scores whether a team's
   reservation matches realized use; it never decides who gets capacity. COI named: author
   built it; half its claims are simulation-only (forward pointer to §4).
3. **How the score works** (~350 w). Two anchors: peer reference class (the bar is peers,
   not an absolute threshold) and the org's own requested-vs-realized track record,
   shrunk with empirical Bayes — the same post-audit EB design as Runway's slip priors,
   property tests transplanted between repos. Anytime-valid e-processes / e-BH for fleet
   FDR. Archetypes: padder caught (TP=1.0), honest teams not flagged (FP=0), including
   the high-stockout-cost honest team.
4. **What's real and what isn't** (~300 w). F1 predictive skill (53.2% held-out SSE
   reduction vs reference-class mean) and F2 interval coverage (0.802 vs nominal 0.80 at
   ±1.28σ) passed pre-registered on real LBNL traces (439 orgs, 5 GPU-type classes),
   replicated with verbatim thresholds on Azure Public Dataset V2 (2.7M VMs). F3
   junk-resistance and F4 incentive efficacy (padding 0.527→0.150, −71.6%, without
   sandbagging) are simulation-only. Asymmetry stated flatly in body text.
5. **Payoff — feeding the planner** (~250 w). Two-stage Ballast→Runway adapter
   (`runway_core.adapters.ballast`, JSONL wire formats, no live coupling): stage 1
   display-only justification packets labeled "synthetic-validated evidence; displayed as
   claimed, not established fact"; stage 2 (unlocked 2026-07-06 by the F1/F2 real-data
   pass) calibration scores (`r_hat` + predictive σ) driving Runway's
   `HonestDemandCalibration` what-if. Include the product decision: audit instrument on
   the Executive hub, not the Service VP's section; VP sees only their own score because
   the incentive loop requires the scored party to see it.
6. **Kicker close** (~80 w). Draft 2–3 candidates at writing time; direction:
   "Control tells you who gets the GPUs. Legibility tells you who earned them."

## Deliverable 2 — LinkedIn article

- **File:** `linkedin/2026-07-runway-ballast.md` in johnwarren-blog (paste-ready
  Markdown; user transfers structure into LinkedIn's article editor by hand).
- **Length:** 1,000–1,200 words.
- **Audience:** professionals fluent in budgets/forecasting/politics, zero GPU
  scheduling background. Every technical term glossed on first use.

### Structure

1. **Hook line 1** (no citation): "Every company buying AI compute is quietly fighting
   the same two arguments: 'when do we run out?' and 'who's hoarding?'" Then a 2–3 line
   TL;DR: two open-source tools, Runway answers the first, Ballast the second, they now
   talk to each other.
2. **The runway problem** (~250 w). Runway in plain terms: forecast with honest error
   bars; the pre-registered validation study that published its own miss, told as a trust
   feature.
3. **The hoarding problem** (~250 w). Origin anecdote; Ballast keeps score on
   reservations vs realized use, judged against peers (insurance-record analogy), never
   allocates. Padding result glossed with the simulation-only caveat inline.
4. **Why they talk to each other** (~200 w). Heading as payoff ("A forecast is only as
   good as the asks feeding it"). Padded reservations poison forecasts; Ballast's scores
   let Runway ask "what if demand were honest?"
5. **Honesty section** (~150 w). Real-data vs simulation split; publishing the miss as
   the differentiator.
6. **Close on invitation** (~80 w): "If you manage a shared compute pool: when someone
   asks for double what they used last quarter, what evidence do you ask for?" Then
   "Full write-ups:" links to both blog posts + both GitHub repos.

### Visual (approved)

One simple diagram, generated as SVG rendered to PNG for upload: two boxes
("When do we run out?" → **Runway**; "Who's hoarding?" → **Ballast**) with the connecting
arrow labeled "honest-demand check". Follow the dataviz skill when producing it. Save
alongside the article file (e.g. `linkedin/2026-07-runway-ballast-diagram.png`).

## Constraints and checks

- Facts come from the repos (`~/concord/ballast`, `~/concord/runway`) and their studies/
  STATE docs; every number quoted must be verified against source before publishing.
- Do not cite gate-value-study results (known-compromised).
- Pre-publish checklist from WRITING-STYLE.md: grep for "not just"/"not merely"/
  "load-bearing"/"in the limit"; count em-dashes; verify absolutes; COI before claims;
  closing line lands.
- Blog builds cleanly (`pnpm build` in johnwarren-blog) with the new post.

## Out of scope

- No feed post (article only, per user).
- No changes to Runway or Ballast repos.
- Publishing to LinkedIn itself (user pastes manually).
