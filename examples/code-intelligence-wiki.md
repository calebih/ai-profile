# Build log: an agent-first code wiki for 194 repos

> Internal draft — needs human review before publishing externally. No customer data, but it names internal platform repos and product areas; verify it's OK to share before posting.

## The problem: cloud agents wake up blind

A developer's local Claude Code session has every repo they care about already cloned. A *cloud* agent — fresh sandbox, no warm cache — doesn't. So the first thing it does on any non-trivial task is burn tokens trying to figure out which of our ~300 repos the work even lives in. Then it burns more tokens searching inside whichever repo it guessed.

That was already wasteful in late 2025. After the Anthropic price increase and tighter rate limits in early 2026, "agent rediscovers the architecture from scratch every run" stopped being a cost-of-doing-business item and turned into a real budget line. I wanted fresh cloud agents to spend their first few thousand tokens *routing* — picking the right repo and the right file — not relearning the platform.

## The pivot: throw away the platform, ship the wiki

My first design was overbuilt. The April 5 spec called for an ingestion pipeline against GitHub webhooks, MSSQL plus MongoDB plus Redis plus Azure Cognitive Search, a Roslyn analyzer, a TypeScript compiler analyzer, an ASP.NET Core API, a React browse UI, an MCP server in front of all of it. Months of work to stand up. Lots of moving parts to keep alive.

Then I read [Andrej Karpathy's LLM Wiki gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f), and the entire design collapsed into something better: **let the LLM do the heavy lifting, and store the output as plain `.md` files in a git repo.** No databases. No API server. No queue. No static analyzer. The LLM reads code, writes pages against a schema, follows relative file paths, and the whole thing is just text on disk.

The infrastructure design isn't *wrong* — it's the right answer if you don't trust the LLM. In 2026, I trust it enough that the simpler shape wins.

## The bet: optimize for agents, not humans

A few design choices fell out of "the agent is the primary reader":

- **Deterministic structure.** A `schema.md` pins down every page type — repo pages, domain pages, entity pages, dependency pages, feature maps. No artisanal one-off layouts. Agents can predict where information lives without a tour.
- **Relative file paths over wiki-links.** When a feature map says "request routing lives in `src/Routing/Assignment/...`," the agent follows the path directly. No guessing where a `[[wiki link]]` resolves.
- **Feature maps tied to real code locations.** Not architecture summaries — controllers, SQL tables, queue topics, frontend ownership for one feature area. The things you actually need to investigate a bug.
- **Explicit uncertainty.** Where the wiki isn't sure, it says so, instead of confidently lying.

I designed the schema, the page types, and the operating model. I didn't write 194 repo pages by hand. A dedicated maintenance agent did the analysis, drafting, and ongoing maintenance against the schema. Of the 113 commits in the wiki repo, 111 are the agent's. The two that aren't are mine, and they're both schema/operating-model changes.

## Three stages: index, ingest, lint

The wiki is a loop, not a build:

1. **Index.** `index.md` is the master catalog — every page, organized by category, with one-line summaries. It's how an agent (or a person) finds things without grepping the whole vault. Whenever a page is created or moved, the index entry has to track.
2. **Ingest.** Read code, write pages. On the initial pass that meant going repo by repo: structure, modules, public surfaces, data stores, feature maps for the bug-investigation hot zones. Ongoing, ingest is the diff-driven update — when a tracked repo changes, regenerate the affected sections of the affected pages, append to `log.md`, update `index.md` if the topology shifted.
3. **Lint.** Periodic audit. Broken `[[links]]`, orphaned pages, schema violations, contradictions across pages, code anchors that point at files no longer in the repo. The lint pass auto-fixes what it can and writes a `lint-report.md` for what needs a human eye.

A daily cron job runs the ingest+lint pair in small batches. Each run picks up changed repos and continues the deepening backlog — sharpening code anchors, strengthening data-store claims, retiring stale paths. Over 15 days the repo logged 96 deepening runs.

## The fresh-vs-stale insight

Here's the part that made this worth doing instead of just telling agents to clone repos:

**A cloud agent reading the wiki has fresher code intelligence than a developer reading their local clones.** Local checkouts rot the moment you stop pulling — yesterday's branch, yesterday's signatures, yesterday's tables. The wiki ingests every day. So an agent with no repos cloned, hitting the wiki cold, can be more current than a senior engineer who hasn't run `git pull` since Monday.

That inverts the usual assumption that "the developer has more context than the agent." For routing-and-recall — what file owns this feature, what tables back this domain, which queue carries that event — the agent ahead of the developer is the new default.

## What's actually in there

Built from 2026-04-11 onward:

- 194 tracked repo pages (10 deeply analyzed on day one; expanded to the full set on day two)
- 9 feature maps for bug-investigation hot zones — request workflows, third-party sync engines, search/indexing pipelines, entity change tracking, and similar high-traffic feature areas
- 9 domain pages, 4 entity pages, 3 dependency pages
- 96 logged incremental-deepening runs across 15 days

## What surprised me

Three course corrections I didn't see coming:

1. **The first wave was too high-level.** Initial repo pages read like architecture overviews. Useful for orientation, useless for "where does this bug live." That's what triggered the feature-map page type on day two. Lesson: agents don't want the executive summary. They want the file path.
2. **Paths were wrong more than I expected.** When I verified feature-map paths against the actual cloned repos, I had to systematically fix versioned controller paths, project-name mismatches, SQL file locations, and frontend module ownership. Anything sourced from intuition rather than `grep` was suspect. Now every code-location claim has to round-trip through the real repo before it lands.
3. **Placeholder data-store sections were a silent lie.** The 184 fast-pass repo pages from the scale-up day had templated "Data stores: SQL Server, Redis, …" sections that were defaults, not findings. I had the agent replace 181 of them with evidence-backed claims grounded in actual code. Net result: 138 pages with concrete data-store tables and zero placeholders left in canonical content. The lesson generalizes — at scale, "looks plausible" is the most expensive failure mode.

## What's next

Wiring fresh cloud agents to consult the wiki as their first routing step before they clone anything. The wiki is in place; the integration is what makes the rest pay off. The token-savings number is the receipt I want to ship next, and I'll write it up when I have it.

## Takeaways

- **Cloud agents need a routing layer.** Local-cache assumptions don't transfer.
- **Trust the LLM enough to delete the platform.** Karpathy's wiki pattern beat my four-database architecture because in 2026 the model can do the analysis the static tools were going to do.
- **Index, ingest, lint — pick a loop, not a build.** A wiki you don't maintain is a wiki that's wrong.
- **Daily ingestion beats local clones.** Stale checkouts are the new technical debt.
- **Be precise about the human/agent split.** Saying "I built it" when an agent wrote 111 of 113 commits trades short-term credibility for long-term credibility. Saying "I designed the operating model and an agent executed it" is more accurate and, in 2026, more interesting.

---

*Configs that drive workflows like this one live in the rest of [`ai-profile`](../). If you're hiring for AI-native engineering work, that repo is the receipts.*
