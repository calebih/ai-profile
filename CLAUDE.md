# CLAUDE.md — annotated, drawn from my working configs

> This file is loaded automatically by Claude Code at session start. It's the single most important config in this repo because it shapes every interaction.
>
> Below is a paste-ready template, instantiated for a representative codebase ("Concord" — a fictional multi-tenant ops platform), followed by section-by-section commentary explaining why each part is shaped the way it is. Drop your real project's specifics into the template; the *structure* is what's portable.
>
> **Length matters.** The agent processes this on every prompt. A 200-line CLAUDE.md gets skimmed; a 60–100-line one gets followed.

---

## The template

```md
# Project: Concord — multi-tenant ops platform; .NET 9 backend, React + AngularJS frontend; multi-repo umbrella.

## Personal context

Read `~/vault/CLAUDE.md` at session start for cross-project conventions and identity (work-item query strings, ADO display name, preferred tools). When searching for notes/memory, prefer the Obsidian CLI (`obsidian search`, `obsidian backlinks`) over `grep` across the vault. The vault is the single source of truth — write session summaries there, not into project memory.

## Workspace layout

This is a multi-repo umbrella, not a single solution. There's no root build — work *inside* the target repo:

- `concord-web/` — main backend monolith (`Concord.sln`, ASP.NET Core)
- `concord-web-ui/` — frontend monolith (React + legacy AngularJS, TypeScript)
- `concord-modules-*/` — microservice modules (.NET 9; one per bounded context)
- `concord-app-*/` — third-party integration apps
- `concord-engine-*/` — sync engines
- `concord-local-dev/` — local environment orchestration (Tilt-based)

Some sub-projects have their own nested `CLAUDE.md`. Claude Code loads them automatically when you work in that directory — read those when you're inside them.

## Build & test commands

**.NET module (`concord-modules-*`):**
```powershell
$env:ACCESS_TOKEN = (az account get-access-token --query accessToken --out tsv)
dotnet build application/*.sln
dotnet test application/*.sln
```

**Main monolith:**
```powershell
cd concord-web
dotnet build Concord.sln
dotnet test Concord.sln
```

**Frontend (from `concord-web-ui/src`):**
```powershell
npm install
npm run build-modules
npm test
```

**Local dev environment:**
```powershell
cd concord-local-dev
.\dev.ps1 up        # spins up SQL/Mongo/Redis/RabbitMQ/ES via Tilt
.\dev.ps1 status
```

Run the relevant suites before declaring work done. If you can't run them, say so explicitly — don't claim success on a green build alone.

## Architecture pointers

- Domain logic lives in `Concord.Modules.<X>.Services/` — pure C#, no I/O.
- Operation handlers (`IOperationHandler<TRequest, TResponse>`) in `Concord.Modules.<X>.Operations/`.
- Cross-module calls go through `Concord.Modules.<X>.ApiClient/` (extends `ApiClientBase`, uses `JWTAuthorizationDelegatingHandler`). Inject `IApiClient` via DI; never `new` it.
- DI registration order matters: `AddDataRepositories(cfg)` → `AddFrameworkServices(cfg)` → `AddApplicationServices(cfg)` → `AddOperationHandlers()`.
- New frontend work goes in React. Touch AngularJS only when extending legacy modules.
- DB migrations: additive only (NOT NULL → backfill → constraint, never destructive).

## Conventions

- Trust internal callers. Validate at system boundaries (HTTP, DB, external APIs) only.
- No comments unless the *why* is non-obvious. Don't restate what the code does.
- New abstractions require three concrete callers. Don't pre-abstract.
- Errors that can't happen don't need handling. Crash-loud over silent-fallback.
- File-scoped namespaces, nullable reference types, tab indentation, ~80-char lines.
- Frontend: Prettier (tabs width 2, single quotes, trailing commas), ESLint (semicolons, 120-char lines).

## Tooling rules

- For work-item queries, use the issue-tracker MCP server (ADO / Jira / Linear) — don't browse the tracker UI.
- For wiki lookups, use the Confluence MCP plugin (or `scripts/confluence.ps1` fallback) before web-browsing.
- For cross-repo code navigation, consult the internal code-intelligence wiki first — it tracks fresher than local checkouts.
- For "my active work items," filter on the assigned identity from the personal context file, excluding terminal states (Closed, Resolved, Done, Removed).

## Commit & PR conventions

- Commit subject prefix: `[WI#1234] <imperative summary>` (work-item ID lives in brackets).
- One logical change per commit. Squash WIP before PR.
- PR body must include: problem statement, linked work-item, test evidence (paste `dotnet test` / `npm test` summary), screenshots for UI behavior changes.

## What NOT to do

- Don't add backwards-compat shims for in-progress work. Just change the code.
- Don't suggest new third-party libraries without flagging them for security/license review.
- Don't touch authentication, authorization, payment, or encryption code without explicit approval — these require senior sign-off.
- Don't `git push` or merge PRs unless I explicitly ask. Local commits are fine.

## Verification before claiming done

When you say "this is done":
- Run the build and the relevant test suites. Report actual output, not summaries.
- For frontend changes: start the dev server, use the feature in a browser, test the golden path *and* one edge case.
- If you can't actually run something, say so plainly. False confidence costs more than an extra check.

## Active context (kept current)

- Current branch: `<feature/...>`
- Open question: `<thing I'm trying to decide>`
- Known broken: `<thing I haven't fixed yet that the agent might trip on>`
```

---

## Why each section is shaped that way

**Project line (1 sentence):** Anchors the agent on what the codebase *is* before it reads anything else. One sentence forces clarity. If I can't compress to a sentence, the project's identity is fuzzy — fix that first.

**Personal context (vault pointer):** I keep cross-project state — work-item identities, query strings, preferred tools, ongoing context — in an Obsidian vault, not in this file. The vault is the single source of truth; project memory is a per-codebase view onto it. The agent reads the vault `CLAUDE.md` at session start so it inherits *who I am* before it learns *what this project is*.

**Workspace layout:** A multi-repo umbrella is normal at scale. The agent will guess wrong about where to run commands ("`npm test` from the workspace root" — fails) unless this section is explicit. The "work inside the target repo" line is doing real work — it converts the agent's default of running things from CWD into thinking about which repo it's actually in.

**Nested CLAUDE.md:** Claude Code automatically loads sub-CLAUDE.md files when you work in a child directory. I use this for module-specific rules (e.g., a tickets module might have shadow-testing-specific rules that don't apply elsewhere). Don't inline those at the top — the umbrella file would balloon.

**Build & test commands:** The agent will run these. If they're missing, it'll guess and fail loud (`npm test` instead of `npm --prefix web test`, etc.). The Azure `ACCESS_TOKEN` dance is the kind of detail that gets the agent unstuck on day one — without it, the first NuGet restore fails opaquely.

**Architecture pointers:** Three to seven lines of "where things live + the rules that decide where new code goes." The DI registration order is here because it's a real source of bugs when an agent registers things in the wrong order — it compiles, then explodes at runtime. Spelling it out costs five lines and saves an investigation.

**Conventions:** These are the rules I'd give a new hire on day one — stated as principles, not commandments. *"Trust internal callers"* explicitly disables the agent's default to add defensive checks everywhere, which is a real failure mode.

**Tooling rules:** This is the section that makes cloud agents work. Without it, the agent burns tokens browsing UIs to look up work items or wiki pages instead of using the MCP servers that already give it structured access. The "code-intelligence wiki tracks fresher than local checkouts" line is non-obvious — local clones go stale the moment you stop pulling, and a daily-ingested wiki is more current than what's on most engineers' machines.

**Commit & PR conventions:** Work-item-prefixed commits make rollback and audit trivial. The PR body checklist is what I'd ask of any human contributor — having it here means the agent self-checks before opening the PR.

**What NOT to do:** This is the most valuable section. It's where I encode the failure modes I keep seeing. Every line started as a frustration:

- *"Don't add backwards-compat shims"* — agents love writing `// removed for backwards compat` cruft. Killing this saves PRs from looking like museum pieces.
- *"Senior sign-off"* on auth/payment/encryption — matches employer policy AND it's good hygiene.
- *"Don't push or merge"* — non-negotiable. Some agents helpfully push. Tell them not to.

**Verification before claiming done:** Agents are optimistic by default. They'll claim a feature works because the unit tests pass, even when the UI is broken. The "start the dev server, use the feature in a browser" line forces actual verification for frontend work. The "say so plainly" line gives the agent permission to admit it can't verify — without it, you get false success claims.

**Active context:** The only mutable section. I update it weekly. The agent uses it to know what I'm in the middle of and what to avoid stepping on. If you don't keep this current, delete it — stale context is worse than none.

---

## Anti-patterns I see in other people's CLAUDE.md files

- **The kitchen sink.** 400 lines covering every code path. The agent skims past it.
- **The aspirational rulebook.** Rules nobody actually follows ("100% test coverage required"). The agent will call you on it.
- **The vague principle.** *"Write clean code."* Useless. Be specific or omit.
- **The narrative tour.** Paragraphs of prose explaining the system's history. Move that to a `docs/architecture.md` and *reference* it from CLAUDE.md if needed.
- **No `Why:` lines.** When a rule is non-obvious, the agent (and a future maintainer) needs to know whether it still applies on edge cases.

## When to reference other files

If your project has:

- Long-form architecture docs → put them in `docs/` and reference: *"See `docs/architecture.md` if touching the event pipeline."*
- Workflow-specific rules (e.g. how to do migrations) → put them in `references/` and the agent will load them on demand. Don't inline.
- Per-subdir conventions → put a smaller `CLAUDE.md` in that subdir. Claude Code automatically loads nested ones.

## Review cadence

I re-read this file every 2 weeks. Things to check:

- Does any rule still trip the agent? If yes, sharpen it.
- Did the agent surprise me (good or bad)? If a good behavior emerged, encode it. If a bad one, prevent it.
- Did "Active context" go stale? Update or delete.
