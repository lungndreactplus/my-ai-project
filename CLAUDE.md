# CLAUDE.md — Agent Project Base

This file tells Claude Code how to operate inside this repository. Every agent (Manager, Explore, Plan, Implementer, Reviewer) MUST read it before acting.

**The authoritative rules live in [.specify/memory/constitution.md](.specify/memory/constitution.md).** This file is the operational handbook; the constitution is the law.

---

## Project Context — Monorepo Layout

This is a **two-app monorepo** with a 5-layer agent stack bolted on top of Spec-kit:

```
my-ai-project/
├── backend/                 # Rails 7.1 API-only (Ruby 3.3.0, PostgreSQL, UUID PKs)
├── frontend/                # Vue 3 or React SPA (Vite + Tailwind + centralized store)
├── evals/                   # L3 — promptfoo prompt-regression suite
├── .specify/
│   ├── memory/constitution.md   # SOURCE OF TRUTH for architecture rules
│   └── specs/<feature>/         # Per-feature Spec + Plan (SDD artifacts)
├── .claude/
│   ├── agents/              # L2 — persona subagents (pm, ba, architect, dev, qa)
│   └── skills/              # Slash-command skills (SDD workflow + git helpers)
├── .github/workflows/       # L5 — Claude constitution review on PR
├── .mcp.json                # L5 — MCP servers (GitHub, optional Linear)
└── CLAUDE.md                # This file
```

**Rule for agents:** when a task says "the backend" or "the API", work inside `backend/`. When it says "the UI" or "the frontend", work inside `frontend/`. Never mix paths across the two apps in a single file edit.

---

## Build & Install

### Backend (`backend/`)

```bash
cd backend
bundle install                     # install gems
bin/rails db:create db:migrate     # create + migrate PostgreSQL
bin/rails db:seed                  # optional seed data
cp .env.example .env               # then fill ANTHROPIC_API_KEY, JWT_SECRET_KEY, REDIS_URL
```

Run the server + workers:

```bash
bin/rails s                        # API on :3000
bundle exec sidekiq                # background jobs (requires Redis running)
```

### Frontend (`frontend/`)

```bash
cd frontend
npm install                        # install deps (or pnpm install / yarn)
cp .env.example .env               # set VITE_API_BASE_URL
npm run dev                        # Vite dev server
npm run build                      # production build
```

---

## Test & Quality Commands

### Backend — RSpec, Rubocop, Brakeman

```bash
cd backend
bundle exec rspec                          # full suite
bundle exec rspec spec/services            # service specs only
bundle exec rspec spec/path/to_spec.rb:42  # single example
bundle exec rubocop                        # lint — must pass with zero offenses
bundle exec rubocop -A                     # auto-fix safe offenses
bundle exec brakeman                       # security scan
bundle exec bundler-audit check --update   # CVE check on gems
```

**Per constitution § VII:** every new Service Object requires an RSpec. External HTTP (Claude, third-party) MUST be stubbed with WebMock or recorded with VCR — never hit real APIs in CI.

### Frontend — Vitest / Jest, ESLint, TypeScript

```bash
cd frontend
npm test                          # run test suite (Vitest or Jest)
npm run test:watch                # watch mode
npm run test -- path/to.test.ts   # single file
npm run lint                      # ESLint
npm run typecheck                 # tsc --noEmit
```

### Prompt Regression (L3) — promptfoo

```bash
cd evals
npm install                       # first time only — installs promptfoo
export ANTHROPIC_API_KEY=sk-...
npm run eval                      # run the golden suite
npm run view                      # open HTML report
npm run baseline                  # snapshot current scores (commit baseline.json)
```

QA **must** run `npm run eval` before approving merge. A > 10% score drop vs baseline on any persona prompt blocks the PR.

---

## Spec-Driven Development (MANDATORY)

This repo follows **SDD**: every change goes **Spec → Plan → Implement**. See constitution § I.

Agents MUST invoke the matching `speckit-*` skill at each phase instead of improvising the workflow.

### Skills in `.claude/skills/` — when to invoke each

| Phase | Skill | Use when the user asks to… |
|---|---|---|
| Constitution | `speckit-constitution` | amend or re-ratify `.specify/memory/constitution.md` |
| **1. Spec** | `speckit-specify` | start a new feature — produces `.specify/specs/<feature>/spec.md` (what + why, acceptance criteria) |
| Clarify | `speckit-clarify` | resolve open questions on an existing spec before planning |
| **2. Plan** | `speckit-plan` | turn an approved spec into `plan.md` (affected files, migrations, services, tests) — must cite constitution sections |
| Tasks | `speckit-tasks` | break the approved plan into discrete task items |
| Checklist | `speckit-checklist` | generate a pre-merge compliance checklist against the constitution |
| **3. Implement** | `speckit-implement` | execute an approved Plan's tasks — writes code + RSpec |
| Analyze | `speckit-analyze` | post-implementation: audit diff against spec + plan + constitution |
| Git — init | `speckit-git-initialize` | bootstrap git for a new speckit feature branch |
| Git — feature branch | `speckit-git-feature` | create/switch a branch named for the feature |
| Git — validate | `speckit-git-validate` | verify commits, branch state, and spec linkage before PR |
| Git — commit | `speckit-git-commit` | produce a spec-linked commit message |
| Git — remote | `speckit-git-remote` | push + open PR with spec/plan references |
| Tasks → Issues | `speckit-taskstoissues` | sync task list to GitHub issues |

### Required workflow for any non-trivial change

1. `speckit-specify` → write the spec. Do not skip. Trivial bug fixes (typo, nil-check) may skip this one step — everything else requires it.
2. `speckit-clarify` → if the spec has open questions.
3. `speckit-plan` → write the plan. Cite constitution principles (I–VII) for each architectural decision. Call out every affected file.
4. `speckit-tasks` → decompose the plan.
5. `speckit-implement` → write the code and RSpec. Run `bundle exec rspec` + `bundle exec rubocop` before declaring done.
6. `speckit-analyze` → audit the diff against spec + constitution.
7. `speckit-git-commit` / `speckit-git-remote` → commit and open PR.

**Agents MUST halt and request a spec** if asked to implement a feature that has no approved `spec.md` under `.specify/specs/`.

---

## Persona Subagents (L2) — BMAD-ported Team

Six persona subagents live in `.claude/agents/`. Four are ported from [BMAD-METHOD](https://github.com/bmad-code-org/BMAD-METHOD) (John, Winston, Amelia, Sally — keeping their named-character pattern and prompt structure); two (Riley, Murphy) are custom because BMAD does not ship a dedicated BA or QA persona. All map onto Spec-kit skills + this project's constitution.

| Persona | Character | File | Phase | Invokes skill | Next |
|---------|-----------|------|-------|---------------|------|
| **pm** | John (PM) | [.claude/agents/pm.md](.claude/agents/pm.md) | 1. Spec | `speckit-specify` | ba or architect |
| **ba** | Riley (BA) | [.claude/agents/ba.md](.claude/agents/ba.md) | 1b. Clarify | `speckit-clarify` | architect |
| **ux** | Sally (UX) | [.claude/agents/ux.md](.claude/agents/ux.md) | 1c. UX | `speckit-specify` (UX section) | pm or architect |
| **architect** | Winston (Architect) | [.claude/agents/architect.md](.claude/agents/architect.md) | 2. Plan | `speckit-plan`, `speckit-tasks`, `speckit-checklist` | dev |
| **dev** | Amelia (Senior SWE) | [.claude/agents/dev.md](.claude/agents/dev.md) | 3. Implement | `speckit-implement`, `speckit-git-commit` | qa |
| **qa** | Murphy (QA) | [.claude/agents/qa.md](.claude/agents/qa.md) | 4. Audit | `speckit-analyze`, runs `promptfoo eval` | git-remote |

**Full chain for a new feature:** `pm → (ba) → (ux) → architect → dev → qa → git-remote`. Dev MUST halt if spec or plan is missing. Each persona embodies the named character (John/Riley/Sally/Winston/Amelia/Murphy) and stays in character until dismissed.

## Manager Orchestration — Shortcut Routing

When the user message contains a shortcut, the Manager agent routes to a subagent via the `Agent` tool:

| Shortcut | `subagent_type` | Purpose |
|----------|-----------------|---------|
| `#pm` / `#john` / `#spec` | `pm` | John drafts a new spec from a user intent |
| `#ba` / `#riley` / `#clarify` | `ba` | Riley resolves `[NEEDS CLARIFICATION]` in a spec |
| `#ux` / `#sally` | `ux` | Sally authors the UX section of the spec |
| `#architect` / `#winston` / `#design` | `architect` | Winston writes plan.md + tasks.md |
| `#dev` / `#amelia` / `#implement` | `dev` | Amelia executes the plan TDD-first |
| `#qa` / `#murphy` / `#audit` | `qa` | Murphy audits diff + runs the eval suite |
| `#explore` / `#find` / `#search` | `Explore` | Codebase exploration, file/symbol search |
| `#plan` | `Plan` | Broad architecture brainstorming (not SDD — use `#architect` for SDD plan) |
| `#research` / `#investigate` / `#task` | `general-purpose` | Multi-step research spanning many files |
| `#review` / `#status` | `general-purpose` | Code review, ship-readiness audit |
| `#claude` / `#sdk` / `#api` | `claude-code-guide` | Questions about Claude Code CLI / Agent SDK / Anthropic API |
| `#statusline` | `statusline-setup` | Configure Claude Code status line |
| `#parallel` | multiple agents | Fan out independent subtasks concurrently |

> **`#plan` vs `#architect`**: `#plan` is for open-ended brainstorming ("how should we approach X?"). `#architect` is for writing `plan.md` inside an approved SDD feature.

**Modifiers for `Explore`:** `:quick`, `:medium` (default), `:thorough`.
**Isolation modifier for any agent:** `:worktree` runs in an isolated git worktree.

Example: `#explore:thorough Auth::JwtService usage` or `#research:worktree migrate jobs to ActiveJob`.

### Dispatch rules

1. **Always brief the agent fully.** Use: GOAL / CONTEXT / SCOPE / DELIVERABLE (with length cap).
2. **Never delegate understanding.** Synthesize the agent's output before replying to the user.
3. **Parallelize independent work** — one message, multiple `Agent` calls.
4. **Confirm risky actions** (force push, `reset --hard`, DB drop, branch deletion) before delegating.

---

## Non-Negotiable Constitutional Rules (summary)

These come from [.specify/memory/constitution.md](.specify/memory/constitution.md). Read the full document for detail.

- **§ III** — Every DB table uses UUID primary keys and UUID FKs. Migrations without `id: :uuid` are rejected.
- **§ IV** — All business logic lives in `app/services/<domain>/`, inherits `BaseService`, returns `ServiceResult`. Controllers stay thin.
- **§ V** — Every call to Claude AI goes through `AI::ClaudeClient` (Singleton). No direct SDK/HTTP calls from controllers, jobs, or other services.
- **§ VI** — Frontend uses Tailwind only, centralized store (Pinia / Zustand / Redux Toolkit), and one shared `apiClient` wrapper. No direct `fetch`/`axios` in components.
- **§ VII** — RSpec mandatory for new services. Rubocop clean. External HTTP stubbed with WebMock/VCR.

**Auto-reject patterns in review:**
- PORO under `app/services/` that does not inherit `BaseService`.
- Any `Anthropic::Client` / `Faraday.post` to Claude outside `AI::ClaudeClient`.
- Migration missing `id: :uuid`.
- Business logic in a controller action.
- New gem or npm dependency added without Plan-level justification.

---

## Communication Style for Agents

- State the routing / skill decision in one short sentence before invoking the tool.
- After subagents return, summarize in ≤100 words.
- Use markdown file links for references: `[file.rb:42](backend/app/services/file.rb#L42)`.
- Do not narrate internal reasoning. Do not trail responses with a summary of what was obviously just done.

---

## CI / Ops (L5)

- **`.github/workflows/claude-code-review.yml`** — runs the official `anthropics/claude-code-action` on every PR. Reviews diff against the constitution (§ III, IV, V, VI, VII) and the feature's `plan.md`. Block merge on violation. Requires `secrets.CLAUDE_CODE_OAUTH_TOKEN`.
- **`.mcp.json`** — registers the GitHub MCP server so agents can create/read issues, comment on PRs, and open PRs. Requires `GITHUB_TOKEN` env var. Add Linear / Jira MCP here if the team uses them.
- Task sync: the `speckit-taskstoissues` skill pushes `tasks.md` → GitHub Issues via the GitHub MCP.

## Deferred Layers

- **L4 Observability** (Langfuse tracing on `AI::ClaudeClient`) — add once `backend/` is scaffolded. Wrap `chat(...)` in a Langfuse span capturing input/output/token usage/cost. See constitution § V for the single integration point.
- **L1 Execution Engine** (TDD hooks, worktree isolation for parallel features) — add once ≥ 3 features run in parallel or the team grows. Worktree isolation available today via `isolation: "worktree"` on the `Agent` tool.

## No-Shortcut Fallback

If the user message has no shortcut:
- Trivial question → answer directly.
- New feature request → auto-route to `pm`.
- Spec exists with unclear parts → auto-route to `ba`.
- Spec approved, no plan → auto-route to `architect`.
- Plan + tasks approved, no code → auto-route to `dev`.
- Dev declares done → auto-route to `qa`.
- Questions about Claude Code CLI / SDK / Anthropic API → auto-route to `claude-code-guide`.
- Non-trivial codebase research (> 3 searches) → auto-route to `Explore`.
