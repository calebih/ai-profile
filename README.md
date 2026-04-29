# Caleb Hur — AI-First Engineering Configs

My working configuration for AI-augmented software engineering — the actual files I keep in production projects, with the reasoning behind each decision.

I direct Claude Code, Codex CLI, GitHub Copilot, and Cursor as a core part of how I architect and ship. The artifacts here are what makes that workflow repeatable across teams and codebases.

> If you're a hiring manager: this repo is the receipts. The headline is "AI-First engineer who orchestrates agents at scale" — these are the configs that back the claim. None of this is theoretical; every file here is in use on a real project. The build logs in [`examples/`](./examples/) show the same workflow applied at scale.

## What's in here

| File / folder | What it is | When you'd use it |
|---|---|---|
| [`CLAUDE.md`](./CLAUDE.md) | Project-memory file Claude Code reads on every session start. Written as an annotated example with commentary. | Drop into the root of any repo where you use Claude Code. |
| [`AGENTS.md`](./AGENTS.md) | One-line pointer at `CLAUDE.md`. The cross-tool standard (Codex CLI, OpenAI Codex, etc.) reads this. | Single source of truth in `CLAUDE.md`; this file just redirects. |
| [`examples/`](./examples/) | Build logs and PR walkthroughs from real AI-augmented work. | The "show, don't tell" half. |

## How these files relate

I keep one source of truth (`CLAUDE.md`) and let every other agent point at it. Cursor and Copilot fall back to `CLAUDE.md` cleanly; tools that need their own filename (Codex CLI's `AGENTS.md`, Gemini CLI's `GEMINI.md`) get a one-line pointer. This stops me from drifting near-identical files out of sync.

## Principles I configure around

These are the opinions baked into the files in this repo. If you disagree with these, the configs will fight you — re-read them with this lens.

1. **The agent is a junior engineer with perfect recall and zero judgment.** My job is to give it the context and constraints that let it use that recall well. Configs exist to reduce the chance the agent does something I'd reject in code review.
2. **Spec the constraints, not the steps.** Configs say "trust internal callers — validate at system boundaries only" and "three concrete callers before you extract a helper" — not "first do X, then Y." Procedural prompts decay; principle-level rules don't.
3. **The why is more durable than the rule.** Every non-obvious rule has a `Why:` line. Future-me (and the agent) needs to know whether a rule still applies on a new edge case.
4. **Less is more.** Lead with a single-sentence project line. Keep architecture pointers tight — six to ten lines, not a tour. If a section needs more depth, push it to a `docs/` file and reference it from CLAUDE.md so the agent loads it on demand.
5. **Review the diff, not the prose.** I don't trust agent self-summaries. Configs encourage agent updates that are diff-checkable, not narrative.

## How to use this repo

1. Copy the relevant files into your project root.
2. Edit them — they're a starting point, not a drop-in. Most of the value is in tuning to your codebase's specifics (commands, test setup, style rules).
3. Iterate. After every session where the agent does something you'd reject, ask: *was this preventable with a rule?* If yes, add it. If no, move on.

## License

MIT. Take whatever's useful.
