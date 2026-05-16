# Build log: codifying my engineering workflow into a Claude Code skill

> Internal draft — needs human review before publishing externally. Describes my working methodology for directing AI-augmented engineering work.

## Why workflow discipline matters more when agents are involved

Most process-discipline failures look the same. You improvise. The work drifts from the original requirement. Investigation gets folded into execution and bad data leaks into decisions. The rationale for a choice lives only in your head until someone asks two months later. Estimates have no methodology so they can't be defended. Lessons from one project don't carry to the next because there's no place to put them.

This is bad enough on a solo human effort. It's *worse* when you're directing agents. An agent will execute the plan you give it, faithfully and quickly. If the plan is fuzzy, the execution is fuzzy at speed. If the success criterion is "do X" instead of "X is true," the agent will produce something that looks like X-was-done and you'll find out months later it wasn't. AI-augmented engineering pays the cost of weak planning faster than human-only engineering does.

So I codified my own engineering workflow into a Claude Code skill. The discipline runs on rails. Process discipline you have to remember is process discipline you don't have.

## What the skill does

The skill takes a unit of work — anything from a one-shot bug fix to a multi-milestone overhaul — and runs it through a fixed sequence:

```
Analyze → Investigate → Plan → Implement → Test → PR
```

Each phase produces artifacts that the next phase consumes. Each phase gates the next: implementation doesn't start until the plan exists; the plan doesn't start until investigation has answered the open questions; the PR doesn't open until tests have actually run.

For larger work, the same flow nests:

```
Project
└── Milestone (versioned)
    └── Phase (numbered; ordered)
        └── Plan (concrete, executable)
            └── Session (one or more execution runs per plan)
```

The hierarchy is rigid; the content is whatever the work needs. At the project level: a project file, a state file, a living retrospective. Per milestone: a roadmap and requirements. Per phase: context, research, summary, and verification artifacts. Per plan: a concrete `PLAN.md` and a post-execution `SUMMARY.md`.

Two practices do most of the work:

1. **Decisions live in writing before code does.** Requirements and success criteria — written as TRUE-statements ("a complete project list anchored to `dotnet sln list` *exists*") — get captured up front. Execution doesn't drift because there's nothing to drift against — the plan IS the criterion. "Was X true at completion?" is mechanical. "Did we do X?" is interpretive.
2. **Investigation phases gate implementation phases.** For high-impact changes, the phase immediately before execution is dedicated to *collecting decision data, not writing code*. The implementation phase then references the investigation artifacts directly. Zero discovery gaps during execution.

The bookkeeping runs automatically. The human work is the thinking — what's the requirement, what's the TRUE-statement, is the investigation deep enough, did the retrospective surface a pattern worth carrying forward.

## How I applied it: a CI-test optimization case study

I used this workflow to scope a test-suite + CI overhaul for a .NET 9 monolith with ~6,100 tests across 45+ assemblies. Two milestones:

- **v1.0 (analysis pipeline, 4 phases, ~2.4 hours of execution).** Phase 1: static analysis — project inventory, fixture-dependency map, scan for sleep / sync-over-async patterns. Phase 2: dynamic timing collection via PowerShell-driven `dotnet test --logger trx`. Phase 3: bottleneck classification with explicit deduplication rules and **estimated CI minutes saved** per finding. Phase 4: ranked findings report. Output: a prioritized list projecting **~25.7 min (43.8%) CI time reduction** if all six findings shipped.
- **v1.1 (remediation, 3 phases, 1 day).** Phase 5: mechanical pattern fixes (no investigation needed) — `Task.Delay`/`Thread.Sleep` → event-based, sync-over-async → `await`, DI container slimming. Phase 6: investigation & audit — fixture lifecycle verified, 33 fixture-using assemblies classified shared-vs-isolation, 272 flaky tests triaged, 5 mock-candidate projects code-inspected per-test. Phase 7: high-impact implementation — fixture consolidation across 29 assemblies, root-cause fixes for 61 intermittents, mock conversions where the audit confirmed.

The investigation gate (Phase 6) paid for itself: it identified 4 isolation-required assemblies that would have caused regressions if Phase 7 had executed blindly, plus 80 non-mockable tests that would have been wasted conversion work.

The full case study — including the silent-greens discovery, the SQL-bound parallelism tuning, the cross-process circuit breaker, and the synthetic-`.trx` validation gate — is in [`pipeline-improvements.md`](./pipeline-improvements.md).

## Why it works for AI-augmented engineering

Three things crystallized for me across milestones:

1. **Static before dynamic maximizes information per unit of infrastructure cost.** The catalog phase finds half the issues without spending a single CI minute. By the time you pay for dynamic analysis, the structural problems are already named — and they don't pollute the timing baseline.
2. **Investigation phases turn high-impact changes into boring ones.** When you've already classified the assemblies and triaged the flaky tests, implementation is mechanical. The interesting work happened a phase earlier, on paper. This is exactly what agents are best at: executing well-specified work against decision data they didn't have to gather.
3. **Tooling-enforced discipline survives the moment of execution.** A phase invocation that doesn't produce the required context, research, plan, summary, and verification artifacts fails loud. That guardrail is more durable than my willingness to remember.

## Cost profile from the case study

| Milestone | Sessions | Phases | Plans | Wall-clock |
|---|---|---|---|---|
| v1.0 (analysis) | ~5 | 4 | 7 | ~2.4 hours execution |
| v1.1 (remediation) | ~6 | 3 | 12 | 1 day |

Model mix typically ~50–60% Opus, ~30–40% Sonnet, ~10% Haiku — Opus for planning/research/verification, Sonnet for execution, Haiku for mechanical bookkeeping.

## Takeaways

- **Methodology you have to remember is methodology you don't have.** Codified-into-tooling beats codified-on-paper.
- **TRUE-statement success criteria are the cheapest form of rigor available.** They cost nothing to write and turn verification into a yes/no check.
- **Investigation phases pay for themselves on every high-impact change.** The decision data you collect *before* implementation is what makes implementation boring — and "boring" is what agents execute best.
- **The hardest part of AI-augmented engineering is the thinking, not the typing.** A skill that enforces artifacts and phase gates moves the human work back to where it belongs.

---

*The case study where I applied this workflow is in [`pipeline-improvements.md`](./pipeline-improvements.md). Configs that drive workflows like this one live in the rest of [`ai-profile`](../).*
