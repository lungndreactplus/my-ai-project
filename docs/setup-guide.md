# AI Team Agent — Setup Guide

> Hướng dẫn dựng lại toàn bộ dự án này từ con số 0 cho người chưa quen với AI agent. Đi tuần tự từ trên xuống dưới, không bỏ bước.

**Project name:** `my-ai-project` — một AI team agent cho dự án Rails 7.1 + Vue/React, gồm 5 layer: Spec-kit (L0) + Personas (L2) + Evals (L3) + Observability (L4 — deferred) + Ops/CI (L5).

**Thời gian dự kiến:** ~3 giờ cho người mới.

---

## Mục lục

- [Bước 0 — Prerequisites](#bước-0--prerequisites)
- [Bước 1 — Init repo + cấu trúc monorepo](#bước-1--init-repo--cấu-trúc-monorepo)
- [Bước 2 — Layer 0: Spec-kit](#bước-2--layer-0-spec-kit)
- [Bước 3 — Constitution + CLAUDE.md](#bước-3--constitution--claudemd)
- [Bước 4 — Layer 2: Personas (BMAD-ported)](#bước-4--layer-2-personas-bmad-ported)
- [Bước 5 — Layer 3: Evals (promptfoo)](#bước-5--layer-3-evals-promptfoo)
- [Bước 6 — Layer 5: MCP + GitHub Action](#bước-6--layer-5-mcp--github-action)
- [Bước 7 — Layer 4 & 1 (Deferred)](#bước-7--layer-4--1-deferred)
- [Checklist tổng](#checklist-tổng)
- [Troubleshooting](#troubleshooting)

---

## Bước 0 — Prerequisites

### 0.1 Cài công cụ hệ thống

| Tool | Version tối thiểu | Mục đích | Cách cài (Windows) | Verify |
|---|---|---|---|---|
| **Node.js** | 20 LTS | promptfoo, MCP servers (npx) | https://nodejs.org → installer | `node -v` |
| **npm** | đi kèm Node | quản lý package | (đi kèm Node) | `npm -v` |
| **Git** | 2.40+ | version control | https://git-scm.com | `git --version` |
| **Python** | 3.11+ | chạy `uv` để cài Spec-kit | https://python.org | `python --version` |
| **uv** | 0.4+ | Python tool installer | `pip install uv` | `uv --version` |
| **Claude Code CLI** | latest | agent runner | `npm install -g @anthropic-ai/claude-code` | `claude --version` |

> **Mac/Linux:** dùng `brew install node git python uv` rồi `npm install -g @anthropic-ai/claude-code`.

Sau này (chỉ cần khi bắt đầu code backend/frontend thật):
- **Ruby 3.3.0** + **Bundler** — cho backend Rails
- **PostgreSQL 14+** — database
- **Redis 7+** — Sidekiq queue

### 0.2 Đăng ký + lấy 3 API key

| Key | Nơi lấy | Dùng cho | Format |
|---|---|---|---|
| `ANTHROPIC_API_KEY` | https://console.anthropic.com → Settings → API Keys → "Create Key" | promptfoo eval, Claude Code | `sk-ant-...` |
| `GITHUB_TOKEN` | https://github.com/settings/tokens → "Generate new token (classic)" → scope `repo` + `workflow` | GitHub MCP server | `ghp_...` |
| `CLAUDE_CODE_OAUTH_TOKEN` | Terminal: `claude setup-token` → token hiển thị | GitHub Action review trên PR | dài, ngẫu nhiên |

### 0.3 Set environment variables

**Windows PowerShell (persist vào User scope):**
```powershell
[Environment]::SetEnvironmentVariable('ANTHROPIC_API_KEY','sk-ant-...','User')
[Environment]::SetEnvironmentVariable('GITHUB_TOKEN','ghp_...','User')
```
Đóng terminal, mở lại. Verify:
```powershell
echo $env:ANTHROPIC_API_KEY
```

**Mac/Linux (vào `~/.zshrc` hoặc `~/.bashrc`):**
```bash
export ANTHROPIC_API_KEY=sk-ant-...
export GITHUB_TOKEN=ghp_...
```
`source ~/.zshrc` rồi `echo $ANTHROPIC_API_KEY`.

> **CẢNH BÁO:** Không commit env vars vào git. Không paste key vào file nào khác ngoài terminal.

---

## Bước 1 — Init repo + cấu trúc monorepo

### 1.1 Tạo repo

```bash
mkdir my-ai-project
cd my-ai-project
git init
mkdir backend frontend
```

### 1.2 Tạo `.gitignore` ở root

Tạo file `my-ai-project/.gitignore`:

```gitignore
# Dependencies
node_modules/
vendor/bundle/

# Environment
.env
.env.local
.env.*.local

# IDE
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db

# Logs
*.log
log/

# Build artifacts
dist/
build/
tmp/

# Eval cache
evals/.promptfoo/
evals/output/
evals/node_modules/

# Backend (Rails)
backend/tmp/
backend/log/
backend/storage/
backend/.bundle/
backend/node_modules/

# Frontend
frontend/node_modules/
frontend/dist/
frontend/.vite/
```

### 1.3 First commit

```bash
git add .gitignore
git commit -m "chore: initialize monorepo skeleton"
```

---

## Bước 2 — Layer 0: Spec-kit

Spec-kit là CLI tạo cấu trúc Spec-Driven Development (`.specify/` + 14 slash-command skill).

### 2.1 Cài Spec-kit CLI

```bash
uv tool install specify-cli --from git+https://github.com/github/spec-kit.git
```

Verify:
```bash
specify --version
```

> Nếu lệnh `specify` không nhận ra, restart terminal hoặc thêm `~/.local/bin` (Mac/Linux) hoặc `%USERPROFILE%\.local\bin` (Windows) vào PATH.

### 2.2 Init Spec-kit vào project

```bash
cd my-ai-project
specify init --here --ai claude
```

Lệnh này tạo:

```
.specify/
├── memory/
│   └── constitution.md         ← TEMPLATE rỗng (bạn tự điền)
├── templates/
│   ├── spec-template.md
│   ├── plan-template.md
│   ├── tasks-template.md
│   └── checklist-template.md
└── ...
.claude/
└── skills/
    ├── speckit-specify/
    ├── speckit-clarify/
    ├── speckit-plan/
    ├── speckit-tasks/
    ├── speckit-checklist/
    ├── speckit-implement/
    ├── speckit-analyze/
    ├── speckit-constitution/
    ├── speckit-git-initialize/
    ├── speckit-git-feature/
    ├── speckit-git-validate/
    ├── speckit-git-commit/
    ├── speckit-git-remote/
    └── speckit-taskstoissues/
```

Verify:
```bash
ls .specify/memory/
ls .claude/skills/
```

---

## Bước 3 — Constitution + CLAUDE.md

### 3.1 Viết `.specify/memory/constitution.md`

Đây là **luật bất di bất dịch** của dự án. Spec-kit chỉ tạo template — bạn phải tự viết nội dung.

Mở `.specify/memory/constitution.md`, copy mẫu từ repo này (~ 200 dòng), gồm 7 nguyên tắc:

- **§ I** — Spec-Driven Development (NON-NEGOTIABLE)
- **§ II** — Rails 7.1 API-only / Ruby 3.3.0
- **§ III** — UUID Primary Keys (NON-NEGOTIABLE)
- **§ IV** — Service Objects → BaseService → ServiceResult (NON-NEGOTIABLE)
- **§ V** — Single Claude Integration Point — `AI::ClaudeClient` (NON-NEGOTIABLE)
- **§ VI** — Frontend: Tailwind + Centralized State + apiClient
- **§ VII** — Testing & Code Quality (NON-NEGOTIABLE)

Cộng các section: Technology Stack (Locked), Folder Structure (Authoritative), Mandatory Base Classes, Development Workflow, Governance.

> Nội dung cụ thể: copy nguyên file `.specify/memory/constitution.md` từ repo `my-ai-project` này.

### 3.2 Viết `CLAUDE.md` ở root

CLAUDE.md là **handbook vận hành** — Claude Code đọc file này tự động. Phải có:

1. **Project Context** — sơ đồ monorepo.
2. **Build & Install** — lệnh cho backend (`bundle install`), frontend (`npm install`), evals.
3. **Test & Quality** — `bundle exec rspec`, `bundle exec rubocop`, `npm test`, `npm run eval`.
4. **Persona Subagents** — bảng 6 persona + chuỗi handoff.
5. **Manager Orchestration** — bảng shortcut routing (`#pm`, `#architect`...).
6. **Constitutional Rules** — tóm tắt 7 nguyên tắc + auto-reject patterns.

> Copy nguyên file `CLAUDE.md` từ repo `my-ai-project` này (~ 250 dòng).

### 3.3 Commit

```bash
git add .specify/ .claude/skills/ CLAUDE.md
git commit -m "chore(spec-kit): init constitution + manager handbook"
```

---

## Bước 4 — Layer 2: Personas (BMAD-ported)

6 persona đại diện 1 team Agile. 4 cái port từ BMAD-METHOD (PM/UX/Architect/Dev), 2 cái custom (BA/QA).

### 4.1 Tạo thư mục

```bash
mkdir -p .claude/agents
```

### 4.2 (Optional) Clone BMAD để tham khảo prompt gốc

```bash
cd ..    # ra ngoài my-ai-project
git clone --depth 1 https://github.com/bmad-code-org/BMAD-METHOD.git
cd my-ai-project
```

Persona BMAD gốc nằm tại:
- PM (John): `BMAD-METHOD/src/bmm-skills/2-plan-workflows/bmad-agent-pm/SKILL.md`
- UX (Sally): `BMAD-METHOD/src/bmm-skills/2-plan-workflows/bmad-agent-ux-designer/SKILL.md`
- Architect (Winston): `BMAD-METHOD/src/bmm-skills/3-solutioning/bmad-agent-architect/SKILL.md`
- Dev (Amelia): `BMAD-METHOD/src/bmm-skills/4-implementation/bmad-agent-dev/SKILL.md`

### 4.3 Tạo 6 file persona

Mỗi file format:

```markdown
---
name: <persona-name>
description: <khi nào invoke>
tools: <tool list>
---

# <Character> — <Role>

## Overview
## Identity
## Communication Style
## Principles
## Critical Actions
## Capabilities (table mapping codes → speckit-* skills)
## On Activation (steps 1–4, ending with STOP and WAIT)
## Handoff
## Reference
```

Tạo 6 file (copy nội dung từ repo này):

| File | Character | Vai trò | BMAD source |
|---|---|---|---|
| `.claude/agents/pm.md` | John | Product Manager | `bmad-agent-pm` |
| `.claude/agents/ba.md` | Riley | Business Analyst | (custom — không có trong BMAD) |
| `.claude/agents/ux.md` | Sally | UX Designer | `bmad-agent-ux-designer` |
| `.claude/agents/architect.md` | Winston | System Architect | `bmad-agent-architect` |
| `.claude/agents/dev.md` | Amelia | Senior SWE | `bmad-agent-dev` |
| `.claude/agents/qa.md` | Murphy | Quality Assurance | (custom — BMAD phân tán quality vào nhiều skill) |

### 4.4 Verify

```bash
claude /agents
```

Phải thấy 6 subagent custom (pm, ba, ux, architect, dev, qa) trong danh sách. Test:

```
> #pm tạo spec cho "user đổi tên agent của họ"
```

→ Manager phải route sang persona PM, persona giới thiệu là John, hỏi "WHY?"...

### 4.5 Commit

```bash
git add .claude/agents/
git commit -m "feat(agents): port BMAD personas (John/Winston/Amelia/Sally) + custom (Riley/Murphy)"
```

---

## Bước 5 — Layer 3: Evals (promptfoo)

Eval gác prompt persona khỏi xuống cấp âm thầm. Có 2 cách setup — chọn 1.

### 5A — Cách official (zero-install, cho người mới)

```bash
cd my-ai-project
npx promptfoo@latest init --example getting-started evals
cd evals
export ANTHROPIC_API_KEY=sk-ant-...     # nếu chưa set global
npx promptfoo@latest eval
npx promptfoo@latest view
```

Promptfoo tự scaffold `evals/promptfooconfig.yaml` + ví dụ. Sau đó bạn customize để dùng `anthropic:messages:claude-sonnet-4-20250514` thay cho OpenAI default.

### 5B — Cách CI-friendly (devDependency, version pinned)

```bash
mkdir -p evals/prompts evals/datasets
cd evals
npm init -y
npm install --save-dev promptfoo
```

Sửa `evals/package.json` thêm scripts:
```json
{
  "scripts": {
    "eval": "promptfoo eval",
    "view": "promptfoo view",
    "baseline": "promptfoo eval --output baseline.json"
  }
}
```

Tạo 4 file (copy từ repo `my-ai-project`):
- `evals/promptfooconfig.yaml`
- `evals/prompts/pm_write_spec.txt`
- `evals/prompts/architect_write_plan.txt`
- `evals/datasets/golden.yaml`

Tạo `evals/.gitignore`:
```
node_modules/
.promptfoo/
output/
*.cache
```

### 5.3 Chạy baseline lần đầu

```bash
cd evals
npm run baseline       # tạo baseline.json
git add baseline.json
git commit -m "chore(evals): commit initial baseline"
```

### 5.4 Workflow regression

Mỗi lần sửa prompt persona (`.claude/agents/*.md`):

```bash
cd evals
npm run eval           # so điểm với baseline
npm run view           # mở HTML report ở localhost
```

Nếu drop > 10% trên prompt nào → QA (Murphy) sẽ block merge. Update `baseline.json` chỉ khi đã review thủ công và xác nhận thay đổi là cải thiện.

---

## Bước 6 — Layer 5: MCP + GitHub Action

### 6.1 `.mcp.json` ở root

Tạo `my-ai-project/.mcp.json`:

```json
{
  "mcpServers": {
    "github": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_TOKEN}"
      }
    }
  }
}
```

Restart Claude Code. Verify:

```
> /mcp
```

→ Phải thấy `github` server "connected".

> Thêm Linear/Jira MCP sau bằng cách thêm entry vào `mcpServers`. Danh sách official: https://github.com/modelcontextprotocol/servers

### 6.2 GitHub Action review

```bash
mkdir -p .github/workflows
```

Tạo `.github/workflows/claude-code-review.yml` (copy từ repo `my-ai-project`). Workflow này chạy `anthropics/claude-code-action@v1` trên mọi PR, review diff theo constitution.

### 6.3 Push lên GitHub

```bash
# Tạo repo trống trên github.com (KHÔNG init README)
git remote add origin https://github.com/<username>/my-ai-project.git
git add .
git commit -m "feat: complete L0+L2+L3+L5 setup"
git branch -M main
git push -u origin main
```

### 6.4 Cài secret cho GitHub Action

1. Vào repo trên github.com → **Settings → Secrets and variables → Actions → New repository secret**
2. Add secret:
   - Name: `CLAUDE_CODE_OAUTH_TOKEN`
   - Value: token từ lệnh `claude setup-token` (Bước 0.2)

### 6.5 (Optional) Branch protection

Settings → Branches → Add rule cho `main`:
- Require status checks to pass before merging
- Tích "Claude Constitution Review"
- Require pull request review

### 6.6 Test workflow

```bash
git checkout -b test-pr
echo "test" >> README.md
git add README.md && git commit -m "test PR"
git push -u origin test-pr
gh pr create --title "Test workflow" --body "Testing Claude review"
```

Đợi 1–2 phút, vào PR trên GitHub xem comment từ "Claude" — phải có review chi tiết theo constitution.

---

## Bước 7 — Layer 4 & 1 (Deferred)

### Layer 4 — Observability (Langfuse)

**Khi nào làm:** sau khi `backend/app/services/ai/claude_client.rb` tồn tại (tức là bạn đã scaffold Rails backend và viết ClaudeClient).

**Cài Langfuse self-hosted:**
```bash
git clone https://github.com/langfuse/langfuse.git
cd langfuse
docker compose up -d
# Mở http://localhost:3000 → đăng ký + tạo project → lấy LANGFUSE_PUBLIC_KEY + LANGFUSE_SECRET_KEY
```

Hoặc dùng cloud: https://cloud.langfuse.com (free tier).

**Wrap `AI::ClaudeClient`:**
```ruby
# Gemfile
gem "langfuse-ruby"   # hoặc gọi HTTP trực tiếp
```

Wrap method `chat(...)` để emit span chứa input/output/token usage/cost.

### Layer 1 — Execution Engine

**Khi nào làm:** khi ≥ 3 feature chạy song song hoặc team ≥ 2 người.

**Worktree isolation:** sửa Manager prompt trong CLAUDE.md — khi gọi `Agent(subagent_type: "dev")`, set `isolation: "worktree"`. Mỗi feature implement trong worktree riêng.

**TDD hook:** tạo `.claude/hooks/pre-write.sh` check tồn tại spec.md tương ứng trước khi cho phép Write code:
```json
// .claude/settings.json
{
  "hooks": {
    "PreToolUse": [
      { "matcher": "Write", "command": "./scripts/check-spec-exists.sh" }
    ]
  }
}
```

---

## Checklist tổng

Đánh dấu khi hoàn thành:

### Bước 0 — Prerequisites
- [ ] Node.js 20+ cài
- [ ] Git cài
- [ ] Python 3.11+ cài
- [ ] uv cài (`pip install uv`)
- [ ] Claude Code CLI cài (`npm install -g @anthropic-ai/claude-code`)
- [ ] `ANTHROPIC_API_KEY` set
- [ ] `GITHUB_TOKEN` set
- [ ] `CLAUDE_CODE_OAUTH_TOKEN` lấy được từ `claude setup-token`

### Bước 1 — Repo
- [ ] `mkdir my-ai-project && git init` xong
- [ ] `backend/` + `frontend/` tạo
- [ ] `.gitignore` root viết

### Bước 2 — Spec-kit
- [ ] `specify init --here --ai claude` chạy thành công
- [ ] `.specify/memory/constitution.md` template tồn tại
- [ ] `.claude/skills/speckit-*` (14 skill) tồn tại

### Bước 3 — Constitution + Manager
- [ ] `.specify/memory/constitution.md` điền 7 nguyên tắc
- [ ] `CLAUDE.md` viết ở root

### Bước 4 — Personas
- [ ] `.claude/agents/` tạo
- [ ] 6 file persona (pm/ba/ux/architect/dev/qa) tồn tại
- [ ] `claude /agents` thấy 6 subagent custom

### Bước 5 — Evals
- [ ] `evals/` tồn tại với promptfoo cài
- [ ] `promptfooconfig.yaml` + prompts/ + datasets/golden.yaml viết
- [ ] `npm run baseline` chạy thành công, `baseline.json` commit

### Bước 6 — Ops/CI
- [ ] `.mcp.json` viết, Claude Code thấy GitHub MCP "connected"
- [ ] `.github/workflows/claude-code-review.yml` viết
- [ ] Push GitHub xong
- [ ] Secret `CLAUDE_CODE_OAUTH_TOKEN` add vào repo
- [ ] PR test thấy comment review từ Claude

### Bước 7 — Deferred
- [ ] L4 Langfuse — sau khi `backend/` scaffold
- [ ] L1 TDD hook + worktree — khi cần

---

## Troubleshooting

### `specify: command not found`
PATH chưa có `~/.local/bin`. Trên Windows:
```powershell
[Environment]::SetEnvironmentVariable('PATH', "$env:PATH;$env:USERPROFILE\.local\bin", 'User')
```
Restart terminal.

### `claude /agents` không thấy persona custom
- Kiểm tra file ở đúng `.claude/agents/*.md` (không phải `.claude/skills/`).
- Frontmatter YAML phải có `name`, `description`. `tools` optional.
- Restart Claude Code.

### `npm run eval` báo "ANTHROPIC_API_KEY not set"
- Restart terminal sau khi set env var.
- Hoặc `export ANTHROPIC_API_KEY=sk-ant-...` ngay trước khi chạy.

### GitHub Action fail với "Bad credentials"
- Secret name phải đúng `CLAUDE_CODE_OAUTH_TOKEN` (case-sensitive).
- Token có thể đã hết hạn — chạy lại `claude setup-token`.

### MCP `github` server không connect
- Verify `GITHUB_TOKEN` PAT có scope `repo` và `workflow`.
- Verify `npx -y @modelcontextprotocol/server-github` chạy được tay (không lỗi).

### Persona không "in character"
- Đọc lại Critical Actions section của file persona.
- Pattern `you must fully embody this persona` phải có trong prompt.
- Nếu Manager bypass, kiểm tra routing trong CLAUDE.md.

---

## Tài liệu tham khảo

- **Spec-kit:** https://github.com/github/spec-kit
- **BMAD-METHOD:** https://github.com/bmad-code-org/BMAD-METHOD — docs: https://docs.bmad-method.org
- **promptfoo:** https://github.com/promptfoo/promptfoo
- **Claude Code:** https://docs.claude.com/en/docs/claude-code
- **Anthropic API console:** https://console.anthropic.com
- **MCP servers:** https://github.com/modelcontextprotocol/servers
- **claude-code-action:** https://github.com/anthropics/claude-code-action
- **Langfuse:** https://langfuse.com (cloud) / https://github.com/langfuse/langfuse (self-host)

---

**Phiên bản hướng dẫn:** 1.0 — 2026-04-13
**Áp dụng cho:** my-ai-project layer model L0+L2+L3+L5
