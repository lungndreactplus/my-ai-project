# Project Constitution — Root

> **Source of truth index** for all technical decisions. This project has **two stack-specific constitutions** that agents must load based on scope.
> Every Agent (Manager, PM, BA, UX, Architect, Dev, QA) MUST read this file + the relevant stack file before producing any output.

---

## Which file to read when

| Scope of work | Files to load |
|---|---|
| Anything involving backend code (`backend/**`) | `constitution.md` + [constitution-backend.md](constitution-backend.md) |
| Anything involving mobile app code (`frontend/**`) | `constitution.md` + [constitution-frontend.md](constitution-frontend.md) |
| Cross-cutting feature spanning both (new API endpoint + mobile screen consuming it) | All three files |
| Spec writing (PM phase) — user-facing language only | `constitution.md` only (§ I) |
| Clarification (BA) | `constitution.md` only (§ I) |

**Rule:** if unsure, load all three.

---

## Core Shared Principle

### I. Spec-Driven Development (NON-NEGOTIABLE)

Every change flows through three mandatory gates — in this order:

1. **Spec** — `.specify/specs/<feature>/spec.md` describing *what* and *why* in user-facing language.
2. **Plan** — `.specify/specs/<feature>/plan.md` describing *how*, citing constitution sections from the relevant stack file(s).
3. **Implement** — code changes executing the approved Plan.

**Rules:**

- No direct code edits without an approved Spec + Plan — including "small fixes" that touch business logic.
- Agents that receive a coding task without a Spec must **halt and request one**, not improvise.
- Bug fixes with a trivial root cause (typo, missing nil-check, TypeScript missing type) may skip Spec but still require a one-paragraph Plan.
- A cross-cutting feature produces **one** `plan.md` that cites sections from **both** `constitution-backend.md` and `constitution-frontend.md`.

---

## Stack-Specific Constitutions

### Backend — [constitution-backend.md](constitution-backend.md)

**Stack:** Ruby on Rails 8.1.3 / Ruby 3.4.9 / PostgreSQL 16 / Sidekiq / Solid Queue.
**Sections (§ II–XIII):** API-only mode, ApplicationService pattern, thin controllers, background jobs, Faraday for external HTTP, discard + paper_trail + ransack + pagy, jsonapi-serializer, custom errors, RSpec + SimpleCov ≥ 95% + rubocop-rails-omakase, i18n, Docker + Kamal + Thruster + jemalloc, API versioning.

### Frontend — [constitution-frontend.md](constitution-frontend.md)

**Stack:** React Native 0.83.x / React 19.x / TypeScript 5.8+ / Hermes / yarn.
**Sections (§ II–XX):** TypeScript strict, React Navigation 7, Zustand, React Hook Form + yup, i18next, axios instance, StyleSheet.create, FlashList, folder structure (two component roots + feature-first pages), function components only, Jest, ESLint + Prettier, imports, images + safe area + splash, Hermes/Fabric, mobile security, Android/iOS build variants, Conventional Commits, naming.

---

## Governance (shared)

- The three constitution files together **supersede** ad-hoc conventions, inline comments, and individual preferences.
- **Amendments** require: (a) a proposal PR touching only the affected constitution file, (b) a migration plan for existing code that would become non-compliant, (c) approval before merge.
- **Exceptions** (one-off deviations) must be declared in the feature's `plan.md` under a `## Constitutional Deviations` heading, with justification + scope + compensating controls.
- All PRs and code reviews must verify compliance with the relevant stack file(s).
- Complexity or new dependencies must be justified against the relevant constitution and the feature's plan.
- **Version bumps:**
  - Root (`constitution.md`) — bump when shared principle changes (SDD, governance).
  - `constitution-backend.md` — bump independently.
  - `constitution-frontend.md` — bump independently.

---

**Root version**: 3.0.0 | **Ratified**: 2026-04-14 | **Change**: split stack-specific rules into `constitution-backend.md` and `constitution-frontend.md`; this file now holds shared § I (SDD) + Governance + index.
