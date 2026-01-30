# AGENTS.md

This repository is currently **docs-first** (no source code / build manifests checked in yet).
The conventions below come from the project documents and are intended to be the **source of
truth until real tooling/config files exist**.

Referenced docs:
- `ChatBI工程规范v0.1.md`
- `ChatBI架构设计v0.1.md`
- `ChatBI数据产品需求规格说明书.md`

Cursor/Copilot rules:
- Searched for `.cursorrules`, `.cursor/rules/`, `.github/copilot-instructions.md` and found none.

--------------------------------------------------------------------------------

## Repo Layout (Target)

The architecture doc proposes this layout (agents should follow it when adding code):

- `apps/web/`      React + TypeScript frontend (Vite recommended)
- `apps/api/`      FastAPI backend (Python 3.10)
- `packages/shared/`  Shared types/protocols (optional)
- `agent/`         LangGraph graphs, tools, prompts, eval
- `infra/`         Docker/dev/test infra
- `.github/workflows/` CI

If the actual repo structure diverges later, follow the repo, not this file.

--------------------------------------------------------------------------------

## Commands

### Current State

No repo-enforced commands exist yet (no `package.json`, `pyproject.toml`, lockfiles, Make/Just,
or CI workflows are present). The commands below reflect the **intended toolchain**.

When adding manifests, prefer wiring these as `make` targets or package scripts so agents can
run the exact repo-approved commands.

### Backend (FastAPI / Python 3.10)

Assumes a future `apps/api/` Python project.

- Install (example): `python -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt`
- Run dev (example): `uvicorn app.main:app --reload`

Lint / format / types (per engineering spec):
- Lint: `ruff check .`
- Format: `black .`
- Sort imports: `isort .`
- Typecheck: `mypy .`

Tests:
- Run all tests: `pytest`
- Run a single test file: `pytest path/to/test_file.py`
- Run a single test function: `pytest path/to/test_file.py::test_name`
- Run a single test method: `pytest path/to/test_file.py::TestClass::test_name`
- Run by name pattern: `pytest -k "substring"`

### Frontend (React + TypeScript)

Assumes a future `apps/web/` project using Vite + React Router + TanStack Query.

Lint / format (per engineering spec):
- Lint (ESLint): `eslint .`
- Format (Prettier): `prettier --write .`

Unit tests (per engineering spec: Vitest + React Testing Library):
- Run all: `vitest run`
- Watch: `vitest`
- Run a single file: `vitest run path/to/file.test.ts`
- Run a single test by name: `vitest -t "test name"`

### E2E (Playwright)

Assumes Playwright is wired under `apps/web/`.

- Run all e2e: `playwright test`
- Run a single spec file: `playwright test path/to/spec.ts`
- Run a single test by name: `playwright test -g "test name"`

--------------------------------------------------------------------------------

## Code Style & Engineering Guidelines

### General

- Readability first: small functions, clear names, avoid magic numbers/params.
- No cross-layer shortcuts:
  - UI must not generate/execute SQL directly.
  - Agent code must not connect to DB directly; go through the Query Service.
- All external dependencies must be replaceable:
  - LLM provider, vector DB, and database connections must be behind interfaces/adapters.

### Frontend (React + TypeScript)

- TypeScript:
  - Keep `strict: true` and `noImplicitAny`.
  - Avoid `any`; prefer explicit types and narrow unions.
- State:
  - Server state: TanStack Query.
  - Local UI state: Zustand or React Context; avoid Redux unless complexity forces it.
- Folder conventions (target):
  - `components/`: presentational only (no data fetching)
  - `features/`: grouped by domain (e.g. chat/template/admin/audit)
  - `pages/`: route-level pages
- Error UX:
  - Error messages must be actionable (tell user what to do next), not just "failed".
  - Prefer a single place for request error translation (HTTP -> UI copy).

### Backend (FastAPI)

- Layering (target): `routers/` (HTTP) -> `services/` (business) -> `repositories/` (data access)
- DTO vs ORM:
  - `schemas/` for Pydantic models (request/response)
  - `models/` for SQLAlchemy models
- Error handling:
  - Use a unified `ErrorCode` and include a `trace_id` in responses.
  - Messages should include suggested remediation when possible.
  - No empty `except` blocks; never swallow exceptions silently.

### Agent (LangGraph)

- Graph nodes (Node):
  - Typed input/output (TypedDict or Pydantic).
  - Single responsibility (e.g. clarify, retrieve_kb, generate_sql, guardrail, execute, summarize).
- Tools:
  - Strict parameter validation and explicit permissions (especially SQL).
  - Observability: record duration, hit rate, error types.
- Prompts:
  - Separate system policy vs business hints.
  - Version prompts with an id + change notes; keep rollback paths.

--------------------------------------------------------------------------------

## Agent Operating Rules (Repo Hygiene)

- Do not add new dependencies unless necessary; prefer existing stack.
- Do not introduce cross-layer coupling to “ship faster”.
- Do not write configs that contradict the engineering spec without updating docs/ADR.
- Do not commit secrets; keep `.env.example` and document required env vars.
