# Examples

Build logs and PR walkthroughs from real AI-augmented work. The configs in this repo describe *how* I work; the examples here are *what* it looks like in practice.

## Available

- **[Code Intelligence Wiki](./code-intelligence-wiki.md)** — How I built an agent-first internal wiki for 194 repos, where 111 of 113 commits were agent-authored against a schema I designed. Covers the pivot from a 4-database platform to an `.md`-files-in-git approach (Karpathy's pattern), the three-stage operating model (index, ingest, lint), and the daily-freshness insight that makes cloud agents more current than developers' local checkouts.

- **[CI Pipeline Improvements](./pipeline-improvements.md)** — How I diagnosed and fixed a CI pipeline that was silently passing with only ~1,204 of ~6,100 tests actually running. Covers the discovery (a `|| true` swallowing every failure), the structured 2-milestone effort that surfaced it (analysis pipeline + remediation execution), the bind (SQL connection limits, not CPU), a cross-process circuit breaker for fail-fast, and a synthetic-`.trx` workaround that adds a real gate without modifying an unmodifiable shared workflow.

- **[GSD Methodology](./gsd-methodology.md)** — How I use [GSD (Get Shit Done)](https://github.com/gsd-build/get-shit-done), an open-source planning methodology codified into Claude Code slash commands and subagents, to direct AI-augmented engineering work. Walks through what GSD provides (project → milestone → phase → plan decomposition, TRUE-statement success criteria, investigation phases that gate implementation), how I applied it to the case study above, and why it works particularly well when agents are doing the execution.
