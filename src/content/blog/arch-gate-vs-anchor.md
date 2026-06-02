---
title: "You can't review your way out of a god object"
description: "Even a harness with adversarial review still builds god objects: code that passes every test and review yet is fragile, and breaks later in ways that are brutal to troubleshoot. arch-gate moves the structural floor off the model and onto a deterministic, zero-token ratchet."
pubDate: 'Jun 2 2026'
heroImage: '/og/og_arch_gate.png'
---

The post that started this was [k10s's "I'm going back to writing code by hand"](https://blog.k10s.dev/im-going-back-to-writing-code-by-hand/). The author *vibe-coded* a GPU-aware Kubernetes dashboard for seven months — prompting Claude feature by feature and shipping what came back, with little architecture decided up front. That came to 234 commits over ~30 weekends, the model doing the typing. The velocity felt incredible. The GPU fleet view generated perfectly on the first try. Then one feature interaction broke live updates across multiple views, the author opened `model.go`, found 1690 lines, and was, in their word, *horrified*.

The wreckage was specific and structural. The `Model` struct had become a god object — UI widgets, the K8s client, per-view state, navigation history, caching, all in one place. `Update()` was a 500-line function with 110 switch/case branches. Resources had been flattened to `[]string` with column identity carried by array index, so `ra[3]` meant "Alloc" until someone reordered a column and it silently meant something else. Switching views meant manually clearing nine-plus fields to stop data bleeding between contexts.

None of that happened in a bad commit. It happened across two hundred good ones. The author's own line is the whole thing: *"the velocity makes you think you're winning right up until everything collapses simultaneously."* The trend was visible the entire time. Nothing was watching it.

I built two different responses to that failure, and this post is about why I think one of them is the wrong tool for it.

**Disclosure, because it's load-bearing.** I wrote both of the things I'm about to compare. [Anchor](https://github.com/johnpatrickwarren-oss/anchor) is a multi-role agent harness — Architect → Implementer → Reviewer → Memorial — and I've argued [at length](/blog/three-roads-to-the-same-harness/) that it earns its keep. [arch-gate](https://github.com/johnpatrickwarren-oss/sprag) is the opposite kind of thing: a deterministic, zero-token gate. It grew out of Anchor and ships separately as `sprag`, but it is *not* Anchor. Where the harness is a pipeline of models, the gate has no model in it at all. "arch-gate is better" from arch-gate's author is worth nothing unless I'm honest about exactly where the harness still wins. So I'll draw that line carefully, and the conclusion isn't "don't run a harness." It's "stop paying a model to enforce facts."

## The obvious response is a harness, and it works

If a single agent rots a codebase over 234 commits, the natural fix is more agents: separate the one that writes from the one that judges. That's the Generator–Evaluator pattern Anthropic's [harness-design paper](https://www.anthropic.com/engineering/harness-design-long-running-apps) is built on, it's what Anchor's four roles are, and it works. Anthropic's own headline number is real: on an identical prompt, a 20-minute solo run cost $9 and produced broken work, while a 6-hour harnessed run cost $200 and produced something that functioned. I've watched Anchor's cold-eye Reviewer catch CRITICAL bugs the Implementer had no reason to notice. The harness is not snake oil.

But "it works" was established on the models of a year ago, and most of a harness is a bet about what the model *can't* do — bets that age as the model improves (a point I'll come back to). Three of them are aging badly right now, and all three bite hardest on exactly the k10s failure.

### It's a recurring token tax

A harness doesn't pay for architecture once. It pays every round. Each role re-reads context, re-threads the spec forward into the next role's prompt, and re-pays for all of it on every tool-use turn. When I [measured](/blog/dynamic-workflows-vs-anchor/) one Anchor round run as a Claude Dynamic Workflow against the same round in plain multichat, the workflow cost 2.26× — and even the lean arm spent real tokens per role, per round, for as long as the project lives. The harness is a *subscription* against rot. arch-gate's structural checks cost zero model tokens, forever, because there is no model in them.

### The quality lift is shrinking

The $9-vs-$200 gap was measured against a base model that needed the scaffold. As the base model gets stronger, the marginal value of a separate Architect role (versus a strong Implementer with an internal design phase) shrinks. I [admitted this against my own tool](/blog/three-roads-to-the-same-harness/) months ago: the case for Architect-as-its-own-session weakens with every release. You keep paying the full role tax while the thing it buys gets cheaper to get for free.

### It still ships god objects — and this is the one that matters

Here's the part the harness can't fix by being a better harness, or even a perfect one. Adversarial review and a green test suite answer the same question: *is this correct right now?* A god object answers yes. The code works. The tests pass. The cold-eye Reviewer reads the diff, finds no defect, and signs off — correctly, because on the axis it's measuring there is no defect. Structural fragility is a different axis, and it's invisible to the question both review and tests are asking, because nothing is actually *wrong yet*.

What's wrong is deferred. You can produce code that passes every test, survives every review, and ships looking great while being one feature interaction away from breaking. That is precisely how k10s broke: not a failed test, but a feature interaction that took out live updates across views. Fragile code doesn't announce itself. It works, and works, and works, until the day the structure can't absorb the next change — and by then the god object has tangled everything together, so the break is brutal to localize and brutal to fix. The bill for the fragility comes due long after the review that waved it through.

And the review is blind to the trend by construction. Each of k10s's 234 commits was a locally reasonable diff — one more field on `Model`, one more case in the switch, fine in isolation. A diff-scoped adversarial reviewer approves every one of them, because every one of them *is* fine on its own. The god object is the *sum* of 234 approved diffs. No review of a single change, however skeptical, sees an aggregate that only exists across the whole history. The thing that rots is exactly the thing a review of the current change cannot see.

So this isn't the model getting unlucky or failing to look. A harness's architectural rules are promptware: "keep `Model` thin," "no positional arrays," lines that live in a spec or a `CLAUDE.md`. Even a model that honors them perfectly on every diff still lets the accumulation through, because no single diff is the violation. The k10s author had Claude the entire time. A more expensive Claude, in a more disciplined pipeline, with a skeptical second Claude reviewing every change, still walks the same 234 reasonable commits into the same 1690-line file — slower, and with a bigger bill.

## A god object is a fact, not an opinion

Step back and look at what actually rotted in k10s. A struct grew past eight fields. A switch grew past some number of cases. Columns got addressed by magic integer index. One module got imported by too much of the codebase. Every one of those is a *count*. None of them is a judgment call.

Facts don't need an oracle. They need a measurement. And the moment a violation is a measurement rather than an opinion, you don't want a probabilistic, expensive, occasionally-distracted model in the loop at all — you want a deterministic check that returns the same answer every time, costs nothing, and can't be talked out of it.

That's the whole idea of arch-gate. You author the architectural invariants yourself (the k10s author's Tenet 1, *write the architecture yourself*), and the gate computes each metric over the real code and **blocks** on a breach. The checks run on a real AST (via ast-grep, for Go, TypeScript/JS, and Python; a no-dep heuristic engine is the fallback), so `Model`'s field count is the actual field count, not a regex's guess. The three invariants in the demo map straight onto the k10s tenets:

| Invariant | What it counts | k10s failure it would have caught |
|---|---|---|
| `model-not-god-object` | `Model` struct field count | the god object |
| `bounded-dispatch` | `switch m.view` case count | the 110-branch `Update()` |
| `no-positional-rows` | magic integer-index access (`ra[3]`) | positional fragility |

Exit `0` is pass, `3` is blocked-on-rot, `64` is a usage error. That's it. No tokens, no session, no transcript.

### The ratchet catches the trend, not the collapse

The single most important piece is the **ratchet**, and it's the direct answer to *"the velocity hid the trend."* The gate doesn't only block an absolute maximum. It blocks any regression against a recorded baseline: never-get-worse. Add one field to a six-field `Model` and the commit is blocked at 6→7 — not at 30, not at collapse. You don't need to guess the perfect threshold up front; you baseline from wherever you are today and the gate refuses to let it get worse.

This is also how it sees what per-diff review can't: the ratchet measures the *accumulated* state against a baseline, not the reasonableness of one change, so the trend that's invisible across 234 individually-fine diffs is a single blocked commit the instant it crosses the line. It's the early-warning system k10s never had. The companion `arch-trend` command walks git history and prints each invariant's metric at every commit, flagging where it first breaches — so a `Model` creeping 6 → 10 over twenty commits is a visible line on a chart at commit 20, not a horror discovered at commit 234. The pre-commit hook computes its baseline from `HEAD` on every commit, so the ratchet maintains itself with no manual upkeep, and rot literally cannot land.

### The escape hatch stays visible

A gate with no escape hatch gets disabled wholesale; a gate with *untracked* suppression rots silently. arch-gate's middle path is a per-occurrence comment — and the keyword is a tell about its lineage:

```go
n := legacyRow[2] // anchor:allow no-positional-rows: legacy CSV import, tracked in #123
```

The instance stops counting as a violation but is still *reported* in a Suppressions section. The escape hatch is auditable, never silent. (Yes — the gate's allow-keyword is `anchor`, because the gate grew out of Anchor. It's the part of Anchor I trust enough to run on every commit.)

## Where the harness still wins — and it's the half you should pay for

Here's the line I promised to draw carefully, because over-claiming here would be the same sin the optimistic harness reading commits.

A deterministic gate enforces the **structural** half of quality: size, coupling, dispatch growth, layering and dependency-direction, positional fragility, test presence. It is completely blind to the **behavioral** half. *Is the logic correct? Does this resampler actually preserve the covariance the calibration assumes? Do the tests catch real bugs or just inflate a coverage number?* A struct field count will never tell you that. Anchor's cold-eye Reviewer once caught a σ² underflow that hit 60 of 66 statistical cells and never surfaced as a test failure, because the fixtures had inherited the same wrong assumption the code did. That's exactly the kind of thing **only** a separate, skeptical reader can catch, and no gate replaces it.

So the division of labor isn't gate-versus-harness at all. Split it by what each side is actually good at:

- **Structural floor → deterministic gate.** Counts and shapes. Zero tokens, on every commit, can't drift, no oracle ceiling. This is the half k10s actually failed on.
- **Behavioral judgment → the model, used sparingly.** Correctness, hidden assumptions, test efficacy. Spend the expensive, probabilistic, genuinely-oracular tokens *here*, where the model is the only thing that can do the job.

The mistake isn't running a harness. The mistake is using model tokens to enforce things that are deterministic facts — paying an oracle, per round, forever, to count struct fields it might miscount anyway. k10s's failure was overwhelmingly structural, which means it's the cheapest possible failure to prevent: it never needed an oracle at all.

## Audit both against every new model

There's a maintenance discipline for anything you put between yourself and the model, and it's Anthropic's own: *every component encodes an assumption about what the model can't do; when a new model ships, re-evaluate and delete the components it no longer needs.* Run that audit on the two mechanisms and they come apart in a way that is itself the argument.

A harness is *made of* bets on model weakness. A separate Architect role bets the model won't design unprompted; pre-emit grilling bets it won't self-check; the cold-eye Reviewer bets it can't judge its own work. Good bets today — and some expire on every release. Honest maintenance means retiring roles as the base model absorbs them — and Anchor, for one, can prune individual memorials but has *no documented mechanism for retiring a whole role*. The audit on a harness is mostly subtraction, and what you subtract is what you were paying for.

Turn the same knife on the gate, because the discipline is worthless if it only cuts one way. The gate carries dead weight too. Some of its checks are also bets on the model: `max_function_lines` is a cheap proxy for complexity, and `require_tests` plus the shipped in-context disciplines bet the model won't reliably do TDD unasked. If a future model writes well-factored, well-tested code on its own, those earn retirement exactly like a role does — don't grandfather your own scaffolding just because it's cheap to run.

What survives the audit is the core: the human-authored structural invariant and the ratchet under it. The reason it survives is the whole point of building it this way. `model-not-god-object` was never a bet about what the model can't do; it's a statement about what your architecture should be. A stronger model writes the thirteenth `Model` field faster; it does not make the thirteenth field acceptable. The one component that outlives every model release is the one that was never an assumption about the model to begin with.

That asymmetry *is* the cost curve. A harness costs **O(tokens × rounds × model-price)**: you re-pay for every role on every round, and the audit keeps deleting pieces as the model absorbs them, so you are buying a shrinking thing at full price. The gate costs **~0 per commit** (one AST parse, no model), and its load-bearing part is exactly what the audit can't retire.

| | Harness | Deterministic gate |
|---|---|---|
| Cost shape | O(tokens × rounds × model-price) | ~0 per commit (no model) |
| As models improve | value shrinks; the audit deletes pieces | core survives — it was never a model-weakness bet |
| Guards against | behavioral failure (judgment) | structural failure (counts) |

The behavioral floor doesn't need an orchestration harness either: `require_tests` ratchets test presence, and `arch mutate` proves the tests you *have* actually kill bugs (flip an operator, re-run the suite, see whether a test fails) rather than pad a count. Both are zero-token and deterministic (the heavy one, mutation, runs out-of-band in CI), which leaves the model for the genuine-judgment calls that are the only thing it's the right tool for.

## Close

The k10s author went back to writing code by hand, and concluded the same thing I did from the other direction: *write the architecture yourself, before you prompt.* Where I'd push the conclusion further is on what happens *after* you've written it down. Their architecture lived in their head and in `CLAUDE.md` — which is to say, it lived somewhere a model has to *choose* to honor, every commit, forever. That's the gap two hundred good commits fell through.

So here's the architecture I'd actually run: the deterministic floor on the hot path, blocking rot on every commit for free; the harness (or just a strong model with in-context disciplines) reserved for the behavioral half, and run only when the work genuinely needs an oracle.

You don't have to choose between vibe-coding and hand-writing every line. You have to stop trusting a model — yours, or a more expensive pipeline of them — to be the only thing standing between you and a god object. Write the invariant down once, in a form that can't be reviewed-away or drifted-past, and let a gate that costs nothing refuse the thirteenth field on every commit.

Reserve the tokens for the problems that are actually opinions.
