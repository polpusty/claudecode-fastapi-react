---
description: Add a FastAPI endpoint/resource following project conventions, then generate the React models for it
argument-hint: "<module> <description, e.g. 'budgets: CRUD for monthly budget per cloud account'>"
allowed-tools: Read, Write, Edit, Glob, Grep, Skill, Bash(uv run ruff:*), Bash(uv run pyrefly:*), Bash(uv run python -c:*), Bash(npm run build:*), Bash(npm run lint:*)
---

Add a backend endpoint (or a whole resource) to the FastAPI app in `api/`.

Request: `$ARGUMENTS`

The first word is the target package under `api/` (existing like `scans`, or new like `budgets`); the rest describes what to build. If the description is too vague to decide fields, methods or permissions, ask before writing code.

## 1. Read before writing

- `api/main.py`, `api/db.py`, `api/users/rbac.py`, `api/users/models.py`
- The target package's `models.py` / `router.py` if it exists, and any package whose models you'll reference (foreign keys, joins).
- `api/cloud_accounts/router.py` and `api/scans/router.py` as the reference style.

## 2. Models — `api/<module>/models.py`

For each resource:

```python
class Budget(SQLModel, table=True):          # DB table
    id: uuid.UUID = Field(default_factory=uuid.uuid4, primary_key=True)
    created_at: datetime = Field(default_factory=datetime.utcnow)
    org_id: uuid.UUID = Field(foreign_key="org.id")
    ...

class BudgetCreate(SQLModel):                # request body (POST)
    ...

class BudgetUpdate(SQLModel):                # request body (PATCH) — every field optional, default None
    ...

class BudgetRead(SQLModel):                  # response — only fields the client may see
    id: uuid.UUID
    ...
```

- Only create the schemas the requested endpoints actually need.
- Org-owned data gets `org_id` (directly, or reachable via a FK such as `cloud_account`) so it can be scoped.
- Never put secrets (`password`, `key`, tokens, encrypted values) in `*Read`. Store secrets encrypted (see `cloud_accounts/key.py`) and expose at most a prefix.
- Foreign key strings use the lowercase table name: `"cloudaccount.id"`, `"org.id"`.
- Enums: `class Foo(str, Enum)`.

## 3. Router — `api/<module>/router.py`

```python
from typing import Annotated
from sqlmodel import Session, select
from fastapi import APIRouter, Depends, HTTPException, status
from users.models import User
from users.rbac import get_current_user, get_admin_user
from db import get_session

SessionDep = Annotated[Session, Depends(get_session)]

router = APIRouter()


@router.get("/budgets", response_model=list[BudgetRead])
def list_budgets(session: SessionDep, user: Annotated[User, Depends(get_current_user)]):
    return session.exec(select(Budget).where(Budget.org_id == user.org_id)).all()
```

Rules:

- **Auth**: every endpoint requires `get_current_user`; mutations that configure the org (accounts, settings, budgets) use `get_admin_user`. Ask if unsure. Use `Annotated[User, Depends(...)]` (correctly typed) for new code.
- **Org scoping**: every query filters by `user.org_id`, also on single-object reads/updates/deletes — fetch, then reject if `obj.org_id != user.org_id`. For data owned through a cloud account, verify the cloud account belongs to the org first (pattern in `scans/router.py`).
- **Response**: always set `response_model=` (or a return annotation) with a `*Read` schema. If returning aggregated rows, define a small `*Read` schema for them instead of returning bare dicts.
- **Status codes**: `POST` create → `status_code=status.HTTP_201_CREATED`; `DELETE` → `204` returning nothing; missing or other-org object → `404` (don't reveal it exists); `PATCH` uses `obj.sqlmodel_update(data.model_dump(exclude_unset=True))`.
- **Paths**: follow existing style (`/cloud/account`, `/costs/daily/{id}`), path params typed `uuid.UUID`.
- Keep handlers thin; put non-trivial logic in a helper module in the package (like `cloud_accounts/key.py`, `jobs/job.py`).
- Background work → create a `ScheduledJob`/`Job` and add a `case` in `jobs/worker.py` rather than doing it in the request.

## 4. Register — `api/main.py`

- New package: create `api/<module>/__init__.py` (empty), `import <module>.router` and `app.include_router(<module>.router.router)`.
- New table models must be imported before `create_all` runs (importing the router that imports them is enough; otherwise add `import <module>.models  # noqa: F401`).

## 5. Verify the backend

From `api/`:

```bash
uv run ruff check . && uv run ruff format <changed files>
uv run pyrefly check
uv run python -c "import main, json; s = main.app.openapi(); print(json.dumps({p: list(o) for p, o in s['paths'].items()}, indent=1))"
```

Fix errors in files you touched. Confirm the new paths appear in the OpenAPI output with the expected response schemas.

## 6. Generate React models

Invoke the `react-models` skill with the module name (`/react-models <module>`) so `front/src/models/` gets the new types and API functions.

## 7. Report

- Endpoints added: method, path, auth level, request/response schema.
- Files changed.
- **DB note**: if you added or changed a `table=True` model, say that `api/database.db` must be deleted (or the column added manually) — there are no migrations and `create_all` doesn't alter existing tables. Do not delete it yourself.
- Anything you assumed (permissions, field types) that the user should double-check.
