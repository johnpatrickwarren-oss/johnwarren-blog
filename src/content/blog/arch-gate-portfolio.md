---
title: "Length debt to zero, complexity under ratchet: five AI-built repos"
description: "Portfolio-scale validation of arch-gate across five AI-built repos: 35 god-files and 68 god-functions cleaned to zero with no behavior change across ~2,300 tests — then the gate's primary signal switched to a McCabe-complexity ratchet that freezes each repo's branchy-function count and blocks the next one. Plus the tooling fixes the work surfaced and the exceptions left alone."
pubDate: 'Jun 2 2026'
heroImage: '/og/og_arch_gate_portfolio.png'
---

Five repos, ~2,300 tests, zero behavior change. The one-time cleanup: **35 god-files and 68 god-functions to zero** in the gated product source. The durable result: **every gated repo's branchy-function count is now frozen under a McCabe-complexity ratchet** — deliberately not zero, just unable to grow. Two more repos newly gated so none of it creeps back.

That's the portfolio-scale version of [the argument I made in my last post](/blog/arch-gate-vs-anchor/): an AI-built codebase rots predictably, and a ratcheted structural gate cleans it up the way a per-round review never can. [The k10s post](https://blog.k10s.dev/im-going-back-to-writing-code-by-hand/) was one repo, painfully discovered at collapse. This is the same shape across five repos I own — measured, validated, and gated off going forward.

**Disclosure, as always.** I wrote arch-gate. A "portfolio remediated successfully" claim from arch-gate's author is worth nothing unless the receipts are auditable and the exceptions are honest. So: every "after" was confirmed with the full test suite green; PR numbers are listed where the work is public; the exceptions section names what arch-gate *didn't* remediate and why each was correct to leave alone.

## The remediation

| Repo | God-files | God-functions | Tests after | Delivered |
|---|---|---|---|---|
| deploysignal | 18 → 0 | 31 → 0 | 980 pass / 0 fail | PR #31 |
| deploysignal-engine | 7 → 0 | 12 → 0 | 93 pass / 0 fail | PR #14 |
| runway | 7 → 0 | 12 → 0 | 523 pass / 0 fail | local (no remote) |
| tessera (own code) | 2 → 0 | 10 → 0 | 533 pass / 0 fail | PR #9 |
| anchor (product) | 1 → 0 | 3 → 0 | 210 pass / 0 fail | pushed to main |
| **Total** | **35 → 0** | **68 → 0** | **—** | **—** |

Measure, for this remediation: god-file = source file >500 lines, god-function = function >80 lines — pure length, scoped to the source each gate watches (tests, generated bundles, and vendored copies excluded — see exceptions below). Every "after" row was validated by running the full test suite to green; the pure-build scripts (deploysignal, tessera) were additionally verified by **byte-identical** golden output, not just syntax. (A whole-repo scan still surfaces a handful of god-files — runway's five, for instance, are all in `tests/` — because the gate watches product source, not fixtures.)

Length is a crude proxy, though: a long-but-flat function is fine, a short and deeply-branched one is not. That was the argument in [the last post](/blog/arch-gate-vs-anchor/), so I took my own advice — the gate's *primary* signal is now McCabe complexity, not line count. The length numbers above are the one-time cleanup; the durable line is below.

The shape is consistent across repos. The biggest two — deploysignal and runway — had the most accumulated debt, which is what you'd expect: they're the oldest AI-built systems and ran longest before any structural enforcement existed. The newer ones had less, but not none. None of them had been written with arch-gate in mind; arch-gate didn't exist yet when most of the code shipped.

## What got locked in going forward

The remediation only matters if the debt can't reaccumulate. Two repos got newly adopted this session:

| Repo | Gate status |
|---|---|
| cairn | newly adopted (live pre-commit hook) |
| runway | newly adopted (live hook, watching its 4 source dirs) |
| anchor, deploysignal, tessera | already adopted (still passing) |

Every repo I own that has source code in it is now either under a live arch-gate hook or has been scanned. The pre-commit hook computes its baseline from HEAD on every commit; the ratchet refuses any regression. No new god-file can land, no new over-length function can land — and, as of this session, no new *complex* function can land either, which is the part that actually matters.

## Complexity is the line now, not length

Driving god-files and god-functions to zero was the visible win, but it's the cheap half. The durable change this session was switching what the gate *primarily* watches. **The primary signal is now McCabe cyclomatic complexity (>12); raw length stays only as a >150-line backstop.** It's ratcheted on all five gated repos. The honest current state, gated source only:

| Repo | Gate watches | Complexity it freezes (blocks above) |
|---|---|---|
| deploysignal | `engine`, `tools` | 62 (engine 27 + tools 35) |
| anchor | `.` | 18 |
| tessera | `.` | 9 |
| runway | source dirs | ~8 |
| cairn | `.` | 1 |

Those numbers are *deliberately not zero*, and that's the whole point of a complexity ratchet. It doesn't exist to force every branchy function to zero — some are legitimately branchy and cohesive (a flat dispatcher, a classifier, a parser), and those are a visible `// anchor:allow no-complex-functions: <reason>`, not a silent pass. It exists to **freeze the current count and refuse the next one**. Length-to-zero is a one-time scrub; complexity-as-ratchet is the line that holds, because complexity is what actually tracks how hard the code is to follow and test. (Whole-repo complexity runs higher than gated — deploysignal scans at 95 vs the 62 its `engine,tools` gate enforces — because the scan includes `tests/` and advisory dirs the gate doesn't watch. tessera and anchor watch `.`, so for them the scan number *is* the enforced baseline.)

## Already clean — the nuance that matters

The small and peripheral repos needed no remediation: spotless on length, at or near zero on complexity, so they're scanned but left ungated for now.

| Repo | God-files (>500) | Complex fns (>12) |
|---|---|---|
| clustersynth | 0 | 0 |
| anvil | 0 | 1 |
| anchor-cc-poc | 0 | 4 |

This matters for the honest framing. Not every AI-built repo carries the same debt magnitude. Smaller projects, more recent ones, or ones written under stricter structural intent can already be clean. The headline "35 → 0" is portfolio-wide; it's *not* "every AI-built codebase has roughly seven god-files." The truer statement is that most AI-built repos accumulate some, and the ones that accumulate a lot accumulate them the way k10s described — slowly, across many individually-reasonable commits, invisible to per-diff review.

## What arch-gate deliberately left alone

The exceptions are where the rigor lives. Without naming them, "35 → 0" reads as cherry-picked. With them named, the claim earns its weight:

| Not remediated | Reason |
|---|---|
| `tessera/calibrators/family-c.ts` | vendored from deploysignal (sync-at-pin) — fix upstream, don't fork |
| crucible | yours but vendors the deploysignal portfolio — fix at source |
| confseq, NAB | third-party clones (gostevehoward, numenta) — not yours to refactor |
| each repo's `tests/`, `data/`, generated bundles | not architectural source — fixtures and outputs |

The vendored-from-elsewhere cases are the most interesting. arch-gate's scoping was deliberately strict: if the source-of-truth for a file lives in another repo, fix it there, not in the copy. Otherwise you fork the vendored code and inherit a maintenance burden that no remediation should ever create. That's a discipline the gate enforces by *not* enforcing — it scopes itself to the architectural source under your control, not to every file that happens to be checked in.

The third-party clones (confseq, NAB) are out for a related reason: they're someone else's code, present for reference and benchmarking. Reshaping them would diverge from the upstream and break the whole point of having them in the tree.

## The tooling bugs the work surfaced

Running at portfolio breadth surfaced three fixes to arch-gate itself, all shipped mid-session. Naming them publicly is the trust move — most "tool validation" posts hide what they found wrong with their own tool:

1. **Walkers now skip Python `.venv/`, `site-packages/`, `__pycache__/`.** Without this, a root scan of runway falsely reported **1,848 god-files**, almost all of them dependencies. The pre-fix output was obviously wrong; the fix made the scan honest.
2. **Scan now counts Go and Python god-functions.** Previously god-function counting was JS/TS-only, silently reporting 0 on Go and Python source. This meant tessera and the Python parts of runway had been getting under-reported. The fix surfaced the additional debt, which then got remediated alongside the rest.
3. **The scan now respects `.gitignore` and counts tracked source only.** Before, generated and ignored dirs (`dist/`, `coordination/`, build output) leaked into the totals. Counting only tracked source is what makes the per-repo numbers here trustworthy rather than inflated by artifacts.

All three came out of the portfolio run being the first time arch-gate ran at this breadth. Tools get more accurate when they're applied to harder problems than the ones they were first tested on. That happened here three times; every fix shipped before the remediation completed.

## How the remediation actually worked

The reusable pattern, consistent across all five repos:

- **Subagents for code movement.** Splitting a 1,421-line `charts.py` into a facade + 8 themed sub-modules is the kind of mechanical, scoped, easy-to-verify work a subagent does well. The orchestrator (me) decided the splits; the subagent did the file moves and the import-rewriting.
- **The pre-commit hook as the ratchet.** Every commit ran through arch-gate's live hook. A commit that made things worse was blocked before it landed. Baseline from HEAD per-commit, no manual upkeep.
- **The test suite as ground truth.** Behavior preservation is a question only the suite can answer. Between batches, I ran the authoritative suite; commits only landed when it stayed green. For tessera and the pure-build pieces of deploysignal, I additionally diffed generated artifacts byte-by-byte, because syntax-green isn't strong enough when the code's job is to produce downstream output.
- **Facade re-exports for API preservation.** Every public symbol kept its import path. `from runway.optimize import compute_runway` still resolves; the implementation just lives in a sub-module now. That's what makes a refactor "no behavior change" rather than "no test failures, but every caller has to update its imports."
- **One honest exception:** worktree isolation didn't work for runway. Editable-install + shared venv means a worktree would test the main checkout, not its own edits. The workaround was in-tree work with the authoritative suite between batches; commits held until each batch passed.

That pattern is the discipline. It composes well, scales across repos with similar shapes, and produces auditable evidence at every step.

## What this means for the tools above the gate

A note worth making lightly here; the full unpacking belongs in a follow-up post.

arch-gate didn't *compose* with my other tools. It paid down the structural debt those tools produced. [Anchor](https://github.com/johnpatrickwarren-oss/anchor)'s `CLAUDE-COMMON.md` prose was trying to encode the disciplines that arch-gate now enforces deterministically. The intent was right; the enforcement mechanism wasn't strong enough — prose disciplines aren't enforced by code, and a per-round review can't see what only the *sum* of two hundred individually-fine commits ever becomes.

What this means in practice: for any new repo, arch-gate is now the first tool to reach for. For existing repos, the adopt loop pays down whatever debt the prior tooling didn't catch. The harness — Anchor, [dynamic workflows](/blog/dynamic-workflows-vs-anchor/), whatever — becomes appropriate for the specific behavioral wedge it's actually good at: correctness, hidden assumptions, test efficacy. The structural floor goes to the gate. Always.

The map of all three tools, with measurements from each, is its own post — coming next.

## Close

Five real repos. Thirty-five god-files and sixty-eight god-functions reduced to zero, ~2,300 tests preserved, zero behavior change. Two more repos newly gated. And the line that holds going forward isn't length at all — it's complexity under a ratchet, so the next branchy function is the one that gets blocked.

The transferable principle is the same one as the [original gate post](/blog/arch-gate-vs-anchor/): before you adopt a tool to enforce something, check whether it's a fact or an opinion. Facts go to a deterministic gate. Opinions go to the model. The cost-and-correctness curves point opposite directions, and they don't cross.

Reserve the tokens for the problems that are actually opinions.
