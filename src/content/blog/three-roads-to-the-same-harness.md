---
title: 'Anchor: three roads to the same harness'
description: 'Anthropic, Superpowers, and Anchor arrived at the same patterns for multi-agent code generation — and where Anchor goes further.'
pubDate: 'May 25 2026'
---

Anthropic's [harness-design paper](https://www.anthropic.com/engineering/harness-design-long-running-apps) opens with an empirical finding I keep returning to. On identical prompts, a 20-minute solo agent run cost $9 and produced broken work; a 6-hour harness run cost $200 and produced something that functioned. Same model, same prompt. The thing that changed was the scaffolding around the model.

I read the paper after I'd already built [Anchor](https://github.com/johnpatrickwarren-oss/anchor) — a methodology pack I'd distilled from running DeploySignal, a statistically-rigorous deployment-safety system, as a 5-role multi-agent project. Reading it was strange. I had named different things — what Anthropic called Generator/Evaluator I had called Implementer/Reviewer; what they called sprint contracts I had called pre-route checklists; what they called context resets via structured handoffs I had called audit-trail file discipline — but the claims were identical.

I'd had the same recognition reading [Jesse Vincent's Superpowers](https://github.com/obra/superpowers) some weeks earlier. Superpowers enforces phase-level disciplines (brainstorm, design, execute, review) inside a single agent session. I'd encountered it after starting Anchor and decided to compose with it rather than reinvent the phase-level work it already did well. The patterns weren't identical to Anchor's, but the underlying claims were the same: structure the work, separate generation from judgment, treat the artifact as the source of truth.

Three teams arriving at the same patterns from three different starting positions — a research lab, an open-source IDE-tooling project, a solo operator running one production system — is the strongest possible evidence the patterns are real. This post is about what we converged on, and what Anchor adds beyond what's in either of the others.

## What three teams converged on

### Roles, not loops

Anthropic's central architectural claim is the **Generator-Evaluator pattern**, borrowed from GAN literature. "Separating the agent doing the work from the agent judging it proves to be a strong lever." Their three-agent architecture is Planner → Generator → Evaluator. Their reasoning: agents asked to evaluate their own work "tend to respond by confidently praising the work — even when... quality is obviously mediocre."

Anchor's four-role framework is Architect → Implementer → Reviewer → Memorial-Updater. The mapping is direct. Architect ≈ Planner. Implementer ≈ Generator. Reviewer ≈ Evaluator. The Memorial-Updater has no Anthropic counterpart — we'll return to it.

Superpowers operates within a single agent session and so doesn't enforce inter-role separation by process. But its phase discipline — brainstorm with three distinct approaches, then design, then execute, then review — produces a within-session approximation: each phase has different constraints, and the review phase is supposed to re-read the work as an adversarial reader.

All three reach the same destination: someone other than the generator must judge the work, and self-evaluation is structurally unreliable.

### The harness is structure, not intelligence

Anthropic: "every component encodes an assumption about what the model can't do." When new models release, you re-evaluate the harness and remove components that have become unnecessary.

Anchor's `skills/` directory is exactly this — each skill file is an encoded assumption about what the model (or the operator) won't reliably do unprompted. The catalog grows when a failure reveals a missing assumption.

Superpowers' skill-compliance enforcement — built on Cialdini's commitment-consistency principle — treats each skill as a discrete, named, opt-in discipline. The model agrees to the skill, then is held to it.

The shared claim: the model itself is a substrate. The scaffold around it is what determines whether the output is production-grade.

### Handoffs are artifacts, not chatter

Anthropic recommends **context resets** over compaction — when an agent session approaches the context window, summarize the work into a structured handoff and start a new session, rather than letting the agent summarize-in-place. They cite a phenomenon they call "context anxiety" — long-context models hurry to wrap work prematurely.

Anchor's audit-trail file discipline does the same thing at the role boundary instead of the context-window boundary. Every routing decision goes through `NEXT-ROLE.md`. Every architectural commitment goes into `coordination/specs/`. Every review outcome goes into `coordination/reviews/`. The coordination state is durable, on disk, version-controlled. The chat is ephemeral and re-derivable.

Superpowers' skill-execution model is more session-internal, but its emphasis on producing structured artifacts at each phase (`/brainstorm` produces a written set of approaches; `/execute-plan` produces a plan file) is the same pattern at smaller granularity.

The shared claim: if it isn't a durable artifact, it isn't a handoff. It's a wish.

### Plan before generate

Anthropic's Planner expands a vague user prompt into a detailed spec with ambitious scope. Anchor's Architect produces a Q-NN-SPEC document before the Implementer touches code. Superpowers' brainstorm-then-design phases force the same upstream commitment.

All three identify spec quality as the rate-limiter on downstream throughput. Generators that are unblocked by a bad spec produce code that compiles and ships and is wrong.

### The shared model

Flatten the three onto one diagram and you get something like this:

```
[Plan / Spec]  →  [Generate / Implement]  →  [Evaluate / Review]
        │               │                          │
        └─── durable artifacts at each boundary ───┘
```

Where the three sources differ is in *where* each box sits (a session phase, a process, an agent role) and in what scaffolding they wrap around it. Where they agree is that the boxes exist, that the boundaries are durable, and that the separation between them is load-bearing.

When three teams converge from three different starting positions on the same diagram, the diagram is the contribution. What follows is about what Anchor adds beyond it.

## Where Anchor goes further

Anchor adds four disciplines. Each addresses a failure mode the others don't structurally prevent, and each was developed in response to a specific class of failure on DeploySignal.

### Memorial accretion

Anthropic's harness is per-task. The model that runs your build today will read the same prompt tomorrow with no memory of how the harness performed yesterday. Superpowers is per-session — skills enforced within a session, no carryover of which-skill-failed-where across sessions or projects.

Anchor accumulates. Each violation of a discipline becomes a memorialized record. Each application becomes a confirmation. The records live in `MEMORIAL.md` at the project level and `~/.claude/CROSS-PROJECT-MEMORIAL.md` globally. Every new role-session reads them at startup. By round 30 of a project, the Implementer is reading 80+ accumulated reinforcements that were learned the hard way in rounds 1-29.

**A concrete example.** On DeploySignal Topic 52, an architectural artifact misattributed firing-IDs between two different validation sweeps (TPR vs FPR). The Architect built a hypothesis tree on the wrong premise. The chain ran for 7 artifacts before the misattribution was caught. Wall-clock cost: 2-3 days.

After the chain was resolved, the discipline was memorialized:

```
P3 axis 10: firing-attribution-discipline

"Before drafting any hypothesis tree on a SPECIFIC detector's
 misbehavior, verify firing-ID attribution at the source data
 (validation report card / FPR sweep output / TPR sweep output)
 BEFORE constructing the hypothesis tree. Attribution conflations
 between sweeps (FPR vs TPR; healthy-window vs injection;
 per-cell vs aggregate; per-signal vs cross-signal) propagate
 into hypothesis trees that chase phantom mechanisms."

Application moment: at hypothesis-tree drafting time, BEFORE
 drafting any V/Q variants.
```

The memorial paid for its 2-3 day origin cost within the first prevented recurrence. By the end of the project, it had been confirmed enough times that investigations on similar problems started with attribution checks as a reflex, not a reminder.

**The compounding claim.** Anthropic's harness gets you a single high-quality build. Anchor's memorials make the harness smarter every time you use it. Project N+1 starts with the disciplines project N learned, plus all the cross-project disciplines accumulated before that.

**But — won't this grow forever?** This is the obvious objection, and it's the one that decides whether the discipline is real. Anchor's answer has three parts.

First, an entry filter: a failure has to have cost roughly a half-day or more to merit memorialization. Most mistakes don't qualify. This prevents the catalog from filling with low-value reminders before it starts.

Second, ratios as a diagnostic. Each memorial tracks violations vs. confirmations over time. The DeploySignal firing-attribution memorial ran `1V/9C → 2V/11C → 3V/14C → 5V/20C` across the project's lifetime — confirmations growing faster than violations, which is the signal that the discipline is being internalized. The inverse pattern (violations growing faster than confirmations) is the signal that the rule isn't enforceable as written and needs to be sharpened or retired.

Third, explicit retirement. A memorial is retired when the discipline has stabilized (consistent confirmations, no new violations across multiple rounds), when the underlying failure mode has been structurally eliminated (e.g., a tool now prevents it), or when application has become universally automatic. Retired memorials stay in the filesystem as historical record, marked `Status: retired (date, reason)`. They're never deleted; the audit trail doesn't lie about its own past.

The named failure mode here is *memorial bankruptcy* — when you've accumulated 50 memorials and none are being actively applied, the catalog has become noise. The discipline includes the discipline of pruning itself.

Accretion without pruning is a memory bug, not a methodology. The pruning mechanism is what makes memorial accretion a discipline rather than a slowly-failing memory leak.

### Pre-emit grilling

Anthropic notes that evaluator agents trained to be skeptical "turn out to be far more tractable than making a generator critical of its own work." Their answer is to keep skepticism in the evaluator. Superpowers has a self-review phase inside the generator's session, structured by the skill catalog.

Anchor's pre-emit grilling sits between the two. Before a role's artifact is forwarded to the next role, the role grills the artifact against a discipline checklist — pre-route checklist for the Architect, acceptance-criteria coverage for the Implementer, severity-triage rubric for the Reviewer. Every concern surfaced by the grilling is classified into one of three buckets:

- **CRITICAL** — must be fixed before forwarding. The artifact has a load-bearing defect that would mislead the next role. Hold the artifact; revise; re-emit.
- **LIKELY-SURFACES** — pre-flag in the artifact as an anticipated question. Lets the downstream role hit it as expected rather than as a surprise.
- **PRE-EMPTABLE** — fold into the artifact directly. A scope drift the drafter noticed mid-write, a clarification that's cheaper to write now than to answer later, a small correction that can land in the same emit.

The grilling output is itself a structured artifact — `TPM-GRILL-<artifact-id>.md` or equivalent — that travels with the work. The next role reads it.

A worked example from DeploySignal Q58: TPM had drafted a routing artifact ready to forward to the Implementer. Applied pre-emit grilling. Result: 1 CRITICAL + 4 LIKELY-SURFACES + 4 PRE-EMPTABLE. The CRITICAL was a gap in the Architect spec around `parametricAr1Window` semantics that would have caused the Implementer to guess at intent, with downstream cost of 1-2 days of rework. Routing was held back; an Architect amendment was requested; rework was prevented.

This isn't a replacement for the cold-eye Reviewer. It's a cheap first pass that catches structural issues at the source, so the Reviewer doesn't burn time on problems the generator could have seen if it had been required to look. In DeploySignal, pre-emit grilling caught a meaningful fraction of issues that would otherwise have surfaced at the audit gate — at roughly a tenth the round cost.

The discipline Anchor adds is making this a named, structured step with its own artifact, rather than folding it into the generator's session as one of several phase obligations. The artifact is what makes it auditable later, and what makes "I already self-reviewed" verifiable rather than self-attested.

### Audit-trail file discipline

Anthropic's structured handoffs and Superpowers' phase artifacts move in this direction. Anchor goes further: every coordination event is a durable file with a canonical location.

The structure on DeploySignal looked like this:

```
coordination/
├── PRD.md                          # one file, the source of truth
├── ROUND-INDEX.md                  # what happened in each round
├── specs/Q-NN-SPEC.md              # one file per architectural commitment
├── reviews/R-NN-REVIEW.md          # one file per audit
├── investigations/INV-NN-TOPIC.md  # one file per investigation
└── dispositions/D-NN.md            # one file per CRITICAL disposition
```

By the end of DeploySignal, this tree held 250 coordination files: 95 architect replies, 90 TPM replies, 14 reviewer reports, 13 diagnostic memos, 7 postmortems, and 16 Q-spec documents (each a topic with its own multi-round investigation). None of them were minutes-of-meeting in the traditional sense. Each was a load-bearing artifact in the round it was produced — a spec the Implementer had to read, a review the Architect had to respond to, an investigation chain that determined what shipped.

What this enforces, in practice, is a discipline against treating spec or routing decisions as ambient context. Every Q-spec opens with an explicit `## Existing architectural surface` section — a citation table of every file referenced, with pinned SHA, line range, and a verbatim snippet of the cited code. Empty rows or paraphrased snippets fail the Reviewer audit automatically. The artifact has to demonstrate that the Architect actually opened the files, not just remembered them.

The advantage over session-state-as-source-of-truth: files survive context resets, model upgrades, operator handoffs, and your own bad memory at month three of a project. The cost is the discipline of producing the file in the first place. The trade is almost always worth it for projects that run more than a few rounds.

The memorial system rests on this. If coordination weren't file-backed, there'd be no archive to feed the memorial from.

### Role anchoring

Anthropic's harness assumes an orchestrator owns role identity — the script knows which agent is the Planner and which is the Evaluator, and the agents are bound to those identities by the script. This is a reasonable assumption for headless pipelines.

It is not the assumption that holds when a solo operator is running multiple chat windows. Real-world solo work involves three or four chat instances with different role identities, no orchestrator, and a tendency for the chats to confuse themselves about which role they are. "THIS session = Architect" claims appear in shared documents. The next chat reads the document, accepts the claim, and now two sessions believe they're the Architect.

Role drift is the most expensive failure mode of unguided multi-chat work. Anchor's role-anchoring discipline solves it with three rules:

1. A canonical role-mapping document at `coordination/PROJECT-ROLES.md`, the single source of truth for which session is which role.
2. Anti-drift rule: no role-identity claims in shared documents. The mapping is the only place identity is asserted.
3. Per-chat project instructions as the absolute source of identity — even if a shared document is later corrupted or misread, the per-chat instructions override.

This is the cheapest discipline in the pack — twenty minutes of setup at chat #2 — and the one that prevents the most expensive failure mode. It belongs in any multi-chat workflow even if you adopt nothing else from Anchor.

## What this means for quality, durability, and maintainability

### Quality

The convergent claim across all three sources is that two-pass with separation produces work that the single-pass generator cannot. The Anthropic paper backs this with the 20-minute/$9 vs. 6-hour/$200 comparison.

Anchor's contribution to quality is the **four-anchor pre-merge defense**: pre-emit grilling at the source, role separation between generation and judgment, cold-eye Reviewer with no generator context, and memorial reinforcement on every round-start. Each anchor catches a different class of failure. Single-pass generation produces work that compiles. The four anchors produce work that holds up.

Two concrete examples from DeploySignal, both flagged by the independent post-build audit as having high single-agent miss probability:

**σ² compile-time underflow on bounded-probability signals.** 60 of 66 statistical cells were affected by a precision bug in how σ² was computed for bounded-probability features like `tool_success_rate` and `refusal_rate`. The bug never surfaced as a test failure during initial implementation — every test the Implementer wrote against the suspect surface passed, because the test fixtures inherited the same precision assumption. The Reviewer's cold-context audit caught it during a per-signal cell-level review aimed at a different question. In production this would have caused Hotelling T² blow-ups on those signals during inference monitoring. The catch was not a Reviewer responding to something the Implementer flagged — it was the Reviewer noticing something the Implementer had no reason to notice.

**Diagonal-covariance parametric resampler bug.** The bootstrap resampler generated each signal independently, producing samples with a diagonal joint covariance. Downstream calibration was tuned against non-diagonal Σ, so the resampled distribution was mis-distributed in a way that didn't fail any test the Implementer had written. Caught via wrapper-bypass log diff — the discipline of reading the production output through a different code path than the test fixture used. Both this and the underflow bug would have shipped under a single-pass generation regime; both were caught by the structural separation between who built the thing and who audited it.

The empirical claim from DeploySignal is that an independent post-build audit estimated single-agent baselines would have missed 60-90% per CRITICAL finding that the four anchors caught. The number is unavoidably soft — counterfactuals are hard — but the direction is what matters. The disciplines weren't ceremony.

### Durability

Files outlive chats. A project structured with audit-trail discipline can be picked up six months later — by a different operator, a different model, or your own future self after losing context — and continued from where it left off. The PRD, the specs, the round index, the memorials. The reconstruction cost is low. This is the property that lets a project run for months without becoming unmaintainable.

### Maintainability

Cross-project memorial means the cost of starting project N+1 goes down with each project finished. Project 1 has zero accumulated disciplines. Project 5 inherits four projects' worth of `REINFORCED` lines, plus the pruning of those that became automatic. The harness is monotonically more capable over time — not because the model is better, but because the scaffolding has absorbed more failure modes.

## Where the Anthropic article is right and Anchor is under-developed

Three points where the Anthropic paper is sharper than Anchor as it currently stands.

**Strip complexity incrementally; re-evaluate when new models release.** Anchor's role count is fixed at four for any given round, and the tier dial (`solo` / `audit` / `full`) scales role count but not role shape. As underlying models improve, the case for separating Architect from Implementer-with-internal-design-phase will weaken. Anthropic's framing — every component encodes an assumption about what the model can't do; remove components when the model can do them — is a discipline Anchor needs an explicit version of. The pack adds disciplines and prunes individual memorials, but doesn't have a documented mechanism for retiring whole roles.

**Sprint contracts as a pre-negotiated explicit contract.** Anthropic's sprint-contract pattern — generator and evaluator agree on what "done" looks like *before* implementation begins — is sharper than Anchor's "Implementer reads the spec." The contract framing makes the success criteria first-class and removes the most common Reviewer-Implementer disagreement (whether the work is finished). Worth absorbing into the pre-route checklist.

**Live evaluator interaction.** Anthropic's evaluator interacts with the running system via Playwright MCP — it loads pages, clicks buttons, tests workflows. Anchor's Reviewer reads code and coordination artifacts. For backend work this is fine; for anything with a UI surface, it is a real gap. The Reviewer should be reading the running system, not just the diff.

These aren't fatal critiques. They're the next three things to absorb. Naming them in the open is the same discipline that produces the memorials — failure (or in this case, comparative weakness) gets explicit, named, file-backed treatment, not silence.

## Close

Three teams arriving at the same shape from three different starting positions — Anthropic's research, Jesse Vincent's open-source skill enforcement, and my own solo operator practice — is the strongest possible signal that the shape is real. The convergence is the contribution.

Anchor's contribution isn't to invent the shape. Anthropic's harness paper and Superpowers stand on their own. Anchor adds the four disciplines that turn a per-task harness into a per-program one: memorial accretion that compounds across rounds and projects, pre-emit grilling that catches structural issues at the source, audit-trail file discipline that makes coordination durable, and role anchoring that prevents multi-chat drift.

If you're running multi-agent code generation, the first thing to do is build the harness — read Anthropic's paper, install Superpowers, separate generation from judgment. The second thing is to make the harness learn. That's what Anchor is for.

The next post in this series is about [DeploySignal](/blog/deploysignal/) — the project these patterns came from, and what running a statistically-rigorous safety system as a five-role multi-agent build actually looked like from the inside.
