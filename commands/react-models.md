---
description: Generate TypeScript models and typed API functions in front/src/models from FastAPI schemas and routers
argument-hint: "[module e.g. cloud_accounts | scans | users | all]"
allowed-tools: Read, Write, Edit, Glob, Grep, Bash(npm run build:*), Bash(npm run lint:*), Bash(curl -s http://localhost:8000/openapi.json*)
---

Generate frontend models for the FastAPI backend.

Target: `$ARGUMENTS` (a package under `api/`, e.g. `cloud_accounts`). If empty or `all`, process every package under `api/` that has a `router.py`.

## 1. Read the backend

For each target package read `api/<module>/models.py` and `api/<module>/router.py`, plus any models they import from other packages (e.g. `scans/router.py` uses `cloud_accounts.models`).

If the API is running, `curl -s http://localhost:8000/openapi.json` is the most precise source for request/response shapes — use it to verify. Otherwise work from the source.

Collect:

- **Schemas** that cross the HTTP boundary: request bodies (`*Create`, `*Update`), `response_model=` classes, return annotations, path/query params, and enums used by them.
- **Endpoints**: method, path, params, body type, response type, and whether they require auth (`get_current_user` / `get_admin_user` / `oauth2_schema` dependency).

Do NOT generate types for `table=True` models unless one is returned directly without a `response_model` — and never include secret fields (`password`, `key`, hashed or encrypted values). If an endpoint returns ad-hoc dicts (e.g. `GET /costs/daily/{id}`), derive the type from the dict literal and name it after the resource (`DailyCost`).

## 2. Map Python types to TypeScript

| Python / Pydantic | TypeScript |
|---|---|
| `str`, `EmailStr`, `uuid.UUID` | `string` |
| `datetime`, `date` | `string` (ISO 8601; add a `/** ISO date */` comment) |
| `int`, `float` | `number` |
| `bool` | `boolean` |
| `bytes` | skip (never exposed) |
| `dict` / `dict[str, X]` | `Record<string, unknown>` / `Record<string, X>` |
| `list[X]` | `X[]` |
| `X \| None`, `Optional[X]` | `X \| null` |
| `class Foo(str, Enum)` | `export type Foo = 'a' \| 'b'` |
| field with a default in a request model | optional property `field?: T` |

Keep snake_case field names exactly as the API sends them. Use `export type` (not `interface`), matching `src/api.ts`.

## 3. Write `front/src/models/<module>.ts`

One file per backend package (`cloud_accounts` → `front/src/models/cloudAccounts.ts`). Structure:

```ts
import type { ApiCall } from '../useApi'

export type CloudAccount = {
  id: string
  key_prefix: string
}

export type CloudAccountCreate = {
  key: string
  connection_params: Record<string, unknown>
}

export function listCloudAccounts(api: ApiCall): Promise<CloudAccount[]> {
  return api<CloudAccount[]>('/api/cloud/account')
}

export function createCloudAccount(api: ApiCall, payload: CloudAccountCreate): Promise<CloudAccount> {
  return api<CloudAccount>('/api/cloud/account', {
    method: 'POST',
    body: JSON.stringify(payload),
  })
}
```

Rules:

- Type names mirror the Python class names (`CloudAccountRead` → `CloudAccount`; keep `Create`/`Update` suffixes).
- One function per endpoint, named `<verb><Resource>` (`list`, `get`, `create`, `update`, `delete`). First param is always `api: ApiCall` (from `useApi()`), then path params, then body. URLs always start with `/api` (Vite proxy).
- Endpoints that return nothing → `Promise<void>`, call `api<void>(...)`.
- Form-encoded endpoints (`POST /token` with `OAuth2PasswordRequestForm`) are handled by `AuthContext.tsx` — skip them.
- If `ApiCall` is not exported from `front/src/useApi.ts`, export it there (`export type ApiCall = ...`) and fix the `oprions` typo while at it. Don't change its runtime behavior.
- Add `front/src/models/index.ts` re-exporting each module (`export * from './cloudAccounts'`) and keep it updated.
- If the file already exists, update it in place: keep hand-written additions, replace generated types/functions that changed, remove ones whose endpoint no longer exists.

Do not modify `front/src/api.ts` or existing components — that migration is a separate step. Mention in the summary which types in `src/api.ts` are now duplicated.

## 4. Verify

Run `npm run build` and `npm run lint` in `front/` and fix any errors in the generated files.

## 5. Report

Briefly list: files created/updated, types generated, endpoints covered, and anything skipped or uncertain (e.g. endpoints without a response model where the type was inferred, or backend issues spotted such as a missing `response_model` that could leak fields).
