# Backend Constitution — Mizuho Rails 8 Standard

> Stack-specific constitution for the **backend** (`backend/`). Aligned with **Mizuho Portal Backend** company standard.
> Load this together with [constitution.md](constitution.md) (shared § I SDD + Governance) when working on backend code.
> Every Agent that touches `backend/` MUST read this before producing output. Deviations require an explicit amendment (see Governance in root `constitution.md`).

---

## Core Principles

> § I — Spec-Driven Development is **shared** and lives in the root [constitution.md](constitution.md). All backend changes must follow it.

### II. Rails 8 API-Only Backend (NON-NEGOTIABLE)

- **Ruby 3.4.9** (pinned in `.ruby-version`).
- **Rails 8.1.3** API-only mode (`config.api_only = true`). No views, no session middleware, no cookie middleware.
- **PostgreSQL 16** with writer + reader replica routing (`DB_WRITER_ENDPOINT` / `DB_READER_ENDPOINT`). Read queries auto-route to replica via Rails native database routing.
- Authentication: **SSO via Faraday** → returns `access_token` + `refresh_token` (JWT). Encapsulated in `UserAuthenticationService`.
- Authorization: **Pundit**, default-deny. `policy_scope(Model)` for index, `authorize @resource` for others.

### III. ApplicationService Pattern (NON-NEGOTIABLE)

All business logic lives in **Service Objects** under `app/services/`.

- Every service inherits **`ApplicationService`** (at `app/services/application_service.rb`).
- Class method `.call(*args, **kwargs)` is provided by the base.
- Service returns a **plain object/value** — not a wrapped result type, not raw booleans for control flow.
- One service = one domain operation.
- Service does NOT render. The controller renders.
- Services namespaced by domain: `Auth::`, `User::`, `Order::`, `AI::` (when added), etc.

```ruby
# Correct usage
result = CreateOrderService.call(user: current_user, params: order_params)
```

### IV. Thin Controllers (NON-NEGOTIABLE)

- Each controller has at most **7 actions** (CRUD + 2 custom).
- Action body: parse params → `authorize` (Pundit) → call service → render with serializer.
- No business rules in controllers.
- Inheritance chain: `ApplicationController` (`ActionController::API`) → `Api::V1::BaseController` → `Api::V1::<Resource>Controller`.
- `ApplicationController` includes: `ErrorHandler`, `ResponseHandler`, `Pagy::Backend`, `before_action :authenticate_user!`.

### V. Background Jobs (Solid Queue + Sidekiq)

Two job systems, used for different purposes:

- **Solid Queue** (Rails 8 built-in) — default for ActiveJob-based async work. No Redis dependency. Use for lightweight fire-and-forget tasks (notifications, audit logging).
- **Sidekiq 8** + **Redis** — for jobs needing Redis-backed retry, scheduling (`sidekiq-scheduler`), or high-throughput. All Sidekiq workers inherit `BaseWorker` (`retry: 5`, `queue: default`).

```ruby
class SomeWorker < BaseWorker
  def perform(arg)
    # job logic
  end
end
```

Sidekiq Web UI mounted at `/sidekiq` (protected). Scheduled jobs in `config/recurring.yml`.

### VI. External HTTP via Faraday (NON-NEGOTIABLE)

All external HTTP calls (SSO, third-party APIs, AI providers) go through **Faraday + faraday-retry** with exponential backoff.

- Encapsulate every external integration in an `ApplicationService` subclass.
- Reference pattern: `UserAuthenticationService` for SSO.
- Future `AI::ClaudeClient` (when AI feature is specced) follows the same pattern.
- **WebMock** stubs all external HTTP in tests. Never hit real APIs in CI.

### VII. Data Layer Conventions

- All models inherit `ApplicationRecord`.
- **Soft delete** via `discard` gem (NOT `paranoia`, NOT `acts_as_paranoid`).
- **Audit trail** via `paper_trail` — mandatory for fintech/regulated data; tracks every change.
- **Filtering / search** via `ransack` (whitelist-based — explicitly declare `ransackable_attributes`).
- **Pagination** via `pagy` (default 25, max 100). NOT kaminari, NOT will_paginate.
- Shared model behavior in `app/models/concerns/` (e.g., soft delete, audit).
- Primary keys: Rails default (integer / bigint). UUID only when a feature spec explicitly requires it.

### VIII. Serialization & Response Format

- **jsonapi-serializer** — one serializer per model.
- Success response:
  ```json
  {
    "success": true,
    "code": "success",
    "data": { ... },
    "pagination": { "current_page": 1, "total_pages": 5 },
    "total": 100
  }
  ```
- Error response:
  ```json
  {
    "success": false,
    "code": "validation_error",
    "errors": [{ "field": "email", "message": "is invalid" }]
  }
  ```

### IX. Error Handling

- Custom error classes in `lib/errors/` inheriting `ApplicationError`.
- Centralized `rescue_from` in `ApplicationController` via `ErrorHandler` concern.
- Standard mapping:
  - `ActiveRecord::RecordInvalid` → 422 Unprocessable Entity
  - `ActiveRecord::RecordNotFound` → 404 Not Found
  - `Pundit::NotAuthorizedError` → 403 Forbidden
  - `Pagy::OverflowError` → 200 (empty data)
  - `StandardError` → 500 + Bugsnag report (production only)

### X. Testing & Code Quality (NON-NEGOTIABLE)

- **RSpec** (`rspec-rails` 8) for every layer. SimpleCov coverage **≥ 95%**.
- **FactoryBot** for test data; `shoulda-matchers` for one-liner Rails matchers.
- **WebMock** for HTTP stubbing — every external call must be stubbed.
- **rubocop-rails-omakase** (NOT plain `rubocop-rails`):
  - Target Ruby 3.4
  - Max line length 120
  - Method ≤ 20 lines, Class ≤ 200 lines, Block ≤ 25 lines
  - ABC complexity ≤ 30, Cyclomatic ≤ 10
  - ≤ 3 positional params (use keyword args beyond)
- **Brakeman** + **bundler-audit** in CI — zero warnings allowed.
- **Bullet** in dev — flag N+1 queries.
- Every new service requires an RSpec; every new model requires validation specs.

### XI. Internationalization

- Locales: `en.yml` + `ja.yml` + `api.yml` (API error codes & messages).
- All user-facing strings + API error messages must be i18n'd.
- Default locale: English; secondary: Japanese.

### XII. Deployment & Performance

- **Docker** multi-stage (base / build / production) with `ruby:3.4.9-slim`.
- Memory: **jemalloc** via `LD_PRELOAD`.
- **Kamal 2** (`kamal-2.11.0`) for SSH-based Docker deployment. Config in `config/deploy.yml`.
- **Thruster** for HTTP caching/compression in front of Puma.
- **Bootsnap** for boot-time caching.
- Container runs as non-root user (uid 1000).

### XIII. API Versioning

- All endpoints under `/api/v1/...`.
- Routes split into `config/routes/api/v1.rb` (NOT monolithic `routes.rb`).
- Controller namespace: `Api::V1::<Resource>Controller`.
- Future versions add `config/routes/api/v2.rb` + `Api::V2::*` — never break v1.

---

## Technology Stack (Locked)

| Category | Technology | Version |
|---|---|---|
| Language | Ruby | 3.4.9 |
| Framework | Rails | 8.1.3 (API-only) |
| Database | PostgreSQL | 16 |
| Cache | Solid Cache | (Rails 8 built-in) |
| Job Queue | Solid Queue | (Rails 8 built-in) |
| WebSocket | Solid Cable | (Rails 8 built-in) |
| Background Jobs | Sidekiq | 8.1.2 |
| In-memory Store | Redis | 5.4.1 |
| Web Server | Puma | ≥ 5.0 |
| Deployment | Kamal | 2.11.0 |
| Container | Docker (multi-stage + jemalloc) | — |

### Required Gems

| Gem | Version | Purpose |
|---|---|---|
| `pg` | 1.6.3 | PostgreSQL adapter |
| `jsonapi-serializer` | 2.2.0 | Response format |
| `rack-cors` | latest | CORS |
| `rswag-api` / `rswag-ui` / `rswag-specs` | 2.17.0 | Swagger/OpenAPI from RSpec |
| `pundit` | 2.5.2 | Authorization |
| `faraday` / `faraday-retry` | 2.14 / 2.4 | External HTTP |
| `ransack` | 4.4.1 | Filtering (whitelist) |
| `pagy` | 43.4.4 | Pagination |
| `discard` | 1.4.0 | Soft delete |
| `paper_trail` | 17.0.0 | Audit trail |
| `sidekiq` | 8.1.2 | Background jobs |
| `sidekiq-scheduler` | latest | Cron-like jobs |
| `redis` | 5.4.1 | Cache + Sidekiq backend |
| `bugsnag` | 6.29.0 | Error tracking (production) |
| `lograge` | 0.14.0 | JSON structured logging |
| `kamal` | 2.11.0 | Deployment |
| `thruster` | latest | HTTP caching |
| `bootsnap` | latest | Boot caching |
| `image_processing` | ~1.2 | Image processing |

### Dev / Test Gems

| Gem | Group | Purpose |
|---|---|---|
| `rspec-rails` | test | Testing framework |
| `factory_bot_rails` | test | Test data |
| `shoulda-matchers` | test | Rails matchers |
| `simplecov` | test | Coverage (≥ 95%) |
| `webmock` | test | HTTP stubbing |
| `pry-rails` | dev | Console |
| `debug` | dev | Debugger |
| `bullet` | dev | N+1 detection |
| `annotate` | dev | Auto-annotate models |
| `rubocop-rails-omakase` | dev | Lint |
| `brakeman` | dev | Security scan |
| `bundler-audit` | dev | CVE check |

---

## Folder Structure (Authoritative)

Agents must place new files in the correct folder. Layout matches Mizuho Portal Backend exactly.

```
backend/
├── app/
│   ├── controllers/
│   │   ├── application_controller.rb        # Auth, error handling, pagination
│   │   ├── api/v1/
│   │   │   └── base_controller.rb           # API v1 base
│   │   └── concerns/
│   │       ├── error_handler.rb             # rescue_from definitions
│   │       └── response_handler.rb          # render_success / render_error
│   ├── models/
│   │   ├── application_record.rb
│   │   └── concerns/                        # Soft delete, audit, etc.
│   ├── services/
│   │   ├── application_service.rb           # MUST inherit
│   │   └── <domain>/                        # auth/, user/, ai/, ...
│   ├── serializers/                         # 1 per model
│   ├── workers/
│   │   └── base_worker.rb                   # Sidekiq base
│   ├── jobs/
│   │   └── application_job.rb               # Solid Queue base
│   ├── mailers/
│   │   └── application_mailer.rb
│   └── views/layouts/                       # Mailer templates only
│
├── config/
│   ├── application.rb                       # api_only = true
│   ├── routes.rb                            # Root routing, Sidekiq mount
│   ├── routes/api/v1.rb                     # API v1 routes
│   ├── database.yml                         # writer + reader replica
│   ├── puma.rb / cable.yml / cache.yml / queue.yml / recurring.yml
│   ├── deploy.yml                           # Kamal
│   ├── environments/                        # dev / prod / test
│   ├── initializers/                        # bugsnag, cors, redis, sidekiq, ...
│   └── locales/                             # en.yml, ja.yml, api.yml
│
├── lib/
│   ├── errors/
│   │   ├── application_error.rb             # Custom error base
│   │   ├── active_record_validation.rb      # 422
│   │   ├── active_record_not_found.rb       # 404
│   │   └── runtime.rb
│   └── tasks/
│       └── auto_annotate_models.rake
│
├── db/
│   ├── schema.rb
│   ├── seeds.rb
│   ├── cable_schema.rb / cache_schema.rb / queue_schema.rb
│
├── spec/                                    # RSpec (NOT test/)
│   ├── controllers/ / models/ / services/ / workers/ / serializers/
│   ├── factories/
│   └── rails_helper.rb / spec_helper.rb
│
├── docs/                                    # Developer docs
├── .github/workflows/ci.yml                 # Brakeman + bundler-audit + rubocop + rspec
├── bin/                                     # rails, rake, setup, dev, jobs, ci, kamal, thrust, rubocop, brakeman, bundler-audit
├── Dockerfile                               # Multi-stage + jemalloc
├── docker-compose.yml                       # app + db + redis + sidekiq
├── Gemfile / Gemfile.lock
├── .rubocop.yml                             # Inherits rubocop-rails-omakase
└── .ruby-version                            # 3.4.9
```

**Rules:**
- New business logic → `app/services/<domain>/`. Never controllers, never models, never `lib/`.
- New external HTTP integration → service inheriting `ApplicationService` using Faraday.
- New background job → `app/workers/<Name>Worker < BaseWorker` (Sidekiq) OR `app/jobs/<Name>Job < ApplicationJob` (Solid Queue).
- Custom error → `lib/errors/<name>.rb` inheriting `ApplicationError`.
- Cross-cutting model behavior → `app/models/concerns/`.

---

## Mandatory Base Classes

| Class | Location | Rule |
|---|---|---|
| `ApplicationController` | `app/controllers/application_controller.rb` | Inherits `ActionController::API`; includes `ErrorHandler`, `ResponseHandler`, `Pagy::Backend`. Every controller chain starts here. |
| `Api::V1::BaseController` | `app/controllers/api/v1/base_controller.rb` | All `Api::V1::*` controllers inherit. |
| `ApplicationRecord` | `app/models/application_record.rb` | Every model inherits. |
| `ApplicationService` | `app/services/application_service.rb` | **Every service MUST inherit.** Provides `.call(*args, **kwargs)`. |
| `BaseWorker` | `app/workers/base_worker.rb` | Every Sidekiq worker inherits (`retry: 5`, `queue: default`). |
| `ApplicationJob` | `app/jobs/application_job.rb` | Every ActiveJob (Solid Queue) inherits. |
| `ApplicationPolicy` | `app/policies/application_policy.rb` | Every Pundit policy inherits. Default-deny. |
| `ApplicationError` | `lib/errors/application_error.rb` | Every custom error inherits. |

**Auto-reject patterns in review:**

- A PORO under `app/services/` that does not inherit `ApplicationService`.
- Service returning a wrapped result type instead of plain object/value.
- Business logic inside a controller action beyond param parsing + service call + render.
- A controller with > 7 actions.
- A new external HTTP call not going through Faraday + an `ApplicationService` subclass.
- A model without `ransackable_attributes` whitelist when ransack is used.
- A migration introducing soft-delete columns without using `discard`.
- A new gem in `Gemfile.lock` not justified in the feature's `plan.md`.
- An `# rubocop:disable` comment without a one-line justification.

---

## Environment Variables

| Variable | Purpose |
|---|---|
| `RAILS_ENV` | development / production / test |
| `DB_USERNAME`, `DB_PASSWORD` | PostgreSQL credentials |
| `DB_WRITER_ENDPOINT`, `DB_WRITER_NAME` | Primary DB |
| `DB_READER_ENDPOINT`, `DB_READER_NAME` | Read replica |
| `REDIS_URL`, `REDIS_PASSWD` | Redis (cache + Sidekiq) |
| `APP_HOST` | CORS allowed origin |
| `RAILS_MASTER_KEY` | Credentials decryption |
| `RAILS_MAX_THREADS` | Connection pool size (default 5) |
| `BUGSNAG_API_KEY` | Production error tracking |
| `ANTHROPIC_API_KEY` | (when AI feature added) |

---

## Development Workflow

1. **Spec** — `.specify/specs/<feature>/spec.md`. PM (John).
2. **Clarify** (if needed) — BA (Riley).
3. **UX** (if frontend) — UX (Sally).
4. **Plan** — `.specify/specs/<feature>/plan.md` citing this constitution. Architect (Winston).
5. **Implement** — TDD: failing RSpec → minimum code → green → rubocop → brakeman. Dev (Amelia).
6. **Audit** — `qa-report.md`. QA (Murphy). Verify against constitution + manual scenario walk.
7. **PR** — `speckit-git-remote` opens PR. CI runs Brakeman + bundler-audit + rubocop + rspec.

**Local commands:**

```bash
docker-compose up -d                          # start app + db + redis + sidekiq
docker-compose exec app rails db:create db:migrate db:seed
docker-compose exec app rails console
bundle exec rspec                             # tests
bin/rubocop                                   # lint
bin/brakeman -A -w1 ./                        # security
bin/bundler-audit                             # CVE check
```

---

## AI Extension (when added — not yet specced)

When AI features are specced through the SDD process, they MUST follow the conventions above:

- Anthropic API calls → ONE service `AI::ClaudeClient` inheriting `ApplicationService`, using Faraday + faraday-retry per § VI.
- All AI integration code in `app/services/ai/`.
- AI background work via Sidekiq `BaseWorker` per § V (long-running, retry-needed).
- Models like `AgentConfig`, `Conversation`, `Message` follow data layer conventions § VII (paper_trail audit, discard soft delete, ransack filterable).
- WebMock stubs all Anthropic HTTP in tests per § X.
- Token cost & rate limits enforced inside `AI::ClaudeClient` (circuit breaker pattern).

The specific data model + endpoints for AI are NOT pre-prescribed here — they emerge from individual feature specs.

---

## Governance

Governance rules are **shared** — see [constitution.md § Governance](constitution.md). Amendments to this file follow the same process.

**Version**: 1.0.0 (backend) | **Ratified**: 2026-04-14 | **Split from** root constitution v2.0.0
