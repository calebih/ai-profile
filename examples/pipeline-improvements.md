# Build log: a CI pipeline that silently passed with 80% of tests skipped

> Internal draft — needs human review before publishing externally. No customer data, but it describes internal CI architecture and team workflows; verify it's OK to share before posting.

## The lie

A .NET 9 monolith. ~6,100 unit tests across 45+ assemblies. The CI pipeline was green. The team had been merging against it for months.

Only ~1,204 of the ~6,100 tests were actually running.

A `dotnet test ... || true` in the Dockerfile was swallowing every failure so downstream stages could `COPY` the `.trx` artifact. Under load, testhost processes were crashing mid-run from SQL Server connection saturation; assemblies were silently disappearing from the `.trx`. The reporter saw a `.trx`, the gate said pass, the pipeline said green. Real bugs were merging unblocked.

This is the worst kind of CI failure. A red pipeline tells you what to fix. A green pipeline that's actually green tells you nothing's wrong. A green pipeline that *isn't* actually green is anti-information — it makes you trust a thing that's lying to you.

The thing this post is really about, though, isn't the lie. It's how I scoped and executed the fix.

## The methodology: GSD

I scope non-trivial engineering work using **[GSD (Get Shit Done)](https://github.com/gsd-build/get-shit-done)**, an open-source planning methodology codified into Claude Code slash commands and subagents. It decomposes a project into milestones, phases, and plans, with structured artifacts at every level: a project-level `PROJECT.md` and `MILESTONES.md`, per-milestone `ROADMAP.md` and `REQUIREMENTS.md`, per-phase `CONTEXT` / `RESEARCH` / `PLAN` / `SUMMARY` / `VERIFICATION` docs, and a living `RETROSPECTIVE.md` that compounds lessons across milestones.

Two practices do most of the work:

1. **Decisions live in writing before code does.** Requirements, success criteria as TRUE-statements ("a complete project list anchored to `dotnet sln list` *exists*"), and rationale all get captured up front. Execution doesn't drift because there's nothing to drift against — the plan IS the criterion.
2. **Investigation phases gate implementation phases.** For high-impact changes (data model, fixture lifecycle, parallelism), the phase immediately before execution is dedicated to *collecting decision data, not writing code*. The implementation phase then references the investigation artifacts directly. Zero discovery gaps during execution.

The methodology lives as `/gsd:plan-phase`, `/gsd:execute-plan`, `/gsd:verify`, plus a suite of subagents (`gsd-planner`, `gsd-executor`, `gsd-verifier`, `gsd-roadmapper`, `gsd-phase-researcher`, etc.). The *bookkeeping* runs on rails. The human work is the thinking — what's the requirement, what's the TRUE-statement, is the investigation deep enough.

For the test/CI problem, GSD scoped two milestones: v1.0 (analysis pipeline) and v1.1 (remediation execution). The pipeline stabilization PRs were the deployment step at the end.

## v1.0 — the analysis pipeline (4 phases, ~2.4 hours of execution)

Built in a single day. The static half required no infrastructure:

- **Phase 1 — Test catalog & static analysis.** Project inventory anchored to `dotnet sln list`; every project labeled unit vs. integration; fixture-dependency map across the test framework's fixture types (database, queue server, event server, DI container); per-project counts including 101 `[Fact(Skip)]` and 272 `[Trait("Category","Flaky")]`; static scan for `Task.Delay` / `Thread.Sleep` / sync-over-async with file:line citations.
- **Phase 2 — Dynamic timing collection.** PowerShell driving `dotnet test --logger trx`, parsing per-test and per-assembly durations, separating fixture-init from test-body time via xUnit `diagnosticMessages: true`.
- **Phase 3 — Bottleneck classification.** Merged static + timing into typed findings (DB fixture restore, DI container rebuild, sleep/delay, missing mocks, sync-over-async, WebApplicationFactory startup, flaky-retry inflation), each with affected test count, fixture-scope label, and **estimated CI minutes saved** — using documented methodology with explicit deduplication rules for co-occurring fixes.
- **Phase 4 — Report generation.** Self-contained markdown report with a ranked findings table sorted by estimated CI minutes saved.

**Output:** a prioritized list projecting **~25.7 min (43.8%) CI time reduction** if all six findings shipped.

## v1.1 — remediation (3 phases, investigation-gated)

Six findings, three phases:

- **Phase 5 — Mechanical pattern fixes.** No investigation needed. 21 `Task.Delay` / `Thread.Sleep` instances converted to event-based / `TaskCompletionSource` waiting. 42 `.Result` / `.Wait()` sync-over-async calls converted to proper `await`. DI containers slimmed for 3 isolated projects, scoped to actual dependencies rather than the full root container.
- **Phase 6 — Investigation & audit.** Decision data for the high-impact phase. Verified `IAssemblyFixture<T>` lifecycle behavior under the actual config. Classified all 33 DB-fixture-using assemblies as shared-fixture-compatible vs. isolation-required, with written rationale per isolation case. Triaged all 272 flaky-annotated tests into stale-label / genuinely-intermittent / CI-environment buckets. Code-inspected each of 5 mock-candidate projects per-test for real-DB-call vs. NSubstitute-mockable.
- **Phase 7 — High-impact implementation.** Created a shared-fixtures library with 8 group fixtures; applied consolidation across 29 of the 33 assemblies; isolation-required assemblies left alone; zero regressions. Removed 32 stale Flaky annotations. Root-cause-fixed 61 genuinely-intermittent tests. Added a conditional skip guard for 179 tests gated on a CI-only service. Converted confirmed mock candidates to NSubstitute pure unit tests with integration coverage verified elsewhere first.

The investigation gate paid for itself: Phase 6 identified 4 isolation-required assemblies that would have caused regressions if Phase 7 had executed blindly, plus 80 non-mockable tests that would have been wasted conversion work.

## Stabilizing the pipeline

The remediation work needed a CI pipeline that actually ran the suite. The same PR that carried v1.1 also carried the parallelism work that exposed the silent-greens problem in the first place. Three things had to ship together.

**1. Parallelism tuned to SQL connections, not CPU.** Tests share a single CI SQL Server, so the bind is connection limits, not cores. Empirical tuning:

| Configuration | Result |
|---|---|
| `MaxCpuCount=1` | Stable, slow |
| `MaxCpuCount=4`, no semaphore | Silent assembly crashes — tests vanished from `.trx` |
| `MaxCpuCount=4`, `maxParallelThreads=2` | **Stable**, ~6,100 tests in 25–30 min |
| `MaxCpuCount=4`, `maxParallelThreads=3` | Transport errors (12 connections > SQL ceiling) |

Four assemblies concurrent, each capped at 2 threads via per-project `xunit.runner.json` semaphores → never more than 8 active SQL connections. CPU was 90% idle the whole time. The DB was on fire.

**2. Cross-process circuit breaker.** A sentinel file at a fixed `/tmp/` path. Threshold = 1. Once any testhost trips it, every other testhost short-circuits — converting a 25-minute cascade into instant fail-fast. Cross-process state via a sentinel file is not elegant. xUnit testhosts run as separate OS processes; the sentinel is the simplest thing that crosses the boundary. Elegance loses to "this works and I can explain it in two sentences."

**3. Synthetic `.trx` injection.** The canonical "Build and Test with Docker" job lives in a *separate* shared-workflow repo we don't have write access to. Any new gate had to work inside the consuming repo *and* trick the existing reporter into failing correctly. The shape that fell out: a bash validation script with four hard-fail rules (any test failed, any testhost crash, pass rate < 100% of expected, absolute floor of 5,000 tests). When validation fails, the script generates a fake `.trx` with `<Counters failed="N">` and `outcome="Failed"` entries. The shared workflow's reporter checks `failed === 0 && errors === 0` — feed it a synthetic failure and it trips its own built-in gate. The whole job goes red without touching the shared workflow.

This is the move I'm proudest of. The "right" fix would have been to get write access to the shared workflow repo. Months of cross-team coordination. The synthetic `.trx` shipped in a single PR by inverting the problem: stop fighting the existing reporter, feed it the input that makes it work for you.

## Numbers

- **From ~1,204 tests silently passing → ~6,100 tests deterministically gated.**
- **Run time stabilized at 25–30 minutes**, matching the v1.0 projection of 25.7 min (43.8%) reduction.
- **33 DB-fixture-using assemblies → 8 shared group fixtures across 29 assemblies.** Zero regressions.
- **96 tests** reclassified `Dependent=true → false` after audit. **210+ tests** promoted from `[FlakyFact]` to stable after root-cause fixes. **182 wholesale-skipped tests** un-skipped; 4 isolated as genuine pre-existing bugs.
- **250+ Copilot review comments** triaged and resolved across four PRs.
- **Four PRs merged** over nine days, preceded by ~2 days of structured analysis + remediation work scoped under GSD.

## Takeaways

- **A green CI pipeline that isn't actually green is anti-information.** Treat fidelity as a higher priority than speed. A 30-minute deterministic run beats a 10-minute lying run by infinity.
- **The bottleneck is rarely what you think.** CPU was 90% idle; the DB was the constraint. Tune against the actual bind.
- **Investigation phases pay for themselves.** Phase 6 prevented regressions that would have shipped if Phase 7 had executed blindly. Document the decisions; the implementation gets boring.
- **Codify your methodology into your tooling.** GSD lives as Claude Code slash commands and subagents because process discipline you have to remember is process discipline you don't have. The agents enforce the artifacts. The human work is the thinking.
- **When you can't modify the shared system, feed it the input that makes it work for you.** The synthetic `.trx` works because we stopped fighting the existing reporter and started speaking its language.

---

*Configs that drive workflows like this one live in the rest of [`ai-profile`](../). The structured planning methodology used here — GSD — is described in detail in [`gsd-methodology.md`](./gsd-methodology.md). If you're hiring for AI-native engineering work that still respects production rigor, that repo is the receipts.*
