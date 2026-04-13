# Agent Project Base Constitution

> This constitution is the **source of truth** for all technical decisions in this repository.
> Every Agent (Manager, Explore, Plan, Implementer, Reviewer) **MUST** read and comply with this document before producing any output that affects code, specs, or plans.
> Deviations require an explicit amendment (see Governance).

---

## Core Principles

### I. Spec-Driven Development (NON-NEGOTIABLE)

Every change flows through three mandatory gates — in this order:

1. **Spec** — A written specification under `.specify/specs/<feature>/spec.md` describing *what* and *why* (user story, acceptance criteria, out-of-scope). No implementation details.
2. **Plan** — A technical plan under `.specify/specs/<feature>/plan.md` describing *how* (affected files, data model changes, service objects, migrations, tests). Must cite this constitution for every architectural decision.
3. **Implement** — Code changes that execute the approved Plan. Any divergence from the Plan must be reflected back into the Plan before merge.

**Rules:**
- No direct code edits without an approved Spec + Plan — including "small fixes" that touch business logic.
- Agents that receive a coding task without a Spec must **halt and request one**, not improvise.
- Bug fixes with a trivial root cause (typo, missing nil-check) may skip Spec but still require a one-paragraph Plan.

### II. Rails API-Only Backend (Ruby 3.3.0 / Rails 7.1)

The backend is a **Rails 7.1 API-only** application on **Ruby 3.3.0** with **PostgreSQL**. This is fixed.

- No views, no ActionMailer, no ActionText, no ActiveStorage (project was generated with `--api --skip-*`).
- Dependencies are pinned in `Gemfile`; adding a new gem requires Plan-level justification.
- Background work goes through **Sidekiq** + **Redis**. Do not introduce a second job runner.
- Authentication is **JWT via `Auth::JwtService`**; authorization is **Pundit**. Do not bypass either.

### III. UUID Primary Keys (NON-NEGOTIABLE)

Every table **MUST** use UUID primary keys and UUID foreign keys.

- Migrations must declare `create_table :xxx, id: :uuid do |t|` and `t.references :yyy, type: :uuid`.
- Never use integer IDs, never expose sequential IDs in URLs or payloads.
- Reviewers must reject any migration that omits `id: :uuid`.

### IV. Service Objects Over Fat Models/Controllers (NON-NEGOTIABLE)

All business logic lives in **Service Objects** under `app/services/`.

- Controllers are thin: parse params → call a service → render result. No business rules.
- Models hold validations, associations, scopes, and simple domain methods — **no orchestration, no external calls**.
- Every service **MUST**:
  1. Inherit from `BaseService` (defined at `app/services/base_service.rb`).
  2. Expose a `.call(...)` class method (provided by `BaseService`).
  3. Return a **`ServiceResult`** object — never raw booleans, never raw models, never raise for control flow.
  4. Use `success(data)` / `failure(errors, code:)` helpers from `BaseService`.
- Services are namespaced by domain: `Auth::`, `Agents::`, `Tools::`, `AI::`, `Conversations::`.
- Controllers consume results via `result.on_success { ... }.on_failure { ... }` or `result.success?` — they do not reach into service internals.

### V. Single Claude Integration Point — `AI::ClaudeClient` (NON-NEGOTIABLE)

**All** interaction with Claude AI **MUST** go through `AI::ClaudeClient` (at `app/services/ai/claude_client.rb`).

- No controller, job, model, or other service may call the Anthropic SDK, Faraday, or the Claude HTTP API directly.
- `ClaudeClient` is a Singleton. It owns: retries, timeouts, streaming, token accounting, error normalization (`ClaudeClient::ApiError`).
- Supporting collaborators — `AI::PromptBuilder`, `AI::ResponseParser`, `AI::TokenCounter`, `AI::StreamingHandler` — live in the same namespace and are the only classes allowed to touch Claude-specific payload shapes.
- New agent behaviors (Claude, OpenAI fallback, future providers) go through `Agents::BaseAgent` subclasses that delegate to `AI::ClaudeClient`. Do not fan providers out across the codebase.
- Default model: `claude-sonnet-4-20250514`. Allowed models are the whitelist in `AgentConfig::AVAILABLE_MODELS`.

### VI. Frontend: Component Framework + Tailwind + Centralized State

The frontend uses **Vue 3 (Composition API)** or **React (Hooks)** — pick one per app and stay consistent.

- **Styling:** **Tailwind CSS** only. No ad-hoc CSS files, no CSS-in-JS, no second UI kit unless added via amendment.
- **State management:** centralized store — **Pinia** for Vue, **Zustand** or **Redux Toolkit** for React. Component-local `ref`/`useState` is allowed for purely local UI state; anything shared across routes/components goes in the store.
- **API access:** one shared HTTP client wrapper (e.g. `src/lib/apiClient.ts`) that attaches JWT, handles 401 refresh, and normalizes errors. Components do not call `fetch`/`axios` directly.
- **Server state:** use TanStack Query (React) or `@tanstack/vue-query` (Vue) for server-cache concerns; do not reinvent caching in the store.
- **Routing:** Vue Router / React Router with route-level code splitting.

### VII. Testing & Code Quality (NON-NEGOTIABLE)

- **RSpec is mandatory** for every new Service Object. A service without a spec is not considered complete and must not be merged.
- Spec coverage expected: happy path, each documented failure mode, and any side effects (DB writes, job enqueues, external calls stubbed with WebMock/VCR).
- Models: validations and non-trivial methods require specs.
- Controllers: request specs for each endpoint.
- External HTTP (Claude, third-party) **must** be stubbed with WebMock or recorded with VCR — never hit real APIs in CI.
- **Rubocop** (with `rubocop-rails` + `rubocop-rspec`) must pass with zero offenses before merge. No `# rubocop:disable` without a one-line justification comment.
- **Brakeman** and **bundler-audit** run in CI; high-severity findings block merge.
- Frontend: lint (ESLint) + typecheck (TypeScript) must pass; component tests for shared components.

---

## Technology Stack (Locked)

| Layer | Choice | Notes |
|---|---|---|
| Ruby | 3.3.0 | pinned in `.ruby-version` |
| Backend framework | Rails 7.1 API-only | `--api`, no views/mailer/storage |
| Database | PostgreSQL | UUID PKs everywhere |
| Background jobs | Sidekiq 7 + Redis | `ApplicationJob` base |
| Auth | JWT (`Auth::JwtService`) + Devise + Pundit | access + refresh token pair |
| AI SDK | `anthropic` gem via `AI::ClaudeClient` | Singleton, Faraday + retry |
| Serialization | `jsonapi-serializer` | one serializer per exposed model |
| Validation | `dry-validation` / `dry-types` for complex input | model validations for DB-level |
| Events | `wisper` | domain events from services |
| Rate limit | `rack-attack` | configured in initializer |
| Monitoring | `lograge` + Sentry | structured logs |
| Testing | RSpec, FactoryBot, Faker, WebMock, VCR, SimpleCov | |
| Lint/Security | Rubocop, Brakeman, bundler-audit | CI-enforced |
| Frontend | Vue 3 (Composition) **or** React (Hooks) | one per app |
| Frontend build | Vite | |
| Styling | Tailwind CSS | exclusive |
| State | Pinia / Zustand / Redux Toolkit | centralized |
| Server cache | TanStack Query | |

---

## Folder Structure (Backend — Authoritative)

Layout mirrors what's declared in the project's architecture document. Agents must place new files in the correct folder:

```
app/
├── controllers/
│   ├── concerns/            # authenticatable, error_handling, paginatable, renderable
│   └── api/v1/              # all public endpoints, thin controllers
├── models/
│   └── concerns/            # tokenizable, sluggable
├── services/                # ALL business logic
│   ├── base_service.rb      # MUST be the superclass of every service
│   ├── service_result.rb    # MUST be the return type
│   ├── auth/                # jwt_service, login_service, register_service
│   ├── agents/              # base_agent, claude_agent, agent_orchestrator
│   ├── tools/               # base_tool, tool_registry, concrete tools
│   ├── ai/                  # claude_client (Singleton), prompt_builder, ...
│   └── conversations/
├── jobs/                    # Sidekiq workers
├── serializers/             # jsonapi-serializer classes
├── policies/                # Pundit policies
├── validators/              # dry-validation contracts
└── channels/                # ActionCable (conversation_channel)
```

**Rules:**
- New business logic → `app/services/<domain>/`. Not controllers, not models, not lib/.
- New tool for Claude → subclass `Tools::BaseTool`, register via `Tools::ToolRegistry`.
- New agent behavior → subclass `Agents::BaseAgent`.
- Cross-cutting model behavior → `app/models/concerns/`.

---

## Mandatory Base Classes & Shared Clients

These are **required inheritance / delegation points**. Bypassing them is a constitutional violation.

| Base / Client | Location | Rule |
|---|---|---|
| `BaseService` | `app/services/base_service.rb` | Every service **MUST** inherit. |
| `ServiceResult` | `app/services/service_result.rb` | Every service **MUST** return this type. |
| `AI::ClaudeClient` | `app/services/ai/claude_client.rb` | Only path to Claude API. Singleton. |
| `Agents::BaseAgent` | `app/services/agents/base_agent.rb` | Superclass for all agents. |
| `Tools::BaseTool` | `app/services/tools/base_tool.rb` | Superclass for all tools. |
| `ApplicationRecord` | `app/models/application_record.rb` | Declares `primary_abstract_class`; all models inherit. |
| `ApplicationJob` | `app/jobs/application_job.rb` | Superclass for Sidekiq jobs. |
| `ApplicationPolicy` | `app/policies/application_policy.rb` | Superclass for Pundit policies. |

**Examples of violations (auto-reject in review):**
- A PORO under `app/services/` that doesn't inherit `BaseService`.
- A job that calls `Anthropic::Client` directly instead of `AI::ClaudeClient`.
- A migration that creates a table without `id: :uuid`.
- A controller that returns `render json: user` instead of going through a serializer + `ServiceResult`.
- Business logic added to a controller action beyond param parsing and result rendering.

---

## Development Workflow

1. **Spec** — Draft `.specify/specs/<feature>/spec.md`. Get it approved.
2. **Plan** — Draft `.specify/specs/<feature>/plan.md`. Cite the principles above for each decision. Identify: new/changed migrations, new services, new jobs, new endpoints, new frontend routes/stores, test targets.
3. **Implement** — Write failing RSpec first for new services (TDD-friendly, not strictly enforced). Implement. Run `bundle exec rspec` + `bundle exec rubocop` + `bundle exec brakeman` locally.
4. **Review** — Reviewer checks: constitution compliance, UUID use, `BaseService`/`ServiceResult` use, `AI::ClaudeClient` use, test coverage, Rubocop clean, no direct external HTTP.
5. **Merge** — Only when all gates pass.

**Agent-specific reminders:**
- `Explore` — when asked "how does X work?", cite actual file paths; do not hallucinate classes.
- `Plan` — must reference this constitution by section number for each architectural choice.
- Implementer agents — must halt if asked to put business logic outside `app/services/` or to bypass `AI::ClaudeClient`.

---

## Governance

- This constitution **supersedes** ad-hoc conventions, inline comments, and individual preferences.
- **Amendments** require: (a) a proposal PR touching only this file, (b) a migration plan for existing code that would become non-compliant, (c) approval before merge.
- **Exceptions** (one-off deviations) must be declared in the feature's `plan.md` under a `## Constitutional Deviations` heading, with justification and scope.
- All PRs and code reviews **must verify compliance** with sections I–VII.
- Complexity or new dependencies must be justified against Principle I (SDD) and Section "Technology Stack (Locked)".

**Version**: 1.0.0 | **Ratified**: 2026-04-13 | **Last Amended**: 2026-04-13
