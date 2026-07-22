# Conventions — the rules every v2 REST call obeys

Read this once; it applies to all endpoints in `endpoint-catalog.md`.

## Base URL & versioning — always use v2

```
https://api.getsillage.com/api/v2
```

**Always use `/api/v2`. v1 is legacy — do not use it.** If you (or the user) have an older integration
on `/api/v1`, migrate it to v2: same key, swap the path prefix, and switch conventions (v1 uses
camelCase params like `pageSize` and nanoid document ids; v2 uses snake_case `page_size` and numeric
SQL ids). Never mix the two.

Every section of this skill is v2 — there is no reason to touch a v1 endpoint.

## Authentication

```
Authorization: Bearer sk_live_...
```

- The **workspace is inferred from the key** — it never appears in the URL. One key = one workspace.
- `sk_live_` — direct-client workspace key. This is what you want for single-workspace work.
- `mk_live_` — partner **master** key. Broader scope (can operate across workspaces / create them).
- `401` = key missing, invalid, or revoked.

### Getting a key (walk the user through this)

The key can only be created by someone with access to the workspace — Claude can't mint it. Point the
user at the settings page and have them paste the key back:

1. Open **<https://www.getsillage.com/app/settings/api-keys>** (Sillage → Settings → API Keys), signed
   into the target workspace. Switch workspace first if they have more than one — the key is bound to
   whichever workspace is active.
2. Click **Generate API Key**, give it a name, set an expiration, then generate.
3. **Copy it immediately — it's shown only once.** If lost, generate a new one.

### Setting the key

Store it as an environment variable; never commit it or expose it in browser/client-side code:

```bash
export SILLAGE_API_KEY="sk_live_..."
```

Then send it on every request: `-H "Authorization: Bearer $SILLAGE_API_KEY"`. Smoke-test it with
`curl -s https://api.getsillage.com/api/v2/persona -H "Authorization: Bearer $SILLAGE_API_KEY"` — a
`200` (even `{"data":null}`) means the key works; `401` means it's wrong.

## Response envelopes

- **Single object:** `{ "data": { ... } }`.
- **Offset list:** `{ "data": [ ... ], "meta": { "pagination": { page, page_size, page_count, total } } }`.
  Also mirrored in headers: `X-Total-Count`, `X-Page`, `X-Page-Size`, `X-Page-Count`.
- **Cursor list (signals only):** `{ "data": [ ... ], "meta": { "next_cursor", "has_more" } }`.

Every response carries `X-Request-Id` (quote it in support requests) and `X-Response-Time`.

## Pagination

**Offset (the default, almost everything):** `page` (≥1, default 1) and `page_size` (default 25,
max 100 — except agents, max 25). You've reached the end when `meta.pagination.page ==
meta.pagination.page_count`.

**Cursor (only `GET /workspace/signals`):** pass `limit` (default 25, max 100) and, from the second
page on, `cursor = <previous response's meta.next_cursor>`. Stop when `has_more` is `false`. This is a
deliberate divergence from the offset envelope — the published pagination doc page understates it.

**Capped (only `GET /top-accounts`):** not paginated at all — a single flat list capped at `limit`
(default 100, max 250), with only `meta.limit` returned.

## The async / poll model

Three kinds of write are asynchronous and return **`202 Accepted`** — the body only confirms the work
was _accepted_, not done. Poll the matching status endpoint to a terminal state before reading results.
**Retrying any of these is safe** — it won't create duplicates.

| You called                                                    | Poll                                           | Terminal states                                    |
| ------------------------------------------------------------- | ---------------------------------------------- | -------------------------------------------------- |
| `POST /top-account-list` or `POST /top-account-list/accounts` | `GET /top-account-list/status` → `state`       | `completed`, `failed` (from `queued`/`processing`) |
| `POST /enrich-company-mapping` → `{request_id}`               | `GET /content-requests/{request_id}` → `stage` | `completed`, `account_mapping_failed`              |
| `POST /workspace/signal-runs` → `[{signal_request_id}]`       | `GET /workspace/signal-runs/{id}` → `stage`    | `completed`, `completed_partial`, `failed`         |

`completed_partial` on a run means some accounts weren't scanned — the ids are in
`metadata.failed.dropped_account_ids`; re-run to cover them. For a whole-workspace view of what's
in flight, `GET /requests-status` — a flat `requests[]` array, each item typed `account_mapping` /
`top_account_content` / `signal`.

**Poll politely:** a few-second interval with backoff, not a tight loop. Coverage/ingestion can take
minutes.

## Errors — RFC 9457 `application/problem+json`

Error bodies are problem documents, not the normal envelope:

```json
{
  "type": "https://docs.getsillage.com/errors/<slug>",
  "title": "Human-readable category",
  "status": 403,
  "detail": "Specific explanation",
  "instance": "/api/v2/<path>",
  "errors": { "field": ["validation message"] }
}
```

`errors` is present only on `400` / `422`. Read `detail` and `errors` — don't retry blindly.

| Status | Meaning                                                                                                 | What to do                                                          |
| ------ | ------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| `400`  | Malformed request / invalid query                                                                       | Fix params (see `errors`).                                          |
| `401`  | Bad or missing key                                                                                      | Fix auth.                                                           |
| `402`  | Insufficient credits                                                                                    | Surface to the user; can't proceed.                                 |
| `403`  | Feature not enabled for this workspace (e.g. Account Mapping)                                           | Surface it — do **not** retry.                                      |
| `404`  | Not found / not visible to this key / unresolved identifier                                             | Check the id or identifier.                                         |
| `409`  | Conflict (e.g. deleting a watchlist with bound agents; a duplicate enrich in flight)                    | Resolve the conflict first.                                         |
| `422`  | Structurally valid but semantically wrong (conflicting identifiers, type mismatch, all entities failed) | Fix the payload.                                                    |
| `429`  | Rate limited                                                                                            | Back off (see below).                                               |
| `500`  | Server error                                                                                            | Retry with backoff; if persistent, quote `X-Request-Id` to support. |

## Rate limits

`429` responses (and normal responses) carry:

- `X-RateLimit-Limit` — max requests per window.
- `X-RateLimit-Remaining` — remaining in the current window.

Read `X-RateLimit-Remaining` and slow down **before** you hit zero; on `429`, back off and retry.

## The identifier waterfall (company filters)

On the contents / content-requests / signals read endpoints you can filter companies by several
identifier types. Resolution uses a **single-winning-tier** waterfall — only the highest-priority tier
you supply is used, lower tiers are ignored:

```
company_id  >  company_linkedin_handle  >  company_linkedin_url  >  company_domain
```

- `company_id` is taken at face value (no lookup) → an unknown/untracked id returns an **empty** page,
  never `404`.
- String identifiers (`domain` / `handle` / `url`) that resolve to nothing return **`404`** on the
  signals endpoints.
- Multi-value = union within the winning tier. Max 100 values. Over 100 → `400`.
- When your identifier list is long enough to blow the URL length, use the `POST .../query` body
  variants of `/contents` and `/content-requests` (native JSON arrays, same filters).

The `meta.resolved_companies[]` array on contents/content-requests tells you which tier matched each
identifier (`matched_by`).
