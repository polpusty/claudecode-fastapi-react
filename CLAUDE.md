# CLAUDE.md

Cloud cost tracker: users connect cloud accounts (currently only Scaleway), a background job pulls billing data, and the frontend shows daily costs per resource kind.

Monorepo with two independent apps — no shared tooling at the root:

- `api/` — FastAPI + SQLModel backend (Python ≥3.10, managed with `uv`)
- `front/` — React 19 + TypeScript + Vite frontend (shadcn/ui, Tailwind v4, react-router v7, recharts)

## Commands

Backend (run from `api/` — modules import each other by top-level name, e.g. `import db`, `from users.rbac import ...`, so the cwd must be `api/`):

```bash
uv sync                          # install deps
uv run fastapi dev main.py       # API on http://localhost:8000 (docs at /docs, schema at /openapi.json)
uv run python -m jobs.beat       # scheduler: turns due ScheduledJobs into Jobs
uv run python -m jobs.worker     # worker: executes pending Jobs
uv run ruff check . && uv run ruff format .
uv run pyrefly check             # type checking
```

Frontend (run from `front/`):

```bash
npm install
npm run dev      # Vite dev server; proxies /api/* -> http://localhost:8000/* (prefix stripped)
npm run build    # tsc -b && vite build
npm run lint     # oxlint
npx shadcn add <component>   # add a shadcn/ui component into src/components/ui
```

There are no tests yet in either app.

## Backend architecture

- `main.py` — creates the app, includes routers, calls `create_db_and_tables()` on startup.
- `db.py` — SQLite engine (`database.db`) and the `get_session` dependency. Routers alias it as `SessionDep = Annotated[Session, Depends(get_session)]`.
- Feature packages, each with `models.py` (+ `router.py` when it exposes endpoints):
  - `users/` — `Org`, `User` (role `admin|member`), JWT auth (`token.py`, `security.py`), `rbac.py` with `get_current_user` / `get_admin_user` dependencies.
  - `cloud_accounts/` — `CloudAccount`; API key stored Fernet-encrypted (`key.py`), only `key_prefix` is ever returned.
  - `scans/` — `Scan`, `Resource`, `CostsDaily`; `POST /scan` creates a daily `ScheduledJob`; `GET /costs/daily/{id}` aggregates costs by day and resource kind. `jobs.py` holds the job handler.
  - `jobs/` — `Job` / `ScheduledJob` tables plus `beat.py` (scheduler) and `worker.py` (dispatches on `job_name` via `match`). A new job type needs a `case` in `worker.py`.
  - `providers/` — per-provider scan logic (`scaleway.py` calls the Scaleway billing API with httpx).

Conventions:

- SQLModel pattern: `Foo(SQLModel, table=True)` is the DB table; `FooCreate` / `FooRead` are plain `SQLModel` schemas for request/response. Always set `response_model=FooRead` (or a return annotation) so secrets (`password`, `key`) never leak.
- Every endpoint that touches org data must scope by `user.org_id` (see `scans/router.py`).
- Auth: `POST /token` (OAuth2 password form, `username` = email) returns `{access_token, token_type}`; protected routes need `Authorization: Bearer <token>`.
- No migrations: tables are created with `create_all`, which never alters existing tables. After changing a table model, delete `api/database.db` (local data is lost) or add the column manually.
- `beat.py` uses SQLite-specific `datetime(...)` SQL (TODO for Postgres).
- `SECRET_KEY` in `users/token.py` and the Fernet key in `cloud_accounts/key.py` are hardcoded dev values — don't copy them elsewhere; move to env vars if touching that code.
- Register new table models in `main.py` (import the module) so `create_all` sees them.

## Frontend architecture

- `src/main.tsx` — routes: `/login` public; everything else under `<Layout>` (sidebar) and `<ProtectedRoute>`.
- `src/AuthContext.tsx` — `AuthProvider` / `useAuth()`; token is kept in React state only (lost on page reload).
- `src/useApi.ts` — `useApi()` returns a typed `fetch` wrapper that adds JSON headers and the Bearer token. **Use it for every authenticated call.** The URL must include the `/api` prefix.
- `src/api.ts` — older plain-`fetch` helpers and API types (`CloudAccount`, `DailyCost`, ...). They do **not** send the token, so they fail against protected endpoints; prefer `useApi` + types from `src/models/` for new code.
- `src/models/` — TypeScript types and API functions mirroring the FastAPI schemas (generated with `/react-models`).
- `src/components/ui/` — shadcn/ui components (style `base-vega`, built on `@base-ui/react`); don't hand-edit unless customizing deliberately.
- Path alias `@/` → `src/`.

Conventions:

- Components use a discriminated-union `Status` state (`{ state: 'loading' } | { state: 'success'; ... } | { state: 'error'; message }`) for async data.
- `verbatimModuleSyntax` is on: import types with `import type`.
- Styling with Tailwind classes and CSS variables from `src/index.css`.

## Keeping front and back in sync

When a FastAPI schema or endpoint changes, regenerate the frontend models with the `/react-models <module>` command (see `.claude/commands/react-models.md`).
