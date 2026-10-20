# Conventions — the rules every v3 REST call obeys

Read this once; it applies to all endpoints in `endpoint-catalog.md`.

**Contents:** Base URL & versioning · Authentication · Response envelope · Pagination · Writes that
merge vs replace · The async / poll model · Errors · Rate limits · Company filters

## Base URL & versioning — always use v3

```
https://api.getsillage.com/api/v3
```

**Always use `/api/v3`.** v3 covers the whole direct-client surface with one envelope, one error shape,
`snake_case` everywhere and numeric ids. v2 is frozen (still answers, no new features); v1 is legacy.
Never mix versions in one integration. Some v2 resources have no v3 port on purpose: `agents` (replaced
by `signal-setups`), `watchlists` and `top-account-list` (replaced by `lists`), the singular `persona`
(use `personas`), `enrich-company-mapping`, and the partner signal create/update. The full spec is at `GET /api/v3/docs/spec`.

**Deprecation headers.** Every `/v2/agents` route answers with `Deprecation: true`,
`Sunset: Thu, 01 Apr 2027 00:00:00 GMT` and `Link: <…/docs/api/v3/signal-setups>; rel="deprecation"`
— errors included. After that date it stops answering: migrate to `/v3/signal-setups`.

## Authentication

```
Authorization: Bearer sk_live_...
```

- The **workspace is inferred from the key** — it never appears in the URL. One key = one workspace.
- `sk_live_` — workspace key; what you want here. `mk_live_` — partner master key, used on the
  `/v3/partner/workspaces/{id}/…` surface (`sillage-partner-api`).
- `401` = key missing, invalid, or revoked.

### Getting a key (walk the user through this)

Only someone with admin access to the workspace can create it — Claude can't mint it:

1. Open **<https://www.getsillage.com/app/settings/api-keys>** (Sillage → Settings → API Keys), signed
   into the target workspace (switch workspace first — the key is bound to the active one).
2. Click **Generate API Key**, name it, set an expiration, generate.
3. **Copy it immediately — it's shown only once.**

Store it as an environment variable, never in code or the browser:

```bash
export SILLAGE_API_KEY="sk_live_..."
curl -s https://api.getsillage.com/api/v3/setup-state -H "Authorization: Bearer $SILLAGE_API_KEY"
```

`200` = the key works; `401` = it's wrong.

## Response envelope

- **One object:** `{ "data": { … } }` — counts, balance and setup state included.
- **A page:** `{ "data": [ … ], "meta": { "pagination": { page, page_size, page_count, total } } }`.
- **A search (cursor):** `{ "data": [ … ], "meta": { "cursor": { "next", "has_more" } } }`.
- **No page to describe** (e.g. a signal-run launch): `{ "data": [ … ] }`.
- **Warnings** (advisory text, e.g. after a persona write) sit in `meta.warnings`; `meta` is omitted
  when there is none.

Read everything under `.data` (`jq '.data'`). Every response carries `X-Request-Id` (quote it to
support) and `X-Response-Time`.

A synchronous write answers with the same shape its `GET` returns, read back after the write — no
need to re-fetch.

## Pagination

- **Page number (plain lists):** `page` (≥1, default 1) and `page_size` (max 100). The end is
  `meta.pagination.page == meta.pagination.page_count`.
- **Cursor (every `POST …/query` search except `POST /companies/query`, and `GET /usage`):** send `limit` (max 100) and, from the
  second page on, `cursor` = the previous `meta.cursor.next`, **with the same filters**. Stop when
  `meta.cursor.has_more` is `false`. A cursor is bound to its endpoint, workspace and filters: an
  altered or reused one answers `422`.
- **`POST /companies/query`** is a capped search: `{ "data": [ … ] }`, no `meta`, no next page.

## Writes that merge vs replace

- **Signal setups** change through `PATCH`: sent `parameters` keys replace the stored ones, `null`
  removes a key, the rest stays. `list_id` and `persona_id` are fixed (a different value → `409`).
- **Personas** (`POST /v3/personas`, `PATCH /v3/personas/{id}`) are **strict**: an unknown or
  misspelled field — including the v2 singular names `job_title`, `location` — is a `422`, never
  silently dropped. `PATCH` changes only the fields sent.

## The async / poll model

- **A signal-run launch** (`POST /signal-runs` `{signal_setup_id}`) answers **`202 Accepted`** with
  `data[]`: one run per direction started, plus a Discovery run when the setup has that scope. Poll each `GET /signal-runs/{id}` until `data.status` is
  `completed` or `failed`. Re-launching is safe — detections are deduplicated.
- **Account mapping and content** start on their own, not on a call: mapping when companies are added
  to a customer, competitor or partner list, and for monitored accounts (monitoring pays for it).
  Follow them with `GET /requests-status` until nothing is left in flight for those accounts.

Run, request and content-request statuses are `pending` → `in_progress` → `completed` / `failed`.
Poll politely: a few seconds with backoff.

## Errors — RFC 9457 `application/problem+json`

```json
{
  "type": "about:blank",
  "title": "Human-readable category",
  "status": 422,
  "detail": "Specific explanation",
  "instance": "/api/v3/<path>",
  "errors": { "field": ["validation message"] }
}
```

On v3 `type` is always `about:blank` — **`title` is what tells one problem from another**. `errors`
is present on `400` / `422`. Read `detail` and `errors`; don't retry blindly.

| Status | Meaning                                                                                                                   | What to do                                |
| ------ | ------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------- |
| `400`  | Malformed request / invalid query                                                                                         | Fix params (see `errors`).                |
| `401`  | Bad or missing key                                                                                                        | Fix auth.                                 |
| `402`  | Not enough credits (e.g. a list created with monitoring on)                                                               | Surface to the user.                      |
| `403`  | Not in the plan (contents), workspace blocked (on a launch), or a **monitoring write from a non-admin app user**        | Surface it — do **not** retry.            |
| `404`  | Not found / not visible to this key                                                                                       | Check the id.                             |
| `409`  | Conflict (changing a setup's list or persona, launching a disabled setup, a list attached to another business unit)       | Resolve first (recreate the setup).       |
| `422`  | Valid JSON, wrong content (unknown field, incomplete persona, setup `parameters` failing the template schema, bad cursor) | Fix the payload field by field.           |
| `429`  | Rate limited                                                                                                              | Wait `Retry-After`, then retry.           |
| `500`  | Server error                                                                                                              | Retry with backoff; quote `X-Request-Id`. |

The monitoring `403` applies to app sessions only: an `sk_live_` key isn't role-checked (creating a key
is already admin-only). Monitoring writes are anything that sets `signal_buckets` (list create/update,
`PUT /account-monitoring/accounts`) and changes that extend monitoring on a monitored list (adding
members, changing its `persona_ids`, restoring it).

## Rate limits

Responses carry `X-RateLimit-Limit`, `X-RateLimit-Remaining` and `X-RateLimit-Reset`; a `429` adds
`Retry-After` (seconds). Slow down **before** `Remaining` hits zero.

## Company filters

The contents / content-requests / signals / leads searches filter companies by `company_ids`,
`company_linkedin_handle`, `company_linkedin_url` and `company_domain`. Everything you send combines
as a **union**: a company matches when it is any of the ids or any of the identifiers.

- `company_ids` is the **workspace-company** id (the `company_id` signals, leads and monitoring return), taken at face
  value: an unknown id gives an empty page, never `404`.
- Max 100 values per field (more → `422`). Send them in the JSON body of `POST …/query`.
