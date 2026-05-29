---
title: 'Dynamic Workflows: the hype, the receipts, and where they sit next to Anchor'
description: "What Claude Code's Dynamic Workflows actually are, why 'your session stays free' is not a cost claim, and what happened when I ran an Anchor cycle on them — measured both ways."
pubDate: 'May 31 2026'
heroImage: '/og/og_dynamic_workflows.png'
---

A line about Claude Code's new Dynamic Workflows kept showing up in my feed: *orchestrate agents in parallel while your session stays free.* I run a four-role agent methodology — [Anchor](https://github.com/johnpatrickwarren-oss/anchor), Architect → Implementer → Reviewer → Memorial — and "your session stays free" read like it might erase the coordination overhead I'd been paying for. So I built a small proof-of-concept and measured it.

On the same task, the dynamic workflow cost **about twice** what running Anchor the normal way cost. That surprised me enough that I spent most of the effort trying to prove the result wrong. It held. And when I finally read Anthropic's own documentation closely, the docs predicted it in plain language. The hype and the manual are not saying the same thing.

I've spent the last few posts on [convergence](/blog/three-roads-to-the-same-harness/) — where Anchor, Anthropic's harness-design paper, and Superpowers independently arrived at the same shape. This one is about divergence: a newer primitive that looks like it should subsume a multi-chat framework, and the measured reasons it doesn't, for the kind of work Anchor is built for.

**One disclosure, since it's load-bearing.** I wrote Anchor. An "Anchor wins" conclusion from Anchor's author is worth nothing unless the evidence is adversarial and the caveats are honest. So this leans on Anthropic's docs, hands you the raw numbers with their warts, and includes the part where I caught myself overstating. Judge it on that.

## What the primitive actually is

Per the [documentation](https://code.claude.com/docs/en/workflows.md): a dynamic workflow is *"a JavaScript script that orchestrates subagents at scale. Claude writes the script for the task you describe, and a runtime executes it in the background while your session stays responsive."*

Stated the way the docs state them, the genuinely good ideas are:

- **The plan moves into code.** With subagents or skills, the model is the orchestrator — it decides turn by turn what to spawn, and every intermediate result lands in its context. A workflow script "holds the loop, the branching, and the intermediate results itself, so Claude's context holds only the final answer."
- **Scale.** "Dozens to hundreds of agents per run," far past what one conversation can coordinate.
- **Background execution**, so the session isn't blocked.
- **Repeatability and resume** — the orchestration is a script you save, rerun, and resume mid-flight.
- **Quality patterns.** It can "have independent agents adversarially review each other's findings before they're reported."

For a 500-file migration, a codebase-wide bug sweep, or a research question that needs many sources cross-checked, this is the right tool. Hold that thought; I come back to it.

## "Free" is doing a lot of work in that sentence

Here is the whole misread. "Free" does not mean free of cost. The docs use *free* and *responsive* as synonyms:

> "a runtime executes it **in the background while your session stays responsive**."

> "agents work through a set of phases in the background **while your session stays free**…"

> "Workflows run in the background, so **the session stays responsive while agents work**."

"Stays free" means your session is free *to keep working* while the run executes elsewhere. It's a statement about not being blocked. There is one real resource that genuinely stays free — your main conversation's **context window**, because intermediate results live in script variables instead of piling into the chat. But context is not cost. The agents still spend the tokens.

On cost the docs are explicit, in a section the screenshots never include:

> "A workflow spawns many agents, so **a single run can use meaningfully more tokens than working through the same task in conversation.** Runs count toward your plan's usage and rate limits like any other session."

The marketing optimizes for *responsive and scalable*. The manual adds *and probably more expensive*. Both are true. The entire gap is in how one word gets read.

## The test

I ran one round of the Anchor cycle as a single dynamic workflow against a small, known-good benchmark: implement `compareSemver(a, b)` to the semver.org §11 precedence spec, with tests. Two arms, same task:

- **Arm A — the workflow:** one script, four `agent()` calls, Architect → Implementer → Reviewer → Memorial, run by the workflow runtime.
- **Arm B — Anchor's normal mode:** four independent role agents, coordinated the way Anchor coordinates today — separate sessions, artifacts passed between them.

Both arms ran the same disciplines: cold-context separation, pre-emit grilling, an adversarial cold-eye Reviewer. I reconstructed real per-role cost from the agent transcripts, because the live cost API only exposes output tokens — the full prompt/cache/cost breakdown has to be dug out of the logs, which is a finding in itself.

**First the part where I was wrong.** The first run came in around $9.26 for the workflow against $5.85 for multi-chat — roughly 1.6×. I started writing that up as a clean result. Then I ran an adversarial audit of my own measurement, the way Anchor's Reviewer audits an Implementer. It found a killer confound: the two arms had each generated their *own* spec, and the workflow re-threads spec text through later role prompts — so a larger Arm-A spec inflated Arm-A cost through the exact mechanism I was about to blame. The 1.6× was not a number I could defend. The audit's verdict was blunt: the data was enough to *retire the "5–10× cheaper" hope* — no saving appeared in any direction — but not enough to assert the inverse multiple. I had been one paragraph from quoting a number my own method couldn't support.

So I fixed it. One **byte-identical** spec, fed to both arms by the same path, symmetric prompts, only the orchestration substrate differing. Representative pricing — treat the ratio, not the dollars, as the signal:

| | Arm A — workflow | Arm B — multi-chat | A / B |
|---|---|---|---|
| **cost (USD)** | 8.81 | 3.89 | **2.26×** |
| completion tokens | 19,478 | 9,432 | 2.07× |
| cache-creation tokens | 233,727 | 107,819 | 2.17× |

Fixing the confound didn't rescue the workflow. It widened the gap. Two independent runs, both more expensive.

**The caveats, because full receipts means the unflattering ones too.** N = 1, on one small task — "held on `compareSemver`" is not "holds on a 40-file refactor." Arm B is a *proxy* for multi-chat (independent subagents, not four literal human-driven chats); it omits the human's back-and-forth turns, which would push multi-chat up, but enjoyed lean curated inputs, which push it down. And the 2.26× is directionally solid but numerically soft: a real bug in my harness — the workflow agents ignored the output paths I gave them and scattered files — made the Reviewer hunt around and inflated its cost, and I can't fully separate that artifact from genuine substrate overhead. So trust the direction and the refutation, not the exact multiple. Even so, no honest correction turns a 2× deficit into the 5–10× *surplus* the optimistic reading needed.

## Why no saving was ever on the table

The number isn't the interesting part. *Why* no saving was available is.

**A dynamic workflow's real efficiency win is deleting an LLM orchestrator's coordination tokens.** If your alternative is a manager agent that reads everything and reasons, in tokens, about what to dispatch next, then moving that plan into free-running JavaScript is a genuine saving. That's the case the primitive is built for.

But Anchor's orchestrator was never a model. It's a human routing between chats, or a file-driven state machine (`NEXT-ROLE.md`) in automated mode. Both cost zero model tokens. The workflow replaces a free orchestrator with a free orchestrator — and then adds overhead: each role's agent re-pays for context, threaded forward in prompts and re-read on every tool-use turn. You're paying to remove a cost you didn't have.

**And the headline word — *parallel* — never applied.** Workflows earn their keep on fan-out; the docs' own examples are "a codebase-wide bug sweep, a 500-file migration." Anchor's cycle is a dependency chain: the Implementer needs the Architect's spec, the Reviewer needs the implementation. You cannot parallelize a sequence. The primitive's signature strength was structurally unavailable to the use case before the first token.

One more, dispatched: even the genuine "your context stays free" benefit isn't an edge over Anchor. Multi-chat *already* isolates context — every role is its own session, and nothing accumulates the whole transcript.

## The comparison the screenshots skip

This is the part that actually matters. Anchor's automated mode already does the things workflows are praised for — and keeps two primitives the workflow structurally lacks. Verified in the framework's source, not asserted:

| | Dynamic workflow | Anchor (automated mode) |
|---|---|---|
| Unattended / deterministic / fan-out | yes | yes (multi-track waves) |
| Token-free orchestration | yes | yes (human / state machine) |
| Cold review + grilling + anti-scope | yes (confirmed in the POC) | yes |
| **Mid-run human escalation, then resume** | no | yes (`ESCALATE` pauses; resume) |
| **Dynamic role/tier selection** | no — hand-coded | yes (tier-router) |
| **Per-role model routing** | no — inherits session model | yes (map + dynamic selector) |
| Lean per-role context (cheaper) | no — re-pays context | yes (separate sessions) |

The two bold rows are the ones that bite. The docs are explicit: **"No mid-run user input — only agent permission prompts can pause a run. For sign-off between stages, run each stage as its own workflow."** For a methodology whose quality comes from a human resolving genuine ambiguity at the moment it surfaces, that isn't a small gap. It's the mechanism.

And the model-routing row is why my numbers were, if anything, *generous* to the workflow: every role in Arm A ran on the frontier model, because that's the session default, while Anchor routes the cheap roles — Implementer, Memorial — to cheaper models by design. The deck was stacked toward the workflow looking good on cost, and it still lost.

## Where workflows are the right tool

Being fair to a good primitive: reach for a workflow when the work is **wide, not deep**, and **independent, not sequential**.

- A codebase-wide audit or a large mechanical migration — real parallel fan-out.
- A research question that needs many sources gathered and cross-checked. The bundled `/deep-research` is exactly this, and it's genuinely good.
- A hard plan worth drafting from several independent angles and scored against each other.
- Anything where you want the orchestration codified and rerunnable, or your session to stay responsive while a long job runs behind you.

Those are real wins. None of them is "cheaper per token," and the documentation never claims it is.

## Close

Dynamic Workflows are a strong primitive for parallel, scalable, repeatable, background work — and the documentation sells them honestly on exactly those terms. The hype compresses *your session stays responsive* into *your session stays free*, and a reader hears *free of charge*. The manual says the opposite: more agents, more tokens.

For sequential, cost-sensitive, human-in-the-loop work — the kind Anchor exists for — the measured answer matched the docs. The workflow cost about twice as much, gave up the mid-run escalation gate, and couldn't use its one superpower on a dependency chain. The right move was to make Anchor's own overnight mode the default, not to port it onto a primitive built for a different shape of problem.

If there's one transferable lesson, it's the same discipline that produces Anchor's [memorials](/blog/three-roads-to-the-same-harness/): before you adopt something because it "removes overhead," check whether you were paying that overhead in the first place. I wasn't. The orchestration was already free — in the only sense of *free* that shows up on the invoice.

The methodology these patterns came from is [DeploySignal](/blog/deploysignal/); the convergence story that frames where Anchor fits among the harness designs is [here](/blog/three-roads-to-the-same-harness/).
