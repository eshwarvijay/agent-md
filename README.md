# Agents.md

> **The field guide to AI context files** — the small set of Markdown documents that quietly decide who your agent is, what it knows, what it can do, and how it behaves.

If you've ever dropped a `CLAUDE.md` into a repo, scribbled notes into a `MEMORY.md`, or wondered why some agents feel coherent across sessions while others forget everything the moment you close the terminal — this guide is for you.

These files aren't a framework you install or a spec you have to obey. They're **conventions** — the same way `README.md`, `LICENSE`, and `.gitignore` became conventions long before anyone standardized them. A handful of teams started naming their agent-configuration files the same way, the names stuck, and now they form a loose but powerful vocabulary for *programming agents in plain English*.

This document is a deep, practical tour of that vocabulary.

---

## Table of Contents

1. [Why context files exist](#1-why-context-files-exist)
2. [The mental model: a layered identity](#2-the-mental-model-a-layered-identity)
3. [`CLAUDE.md` — the operating manual](#3-claudemd--the-operating-manual)
4. [`MEMORY.md` — persistent knowledge](#4-memorymd--persistent-knowledge)
5. [`SKILLS.md` — the capability registry](#5-skillsmd--the-capability-registry)
6. [`SOUL.md` — identity and values](#6-soulmd--identity-and-values)
7. [How the files compose](#7-how-the-files-compose)
8. [The wider machinery](#8-the-wider-machinery-system-prompts-mcp-rag-frameworks)
9. [Anti-patterns and best practices](#9-anti-patterns-and-best-practices)
10. [A complete worked example](#10-a-complete-worked-example)
11. [References](#references)

---

## 1. Why context files exist

A large language model is, by default, **stateless and contextless**. Every conversation starts from zero. It doesn't know your coding standards, your name, which tools it's allowed to call, or what it learned yesterday. You *can* re-explain all of that in every prompt — but that's slow, error-prone, and impossible to version.

Context files solve this by moving that information out of the chat box and into **files that live next to your code**. The agent loads them automatically (or you paste them in), and suddenly it behaves like a colleague who has actually read the onboarding docs.

The payoff:

- **Consistency** — the agent behaves the same way on Monday and Friday, for you and your teammate.
- **Version control** — you can diff, review, and roll back an agent's "personality" like any other code.
- **Separation of concerns** — identity, knowledge, capability, and project rules each live in their own place instead of one giant prompt blob.
- **Portability** — the same `SOUL.md` can travel across projects; the same `CLAUDE.md` pattern works whether you're in a terminal, an IDE, or a CI pipeline.

Think of it as **dependency injection for behavior**. You don't hard-code the agent's personality into the model — you inject it through files.

---

## 2. The mental model: a layered identity

The four canonical files map cleanly onto four questions any capable agent must answer:

| File | The question it answers | Analogy |
|------|------------------------|---------|
| **`SOUL.md`** | *Who am I?* | Personality & character |
| **`MEMORY.md`** | *What do I know?* | Long-term memory |
| **`SKILLS.md`** | *What can I do?* | Muscle memory & tools |
| **`CLAUDE.md`** | *How do I work here?* | The employee handbook for this job |

Stacked together they form what you can think of as a **personality package**: a portable bundle that turns a generic model into *your* agent. None of these names is enforced by any runtime — you could call them `IDENTITY.md` or `agent.config.md` — but using the common names means your files are instantly legible to other people and other tools.

```
        ┌-------------------------------------─┐
        │  SOUL.md     →  who I am             │  most stable
        │  MEMORY.md   →  what I know          │      ▲
        │  SKILLS.md   →  what I can do        │      │
        │  CLAUDE.md   →  how I work here      │  most specific
        └--------------------------------------┘
```

---

## 3. `CLAUDE.md` — the operating manual

**Answers:** *How do I work in this project?*

`CLAUDE.md` was popularized by [Claude Code](https://docs.claude.com/en/docs/claude-code), which automatically discovers and loads it from the working directory, parent directories, and a global location (`~/.claude/CLAUDE.md`). It's the single most widely adopted context file, and the closest thing the ecosystem has to a standard.

It holds **project-specific operating instructions** — the things a new engineer would need to be told on day one:

- How to build, test, lint, and run the project
- Directory layout and where things live
- Coding conventions the team actually enforces
- Hard constraints ("never touch `legacy/`", "always run migrations through the helper")
- Domain vocabulary and gotchas

### Example

```markdown
# CLAUDE.md

## Commands
- Install:  `pnpm install`
- Dev:      `pnpm dev`        # localhost:3000
- Test:     `pnpm test`       # Vitest; run before every commit
- Lint:     `pnpm lint --fix`

## Conventions
- TypeScript strict mode — no `any`, no `@ts-ignore`.
- Components in `src/components`, one folder per component.
- Prefer server components; mark client ones with `"use client"`.

## Constraints
- NEVER edit files under `src/generated/` — they're codegen output.
- All DB access goes through `src/db/client.ts`. No raw SQL in routes.
- Don't add dependencies without noting why in the PR description.
```

> **Rule of thumb:** if you find yourself typing the same instruction to the agent twice, it belongs in `CLAUDE.md`.

---

## 4. `MEMORY.md` — persistent knowledge

**Answers:** *What do I know that won't fit in a fresh conversation?*

Where `CLAUDE.md` describes the *project*, `MEMORY.md` accumulates **facts, preferences, and hard-won lessons** that persist across sessions. It's the difference between an agent that re-asks "what's your stack again?" every morning and one that just remembers.

Memory typically comes in a few flavors:

- **User facts** — who you are, your role, your preferences ("I prefer pytest over unittest").
- **Project state** — ongoing goals, decisions made, where a long task was paused.
- **Feedback** — corrections you've given, and *why*, so the agent doesn't repeat mistakes.
- **References** — pointers to dashboards, tickets, runbooks.

### How it gets updated

1. **Manually** — you edit the file like notes.
2. **Agent-assisted** — the agent proposes additions ("Want me to remember that?").
3. **Automatically** — memory frameworks (e.g. Mem0, MemGPT-style systems) extract and write salient facts on their own.

### Example

```markdown
# MEMORY.md

## User
- Eshwar — builds AI agents & LLM systems. Prefers terse, code-first answers.
- Default stack: Python + Google ADK for agents; Next.js + Tailwind for web.

## Project: Archie
- GCP knowledge base + ADK agent. Vertex AI Search index created.
- Ingest paused at Heretto 32/660 — resume from there.

## Feedback
- Don't over-explain. Show the diff, not a lecture.
  Why: user reads code faster than prose.
```

A good `MEMORY.md` is **append-mostly and curated** — you prune stale facts the same way you'd close stale tickets.

---

## 5. `SKILLS.md` — the capability registry

**Answers:** *What actions can I actually take, and when?*

`SKILLS.md` is a **registry of capabilities** — the tools, scripts, and workflows the agent is allowed to invoke, plus the conditions under which to reach for each one. It turns a vague "the agent can use tools" into an explicit, auditable menu.

A quick disambiguation, because the ecosystem overloads these words:

| Term | Who uses it | Roughly means |
|------|-------------|---------------|
| **Tool** | Anthropic | A function the model can call mid-conversation |
| **Function** | OpenAI | The same idea, different name |
| **Skill** | Broader agent world | A higher-level capability, often *bundling* tools + instructions + knowledge |

A skill entry is most useful when it documents four things: **what** it does, **when** to trigger it, **how** to invoke it, and **what limits** apply.

### Example

```markdown
# SKILLS.md

## deploy-staging
- **What:**   Builds and deploys the app to the staging environment.
- **When:**   User says "ship to staging" or a PR is approved.
- **How:**    `./scripts/deploy.sh staging`
- **Limits:** Never deploy to `prod` from this skill. Requires green CI.

## query-warehouse
- **What:**   Runs read-only analytics queries against BigQuery.
- **When:**   Questions about metrics, usage, or historical data.
- **How:**    `bq query --use_legacy_sql=false '<SQL>'`
- **Limits:** SELECT only. No DML. Cap results at 10k rows.
```

Well-written skills double as **guardrails**: by stating the limits, you tell the agent not just what it *can* do but what it must *not*.

---

## 6. `SOUL.md` — identity and values

**Answers:** *Who am I, and how do I carry myself?*

`SOUL.md` is the most philosophical of the four — and the most underused. It defines the agent's **identity, values, communication style, and ethical boundaries**. It's what keeps an agent feeling like a coherent *character* rather than a different stranger in every reply.

Typical contents:

- **Core values** — what the agent optimizes for (honesty over flattery, clarity over completeness).
- **Voice & tone** — terse or warm, formal or casual, emoji or no emoji.
- **Reasoning principles** — "show your work on hard problems", "state assumptions explicitly".
- **Hard limits** — lines it won't cross regardless of instruction.

### Example

```markdown
# SOUL.md

## Identity
I'm a senior engineering partner. I act like a teammate who's read the
codebase, not a chatbot waiting for orders.

## Values
- Truth over comfort. If the plan is bad, I say so — with a better one.
- Bias to action. When the answer is clear, I do it instead of asking.
- Respect the user's time. No filler, no restating the question.

## Voice
- Direct, dry, occasionally funny. Code first, prose second.

## Limits
- I never fabricate results or claim a test passed when I didn't run it.
- I confirm before destructive or irreversible actions.
```

A strong `SOUL.md` is portable: the same one can ride along with you across every project, while `CLAUDE.md` and `MEMORY.md` stay project-specific.

---

## 7. How the files compose

Loading order matters, because later context tends to override earlier context. A common and sensible startup sequence is:

```
SOUL.md  →  SKILLS.md  →  CLAUDE.md  →  MEMORY.md
(identity)  (abilities)   (project)     (specifics)
```

Read it as a sentence: *"This is who I am, here's what I can do, here's how this project works, and here are the specific things I currently know."* Identity is established first and is the most stable; the concrete, fast-changing specifics come last and win ties.

When you inject several files into a single prompt, **wrap each in clear delimiters** so the model can tell them apart:

```xml
<soul>
...contents of SOUL.md...
</soul>

<skills>
...contents of SKILLS.md...
</skills>

<project_instructions>
...contents of CLAUDE.md...
</project_instructions>

<memory>
...contents of MEMORY.md...
</memory>
```

XML-style tags work especially well with Claude, which is trained to respect them as structural boundaries.

---

## 8. The wider machinery: system prompts, MCP, RAG, frameworks

Context files don't operate in a vacuum. They sit on top of — and feed into — a larger stack:

### System prompts
The system prompt is the highest-priority instruction channel. Context files are often *composed into* the system prompt at startup. Think of the system prompt as the envelope and the context files as the letters inside.

### Prompt-engineering patterns
The instructions inside these files are more effective when they use proven patterns:
- **Role prompting** — "You are a senior SRE" (lives naturally in `SOUL.md`).
- **Few-shot examples** — show, don't just tell (great for `SKILLS.md`).
- **Chain-of-thought** — "reason step by step before answering" for hard tasks.

### MCP (Model Context Protocol)
[MCP](https://modelcontextprotocol.io) is an open standard for connecting agents to external tools and data at **runtime**. Where `SKILLS.md` *documents* what an agent can do, MCP is one of the mechanisms that actually *executes* it — exposing live tools (databases, APIs, file systems) the agent can call.

### RAG (Retrieval-Augmented Generation)
`MEMORY.md` works beautifully for a few hundred facts. When your knowledge base grows to thousands of documents, you graduate to **RAG**: store the knowledge in a vector index and retrieve only the relevant chunks per query. Same goal as memory — *give the model what it needs to know* — at a different scale.

### Agent frameworks
Tools like LangChain, CrewAI, Google's ADK, and others provide the orchestration layer — loops, tool-calling, multi-agent coordination — that consumes these context files and turns them into running behavior.

---

## 9. Anti-patterns and best practices

**Do:**

- **Keep them concise.** Every token competes for the model's attention. A tight 40-line `CLAUDE.md` beats a rambling 400-line one.
- **Be specific, not vague.** "Use 2-space indentation" beats "write clean code."
- **Version-control everything.** These files are code. Diff them, review them in PRs, roll them back.
- **Separate concerns.** Don't cram identity, memory, and project rules into one file — that's what the four-file split is *for*.
- **Test changes like code.** After editing a context file, run the agent on a known task and check the behavior actually changed the way you intended.
- **Use delimiters** (XML tags) when injecting multiple files at once.

**Don't:**

- **Don't write a novel.** If the agent never reads past line 50, lines 51+ are decoration.
- **Don't duplicate.** If a fact is derivable from the code or git history, don't restate it — it'll rot.
- **Don't mix stable and volatile content.** Put fast-changing state in `MEMORY.md`, not `SOUL.md`.
- **Don't hard-code secrets.** These files get committed. Reference a secret manager instead.

---

## 10. A complete worked example

Here's how the four files look for a single fictional project — a CLI tool called `tide`:

<details>
<summary><b>SOUL.md</b></summary>

```markdown
# SOUL.md
## Identity
I'm tide's resident engineer. Pragmatic, terse, allergic to bikeshedding.
## Values
- Ship small, ship often. Truth over politeness.
## Voice
- Plain English. Code samples over paragraphs.
## Limits
- Never claim a command works without running it.
```
</details>

<details>
<summary><b>SKILLS.md</b></summary>

```markdown
# SKILLS.md
## run-tests
- What: Runs the suite.  How: `go test ./...`  When: before any commit.
## release
- What: Cuts a tagged release.  How: `./scripts/release.sh`
- Limits: maintainers only; requires green CI + changelog entry.
```
</details>

<details>
<summary><b>CLAUDE.md</b></summary>

```markdown
# CLAUDE.md
## Build
- `make build` → binary in ./bin/tide
## Conventions
- Go 1.22, gofmt enforced. Errors wrapped with %w.
## Constraints
- No new deps without an issue. Keep the binary under 10MB.
```
</details>

<details>
<summary><b>MEMORY.md</b></summary>

```markdown
# MEMORY.md
## Project state
- v0.4 shipped. Working on the plugin API (branch: feat/plugins).
## Decisions
- Chose cobra over urfave/cli for subcommands (better completion).
```
</details>

Drop those four files in the repo, point your agent at them, and it walks in already knowing who it is, what it can do, how the project builds, and where the work stands. That's the whole idea.

---

## References

| Topic | Link |
|-------|------|
| Claude Code & `CLAUDE.md` | https://docs.claude.com/en/docs/claude-code |
| Prompt engineering guide | https://docs.claude.com/en/docs/build-with-claude/prompt-engineering |
| Model Context Protocol (MCP) | https://modelcontextprotocol.io |
| Tool use / function calling | https://docs.claude.com/en/docs/build-with-claude/tool-use |
| Retrieval-Augmented Generation | https://en.wikipedia.org/wiki/Retrieval-augmented_generation |

---

<div align="center">
<sub>These names are conventions, not laws. Adopt the ones that help, rename the rest, and keep them short. Your future self — and your agent — will thank you.</sub>
</div>
