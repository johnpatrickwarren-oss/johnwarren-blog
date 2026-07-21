# Ballast Blog Post + Runway/Ballast LinkedIn Article Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publish the missing Ballast blog post ("Show me the math") on johnpwarren.dev and produce a paste-ready layperson LinkedIn article covering Runway + Ballast, with one diagram and an OG hero image.

**Architecture:** Two Markdown deliverables in the johnwarren-blog repo (an Astro content-collection post and a standalone LinkedIn draft), plus two images (an OG card PNG built from the existing `.og-build/` HTML-card pattern, and a simple two-box SVG→PNG diagram for LinkedIn). No changes to the runway or ballast repos.

**Tech Stack:** Astro content collections (Markdown + Zod front-matter), pnpm, headless Chrome for HTML→PNG rendering.

## Global Constraints

- Spec: `docs/superpowers/specs/2026-07-20-ballast-blog-linkedin-design.md` (approved).
- Style guide: `/Users/johnwarren/Writings/WRITING-STYLE.md`. Hard rules for BOTH pieces:
  - Zero instances of "not just", "not merely", "load-bearing", "in the limit", "epistemic status", "doing the work".
  - Flat declaratives over dialectical hedging; precision over absolutes.
  - Disclosure-when-conflicted: name the COI and where the work is unproven BEFORE the claim.
- Blog post: 1,200–1,500 words body text; em-dashes (—) ≤ 13; distilled closing kicker.
- LinkedIn: 1,000–1,200 words; hook in line 1 (no citation); 2–3 line TL;DR up top; every technical term glossed on first use; headings as payoffs; close on a question/invitation; ends with links.
- Every number in the Fact Sheet below was verified against `ballast/STATE.md`, `runway/CHANGELOG.md`, and `runway_core/adapters/ballast.py` on 2026-07-20. Do not introduce numbers from anywhere else. Do not cite gate-value-study.
- Commit after each task in `johnwarren-blog`; commit messages end with `Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>`.

## Fact Sheet (single source of truth for both drafts)

**Ballast (github.com/johnpatrickwarren-oss/ballast, Apache-2.0, TypeScript, v0.0.1-pre, 126 tests passing, sprag arch-gate 0 violations):**
- Origin: a capacity-allocation dispute escalated to senior leadership, who asked to see the requesting team's math; the team couldn't produce it. Centralizing the pool delivers control but does not produce the math. Ballast is the legibility layer that does.
- What it does: scores whether a team's GPU reservation is calibrated against realized use. It never decides who gets capacity.
- Score design: org-anchored longitudinal calibration (your own requested-vs-realized track record) shrunk toward the peer reference class with empirical Bayes (pseudo-count conjugate normal–normal). Same post-audit EB design as Runway's `slip_prediction.py`; known-answer property tests transplanted between the repos. Realization ratio clamped [0, 1.5].
- Fleet layer: anytime-valid e-processes; fleet-level false-discovery control via e-BH (Wang–Ramdas) consumed from `deploysignal-engine`. Demo fleet of 15 (5 padders + 8 honest + 2 risk-averse) at q=0.1 → watch list is exactly the 5 padders.
- Coverage matrix (N=200 seeds/scenario): `padder_steady` flag-rate 1.000; `honest_bursty` 0.000; `risk_averse_high_cu` 0.000. The crux: risk_averse reserves the MOST per unit forecast (≈3.3×) and utilizes the least, yet is not flagged, because the bar is class-relative, not an absolute utilization threshold.
- The published miss: v1's cross-sectional peer-relative flagging FAILED endpoint E1 on the real Alibaba `cluster-trace-gpu-v2020` trace (flag-rate by gap quartile [0, 0, 0.0071, 0.0127]; direction correct, magnitude far below the 0.25 clamp). The FAIL stands untouched in the study record and drove the v2 longitudinal redesign.
- v2 study (pre-registered, same Alibaba trace, 439 orgs across 5 gpu_type classes, run 2026-07-06):
  - F1 PASS — predictive skill (Stein): SSE_eb 3.895 < SSE_class 8.324 → 53.2% held-out SSE reduction vs the reference-class mean.
  - F2 PASS — interval calibration: coverage 0.802 (352/439) at ±1.28σ (nominal 0.80).
  - F3 PASS (SIMULATION-ONLY) — junk-resistance: honest beats junk, margin +60.70.
  - F4 PASS (SIMULATION-ONLY) — incentive efficacy: padding 0.527→0.150 (−71.6%) with utilization non-declining (0.753→0.850, no sandbagging ratchet).
- Azure replication (independent dataset, Microsoft Azure Public Dataset V2, 2019 VM trace, 2,695,548 VMs; 5,767 orgs across 4 core-count classes; thresholds verbatim, endpoint code imported unmodified): F1 PASS −93.4% (caveat, disclosed up front in the study: per-day usage approximated by each VM's lifetime avgcpu, which plausibly inflates F1's magnitude; verdict robust), F2 PASS coverage 0.787. Both endpoints reproduce on a different cloud, resource type, and year.
- H3 "show me the math" demo: padder presents a request of 279 units; the justification packet shows 45.3% utilization vs an 82% sanctioned bar and a calibration flag (e-process wealth 55.8 at cycle 17) — the math the real team couldn't produce.

**Runway integration (verified in runway CHANGELOG [Unreleased] + `runway_core/adapters/ballast.py`):**
- `BallastAdapter` consumes JSONL wire formats; no live coupling, no Python import of Ballast.
- Stage 1 (display-only): `JustificationPacket` → `ReservationIntegritySignal`, always labeled with `BALLAST_EVIDENCE_LABEL`, verbatim: "Ballast peer-calibration — synthetic-validated evidence; displayed as claimed, not established fact".
- Stage 2 (quantitative, unlocked 2026-07-06 by the F1/F2 real-data pass): schema `ballast.calibscore.v1` → `CalibrationScoreSignal` {org, ref_class_id, r_hat ∈ [0, 1.5], sigma_predictive, n_cycles, prior_weight}; drives the `HonestDemandCalibration` scenario delta, which scales each mapped customer's demand by r̂ ("what does the plan look like if demand were honest?"). Labeled with `BALLAST_CALIBRATION_LABEL`, verbatim: "Org-anchored calibration score — externally validated on real trace data (Ballast v2 study F1/F2); incentive results (F3/F4) are simulation-only".
- Product decision (owner, 2026-07-06): the cross-org audit instrument ("Reservation Integrity") lives on the Executive hub, because the Service VP is the forecast provider and the likely padder; the VP's own section keeps only a lightweight personal-score caption, because the F4 incentive loop requires the scored party to see their own score.

**Runway recap facts (for the LinkedIn piece only; the blog links to existing posts instead):**
- Runway (github.com/johnpatrickwarren-oss/runway, Apache-2.0, v0.16.0): vendor-neutral open-source planning tool for GenAI compute capacity — when does demand outrun deployed capacity, what's slipping, which intervention extends runway cheapest. Forecasts carry bootstrap confidence bands (parameter uncertainty, within-regime).
- The July 2026 pre-registered public-data validation study froze endpoints before fetching data and published its miss with its hits: three endpoints passed; the growth-model endpoint failed on a 2026 token-volume regime break. Existing posts: johnpwarren.dev/blog/runway/ and johnpwarren.dev/blog/runway-validation/.

**Links block (LinkedIn close):** johnpwarren.dev/blog/ballast/ · johnpwarren.dev/blog/runway/ · johnpwarren.dev/blog/runway-validation/ · github.com/johnpatrickwarren-oss/ballast · github.com/johnpatrickwarren-oss/runway

---

### Task 1: Blog post draft (`ballast.md`)

**Files:**
- Create: `src/content/blog/ballast.md`

**Interfaces:**
- Produces: the post at slug `/blog/ballast/`; Task 2 adds `heroImage` to its front-matter; Task 3's links block references the slug.

- [ ] **Step 1: Write the draft**

Front-matter (exact schema; description is thesis-bearing, 1–3 sentences — draft in this register):

```yaml
---
title: 'Show me the math'
description: 'Centralizing a GPU pool gives you control over who gets capacity, and no visibility into whether anyone asked for the right amount. Ballast is an accountability layer that scores reservations against realized use — validated on a real cluster trace, replicated on a second cloud, and honest about which of its claims are still simulation-only.'
pubDate: 'Jul 20 2026'
---
```

Body structure and budgets (from the approved spec; total 1,200–1,500 words):

1. **Cold open — the dispute** (~150 w). The escalation anecdote from the Fact Sheet. End the section on: centralization delivers control; it does not produce the math; Ballast is the layer that does.
2. **Lineage/disclosure paragraph** (~120 w). Series placement in sentence one (link the prior posts the way every post in the series does — see the opening of `runway-validation.md`): Ballast is a sibling of [DeploySignal](/blog/deploysignal/), [Tessera](/blog/tessera/), [Cairn](/blog/cairn/), and [Runway](/blog/runway/), built with the same Anchor methodology. State what it does and the anti-scope (never allocates). Name the COI: I built it; two of its four headline results are simulation-only, covered below.
3. **How the score works** (~350 w). Two anchors (peer reference class + own longitudinal track record), EB shrinkage in one plain paragraph, the shared-DNA point (same post-audit EB design as Runway's slip priors, property tests transplanted), e-processes + e-BH fleet layer in one paragraph, then the coverage-matrix crux: the risk-averse team that reserves ≈3.3× and utilizes least is NOT flagged — the bar is class-relative. Include the small table:

| archetype | reserves | flagged? |
|---|---|---|
| `padder_steady` | high, low use | yes (1.000) |
| `honest_bursty` | matched to demand | no (0.000) |
| `risk_averse_high_cu` | highest (≈3.3×), lowest use | no (0.000) |

4. **What's real and what isn't** (~300 w). Lead with the v1 E1 FAIL on the Alibaba trace (the published miss, left standing). Then v2: F1 53.2%, F2 0.802 on 439 orgs; then Azure replication (2.7M VMs, 5,767 orgs, F1 −93.4% with the lifetime-avgcpu caveat, F2 0.787). Then the flat asymmetry statement: F3/F4 (junk-resistance; padding −71.6% without sandbagging) are simulation-only. No footnoting the caveat.
5. **The payoff — feeding the planner** (~250 w). Two-stage adapter: stage 1 display-only with the verbatim `BALLAST_EVIDENCE_LABEL`; stage 2 unlocked 2026-07-06 by the real-data pass, `r_hat` scaling Runway's honest-demand what-if with the verbatim `BALLAST_CALIBRATION_LABEL`. Close the section with the Executive-hub product decision (audit instrument lives with the people holding the VP accountable; the VP still sees their own score because the incentive loop requires it).
6. **Kicker** (~80 w max). Draft 3 candidates in the working notes, pick one, delete the others. Seed direction: "Control tells you who gets the GPUs. Legibility tells you who earned them."

- [ ] **Step 2: Verify word count, em-dash count, anti-tell greps**

```bash
cd /Users/johnwarren/concord/johnwarren-blog
# body word count (strip front-matter): expect 1200–1500
awk 'BEGIN{n=0} /^---$/{n++; next} n>=2{print}' src/content/blog/ballast.md | wc -w
# em-dashes: expect ≤ 13
awk 'BEGIN{n=0} /^---$/{n++; next} n>=2{print}' src/content/blog/ballast.md | grep -o "—" | wc -l
# anti-tells: expect no output
grep -in "not just\|not merely\|load-bearing\|in the limit\|epistemic status" src/content/blog/ballast.md
```

Fix the draft until all three checks pass. (The verbatim `BALLAST_EVIDENCE_LABEL` quote contains an em-dash; it counts toward the 13.)

- [ ] **Step 3: Verify every number against the Fact Sheet**

Re-read the draft once, checking each figure (53.2, 0.802, 352/439, 439, 5, −93.4, 0.787, 2,695,548/2.7M, 5,767, −71.6, 0.527→0.150, 0.753→0.850, +60.70, 279, 45.3, 82, 55.8, ≈3.3×, 1.000/0.000, 2026-07-06, [0, 1.5]) appears only with its Fact Sheet meaning. Fix any drift.

- [ ] **Step 4: Build check**

```bash
cd /Users/johnwarren/concord/johnwarren-blog && pnpm build
```

Expected: build succeeds; `/blog/ballast/` present in output (`ls dist/blog/ballast/`).

- [ ] **Step 5: Commit**

```bash
git add src/content/blog/ballast.md
git commit -m "Add blog post: Show me the math (Ballast)

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 2: OG hero image (`og_ballast.png`)

**Files:**
- Create: `.og-build/og_ballast.html`
- Create: `public/og/og_ballast.png`
- Modify: `src/content/blog/ballast.md` (front-matter only: add `heroImage`)

**Interfaces:**
- Consumes: Task 1's post title/description.
- Produces: `heroImage: '/og/og_ballast.png'` in the post front-matter.

- [ ] **Step 1: Create the HTML card**

Copy `.og-build/og_tessera_rng.html` to `.og-build/og_ballast.html` and edit only the text content (kicker/headline/subtitle) to the Ballast post: kicker `BALLAST`, headline `Show me the math`, subtitle drawn from the description (e.g. "An accountability layer for GPU reservations — scored against realized use, validated on a real cluster trace."). Keep the 1200×630 card, palette, and layout exactly as the template has them (series-consistent).

- [ ] **Step 2: Render to PNG with headless Chrome**

```bash
cd /Users/johnwarren/concord/johnwarren-blog
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new \
  --screenshot="public/og/og_ballast.png" --window-size=1200,630 \
  --hide-scrollbars "file://$PWD/.og-build/og_ballast.html"
```

Expected: `public/og/og_ballast.png` exists, 1200×630 (`file public/og/og_ballast.png` reports PNG 1200 x 630). If Chrome is at a different path, locate it with `mdfind "kMDItemCFBundleIdentifier == com.google.Chrome"` and reuse the command. View the PNG with the Read tool to confirm the text is not clipped.

- [ ] **Step 3: Wire into front-matter and rebuild**

Add to `src/content/blog/ballast.md` front-matter: `heroImage: '/og/og_ballast.png'`. Then:

```bash
pnpm build && ls dist/og/og_ballast.png
```

Expected: build passes, image copied into dist.

- [ ] **Step 4: Commit**

```bash
git add .og-build/og_ballast.html public/og/og_ballast.png src/content/blog/ballast.md
git commit -m "Add OG hero image for Ballast post

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 3: LinkedIn article draft

**Files:**
- Create: `linkedin/2026-07-runway-ballast.md`

**Interfaces:**
- Consumes: Fact Sheet (Runway recap facts + Ballast facts) and the Links block.
- Produces: paste-ready Markdown; Task 4 adds the diagram reference.

- [ ] **Step 1: Write the draft**

Top of file: an HTML comment block for the user only (not to paste): `<!-- Paste into LinkedIn's article editor by hand; headings→Heading 2, bold survives paste. Upload the diagram from linkedin/2026-07-runway-ballast-diagram.png where marked. -->`

Structure (from the approved spec; 1,000–1,200 words):

1. Line 1 hook, no citation: "Every company buying AI compute is quietly fighting the same two arguments: 'when do we run out?' and 'who's hoarding?'" Then a 2–3 line TL;DR: I built two open-source tools; Runway answers the first, Ballast the second; they now talk to each other; here's what's validated and what isn't.
2. **The runway problem** (~250 w). Runway in plain terms (forecast with honest error bars; the cheapest-fix ranking); the pre-registered validation study that published its own miss, framed as the reason to trust the passes.
3. **The hoarding problem** (~250 w). Origin anecdote; Ballast as the insurance-record analogy (your reservations vs what you actually used, judged against peers with similar workloads); never allocates; the padder/honest coverage result in plain words; the −71.6% padding drop with the simulation-only caveat in the same sentence.
4. **"A forecast is only as good as the asks feeding it"** (~200 w, heading-as-payoff). Padded reservations poison capacity plans; Ballast's score lets Runway re-plan as if demand were honest; one sentence on the Executive-hub decision (the audit lives with the people holding the forecaster accountable — and the forecaster still sees their own score).
   Insert diagram here: `![When do we run out? → Runway; Who's hoarding? → Ballast](2026-07-runway-ballast-diagram.png)` plus a one-line alt-style caption.
5. **Where the evidence stands** (~150 w). Real-data vs simulation split stated plainly; the v1 failure published and left standing; replication on a second cloud.
6. Close on the invitation (~80 w): "If you manage a shared compute pool: when someone asks for double what they used last quarter, what evidence do you ask for?" Then "Full write-ups:" + the Links block from the Fact Sheet.

Glossing requirements (each term gets a plain-English handle on first use): empirical Bayes ("your track record, blended with the group average so one weird month doesn't define you"), anytime-valid e-process ("evidence you can check at any time without the peeking problem"), calibration ("do your predictions match what actually happens"), reference class ("teams with workloads like yours"), capacity runway ("the date demand crosses what you've deployed").

- [ ] **Step 2: Verify counts and anti-tells**

```bash
cd /Users/johnwarren/concord/johnwarren-blog
# word count excluding the HTML comment: expect 1000–1200
sed '/^<!--/,/-->$/d' linkedin/2026-07-runway-ballast.md | wc -w
grep -in "not just\|not merely\|load-bearing\|in the limit\|epistemic status" linkedin/2026-07-runway-ballast.md
```

Expected: count in range; grep silent. Also confirm by eye: hook is line 1 of the paste-able content; TL;DR is within the first 3 lines after it; every Fact Sheet number used matches; closing is the question + links.

- [ ] **Step 3: Commit**

```bash
git add linkedin/2026-07-runway-ballast.md
git commit -m "Add LinkedIn article draft: Runway + Ballast

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 4: LinkedIn diagram

**Files:**
- Create: `linkedin/2026-07-runway-ballast-diagram.svg` (source)
- Create: `linkedin/2026-07-runway-ballast-diagram.png` (upload artifact)

**Interfaces:**
- Consumes: Task 3's embed reference (filename must match exactly).

- [ ] **Step 1: Read the dataviz skill, then author the SVG**

Invoke the `dataviz` skill first (mandatory before chart/diagram work). Then hand-author a simple 1600×900 SVG: two rounded-rect boxes side by side — left box question "When do we run out?" over the answer **Runway** (sub-caption "capacity forecast with honest error bars"), right box "Who's hoarding?" over **Ballast** (sub-caption "reservation scorecard vs realized use") — connected by a single arrow from Ballast to Runway labeled "honest-demand check". Use the skill's palette guidance; large type (≥40px labels) so it survives feed compression; light background that also reads acceptably in dark mode per the skill.

- [ ] **Step 2: Render SVG → PNG**

```bash
cd /Users/johnwarren/concord/johnwarren-blog
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new \
  --screenshot="linkedin/2026-07-runway-ballast-diagram.png" --window-size=1600,900 \
  --hide-scrollbars "file://$PWD/linkedin/2026-07-runway-ballast-diagram.svg"
```

Expected: PNG exists at 1600×900. View it with the Read tool; verify no clipped text and the arrow label is legible at 50% zoom.

- [ ] **Step 3: Commit**

```bash
git add linkedin/2026-07-runway-ballast-diagram.svg linkedin/2026-07-runway-ballast-diagram.png
git commit -m "Add Runway/Ballast diagram for LinkedIn article

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"
```

---

### Task 5: Final pre-publish pass

**Files:**
- Modify (only if checks fail): `src/content/blog/ballast.md`, `linkedin/2026-07-runway-ballast.md`

- [ ] **Step 1: Run the WRITING-STYLE.md closing checklist against both pieces**

The seven-item checklist at the end of `/Users/johnwarren/Writings/WRITING-STYLE.md`, applied to each file: anti-tell grep (rerun the Task 1/Task 3 greps), em-dash budget, absolutes audit (each absolute claim true as stated or softened), COI-before-claim ordering, length bounds, closing line lands, flat-declarative test on any sentence that feels shaped.

- [ ] **Step 2: Full build + preview**

```bash
cd /Users/johnwarren/concord/johnwarren-blog && pnpm build
```

Expected: clean build. Then read `dist/blog/ballast/index.html` title/description meta to confirm front-matter rendered.

- [ ] **Step 3: Commit any fixes and report**

```bash
git add -A src/content/blog/ballast.md linkedin/
git commit -m "Pre-publish polish pass on Ballast post + LinkedIn article

Co-Authored-By: Claude Fable 5 <noreply@anthropic.com>"   # only if there are staged changes
```

Report to the user: word counts, em-dash counts, the chosen kicker, and what remains manual (deploying the blog, pasting the article + uploading the diagram to LinkedIn).
