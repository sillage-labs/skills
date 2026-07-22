# Sillage Partner API — endpoint catalog

Base URL `https://api.getsillage.com` · Auth `Authorization: Bearer $SILLAGE_API_KEY` (partner
master key `mk_live_…` — the key alone identifies the partner) · Errors: RFC 9457
`application/problem+json` · IDs: numeric integers, environment-specific · Dates: ISO 8601, UTC ·
Every response carries `X-Request-Id` (quote it when reporting an issue) · Live spec: Swagger UI
`/api/v2/partner/docs`, raw JSON `/api/v2/partner/docs/spec`.

Path shorthand: `…` = `/api/v2/partner/workspaces/{partner_workspace_id}`.
`{partner_workspace_id}` is the opaque id **you** assigned at creation — never a numeric id.

## Contents

- [Identity & workspaces](#identity--workspaces)
- [Persona](#persona)
- [Top account list](#top-account-list)
- [Account mapping](#account-mapping)
- [Companies & leads](#companies--leads)
- [Agents](#agents)
- [Watchlists](#watchlists)
- [Signals & signal runs](#signals--signal-runs)
- [Contents](#contents-resource)
- [Content requests](#content-requests)
- [Enums](#enums)

## Identity & workspaces

| Method               | Path                         | Purpose                                                                                                                                                                      |
| -------------------- | ---------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GET                  | `/api/v2/partner/me`         | Auth check → `data.{id,name,slug}`                                                                                                                                           |
| GET                  | `/api/v2/partner/workspaces` | List, newest first (`page`, `page_size`). Paginated — page through fully before concluding a workspace is missing; direct addressing by `partner_workspace_id` always works. |
| POST                 | `/api/v2/partner/workspaces` | Create. Body: `partner_workspace_id` (req, unique per partner), `end_customer_email` (req), `name` (req), `domain` (opt — recommended).                                      |
| GET / PATCH / DELETE | `…`                          | Read · update (PATCH accepts **only `name`**) · permanent delete                                                                                                             |
| POST                 | `…/archive`                  | Reversible archive (stops signals, keeps data) → 204                                                                                                                         |
| GET                  | `…/ping`                     | Key valid + workspace active                                                                                                                                                 |

## Persona

| Method | Path        | Notes                                                                                                                                                |
| ------ | ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| GET    | `…/persona` | `data` = persona or `null` if never set                                                                                                              |
| PUT    | `…/persona` | Create/replace. Fields (all opt): `job_title[]`, `exclude_job_title[]`, `location[]`, `headcount[]`, `industry[]`, `seniority[]`, `additional_info`. |

Persona rules:

- **Full replace, not merge** — omitted fields become unset. Always `GET` → merge → `PUT` the full
  object.
- **Unknown fields are silently dropped** — a typo'd body returns 200 and the persona is replaced
  with only what validated. Read the persona back after writing it.
- **Always include `location`** (country names; regions like "EMEA" are not guaranteed to resolve).
  Account mapping scopes its people search by it; enrichment requests without it fail.

## Top account list

| Method     | Path                                                   | Notes                                                                                                    |
| ---------- | ------------------------------------------------------ | -------------------------------------------------------------------------------------------------------- |
| POST       | `…/top-account-list/accounts`                          | **Add** (merge), 1–10,000 items, each `{linkedin_url}` and/or `{domain}` → 202, ingestion queued (async) |
| POST       | `…/top-account-list/accounts/remove`                   | Remove batch (idempotent) → `{deleted_count}`                                                            |
| GET        | `…/top-account-list/count`                             | Counts **found** accounts only (unresolved excluded)                                                     |
| GET / POST | `…/top-account-list`                                   | Read the list · **replace the whole list** (destructive — know which of add vs replace you're calling)   |
| GET        | `…/top-account-list/status`                            | Ingestion status — poll until `completed`                                                                |
| GET        | `…/top-account-list/accounts` · `…/accounts/not-found` | List found accounts · list misses                                                                        |
| DELETE     | `…/top-account-list/accounts/{id}`                     | Remove one                                                                                               |

A `domain` can occasionally fail to match without appearing in `not-found`. After ingestion, verify
the accounts you care about are actually present; re-add any miss via its `linkedin_url`.

## Account mapping

| Method | Path                              | Notes                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| ------ | --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| POST   | `…/enrich-company-mapping`        | Start async mapping for one company. Body: exactly one primary identifier — `domain`, `linkedin_url`, or `linkedin_handle` (`domain`+`linkedin_url` may be combined to disambiguate). → 202 `{status, request_id, stage}` — `request_id` is a **content-request id** (poll it there), not a `mapping_id`. Errors: 402 no credits · 403 feature not enabled · 409 ambiguous domain (add a LinkedIn URL). Re-triggering an in-flight company dedupes to the existing request. |
| GET    | `…/company-mappings`              | List materialized mappings (no profiles). `page`/`page_size` only — no status filter; each item has `status` (`in_progress` \| `complete`), filter client-side.                                                                                                                                                                                                                                                                                                             |
| GET    | `…/company-mappings/{mapping_id}` | One mapping + `profiles[]`: `linkedin_url`, `position`, `first_name`, `last_name`, `email`, `phone_number`, `linkedin_headline`, `location{city,region,country}`, …                                                                                                                                                                                                                                                                                                         |

Note: `mapping_id`, `company_id`, and top-account ids are **different id spaces** — don't pass one
where another is expected. To find a mapping from a domain, scan the company-mappings list.

## Companies & leads

| Method | Path               | Notes                                                                                                                                                                                         |
| ------ | ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GET    | `…/companies/{id}` | Resolve a `company_id` (from a signal or content item) into firmographics: name, domain, LinkedIn, locations, employee range, industries… Fields are `null` until enriched. Not a search API. |
| GET    | `…/leads/{id}`     | Resolve a `lead_id` into a profile + company block. Payloads carry no email/phone.                                                                                                            |

## Agents

| Method             | Path                  | Notes                                                                                                                                                                                                                                                                                       |
| ------------------ | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| POST               | `…/agents`            | Create. Body: `name` (1–100), `type` (see enum), `parameters` (per type), `watchlist_id` (opt, watchlist-bound types; omitted ⇒ a new empty watchlist is created and bound). Created `enabled:true` but with **no schedule** — it never runs on its own; fire it with `POST …/signal-runs`. |
| GET                | `…/agents`            | List. `page_size` default 10, **max 25** (422 above — a lower cap than the other lists).                                                                                                                                                                                                    |
| GET / PUT / DELETE | `…/agents/{agent_id}` | Read · partial update (`name`, `enabled`, `parameters`, `watchlist_kind`+`watchlist_id` — pass both `null` to unbind) · delete (already-detected signals are preserved)                                                                                                                     |

Per-type `parameters`: `keyword_detection` → `tracking_keywords[]` (req) + `start_date` (opt) ·
`job_posting_keyword_detection` → `tracking_keywords[]` (req; matched against job **title +
description** of the top accounts' postings) · all other types → none.

Agent facts:

- `tracking_keywords` matching is **semantic, not exact-phrase** — quotes don't force literal
  matches; near-synonyms multiply signal counts without new content. One keyword per concept.
- `agents[].last_run` is not populated by runs — poll the run, or query the signals, to judge one.
- `job_update` is driven by the **workspace persona** (applied when accounts are mapped), not by
  agent parameters, and fires on both arrivals and departures at your top accounts.

## Watchlists

| Method             | Path                                                      | Notes                                                                                                                                                                                                                                                                                                                         |
| ------------------ | --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| POST               | `…/watchlists`                                            | Create: `type` (immutable, derives `kind`), `title` (req), `description` (opt) → 201                                                                                                                                                                                                                                          |
| GET                | `…/watchlists`                                            | List; optional `type` filter; `page_size` max 100                                                                                                                                                                                                                                                                             |
| GET / PUT / DELETE | `…/watchlists/{kind}/{watchlist_id}`                      | `kind` = `company` \| `profile`. PUT: `title`/`description` only. DELETE → **409 while bound to an agent** (unbind first).                                                                                                                                                                                                    |
| POST               | `…/watchlists/{kind}/{watchlist_id}/entities`             | Add 1–100. Each: `linkedin_url` \| `linkedin_handle` (preferred) \| `domain` (company lists only). **Partial-success**: 200 returns `data` (added) + `errors` (per-entity: `multiple_domain_matches`, `apollo_failed`, `resolution_failed`); if every entity failed → 422. Re-adding present members = 200 with empty `data`. |
| GET                | `…/watchlists/{kind}/{watchlist_id}/entities`             | Paginate members (company lists → `name`; profile lists → `first_name`/`last_name`)                                                                                                                                                                                                                                           |
| DELETE             | `…/watchlists/{kind}/{watchlist_id}/entities/{entity_id}` | Remove the membership link (the canonical company/profile is never deleted)                                                                                                                                                                                                                                                   |

## Signals & signal runs

| Method | Path                 | Notes                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| ------ | -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| POST   | `…/signals/query`    | The list route (replaces the former `GET …/signals`). **Cursor pagination**: follow `meta.next_cursor` / `has_more`; `page`/`page_size` are ignored. Default window: last **90 days** (`signal_start_date`, cap 180). Body filters: `cursor`, `limit` (1–100, def 25), `signal_start/end_date`, `detection_start/end_date`, `agent_id`, `signal_run_id`, `company_id`, `company_domain` / `company_linkedin_handle` / `company_linkedin_url` (comma-separated OR). |
| GET    | `…/signals/count`    | Same filters → `{total}`                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| GET    | `…/signals/{id}`     | One signal: `signal_type`, `data` (type-specific), `detected_at`, `signal_date`, `lead_id`, `company_id`, `agent_id`, `source_url`, `author`, `excerpt`                                                                                                                                                                                                                                                                                                            |
| POST   | `…/signal-runs`      | On-demand run for **any** agent type (the run kind derives from the agent; `signal_key` is legacy — pass `"keyword"`). Body: `agent_id` (req), `signal_key` (req), `parameters` (req; `match_mode` `any` \| `all`, keyword agents only). → 202 with an **array** of `{signal_request_id, stage}` — watchlist-bound agents start **two** runs (inbound + outbound). Safe to retry: completed matches are not duplicated. Unknown `agent_id` → 404.                  |
| GET    | `…/signal-runs/{id}` | Poll (`id` = `signal_request_id`) until `completed`, `completed_partial`, or `failed`; on partial/failed, `metadata.failed.dropped_account_ids` lists accounts that weren't scanned.                                                                                                                                                                                                                                                                               |

## Contents (resource)

| Method | Path               | Notes                                                                                                                                                                       |
| ------ | ------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| POST   | `…/contents/query` | The list route (replaces the former `GET …/contents`). Body (all opt, `{}` OK): `page`, `page_size` (1–100, def 25), `date_from`/`date_to`, `content_type[]`, `company_id`. |
| GET    | `…/contents/{id}`  | One content item — returns the bare item (no `data` wrapper)                                                                                                                |

Item shape: `{id, content_type, created_at, data{…}, lead_id, company_id}` — `data` holds the
normalized content. Posts → `text`, `link`, `posted_at`, `engagement`, `author` (+ `mentions`,
`images` when present) · `linkedinComment` → also `post_content_id` (fetch the parent post by id) ·
`linkedinJobPosting` → `title`, `description`, `location`, `linkedin_url`. Absent fields are
**omitted**, not `null`.

## Content requests

A content request is one processing task — mapping an account (`account_mapping`) or collecting a
top account's content (`top_account_content`) — advancing through stages. This is the observable
state machine behind steps 3–4 of the chain; poll it to follow progress.

| Method | Path                      | Notes                                                                                                                                                                                                         |
| ------ | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GET    | `…/content-requests`      | List. **Default excludes `completed`** — pass `stage=completed` explicitly to see finished work. Filters: `page`/`page_size`, `date_from`/`date_to`, `type`, `stage` (repeatable), `company_id`, `person_id`. |
| GET    | `…/content-requests/{id}` | One request: `{id, type, stage, created_at, updated_at, company{}, inputs{}}`                                                                                                                                 |

## Enums

- **Agent types (creatable, 8)**: `keyword_detection`, `job_posting_keyword_detection`,
  `job_update`, `customer`, `competitor`, `partner`, `influencer`, `champion`. On a rejected `type`,
  the 422 message lists the accepted set — it is authoritative. Read-back may also show
  `job_posting`, `competitor_activity`, `content_engagement`, `influencer_engagement`,
  `champion_tracking`.
- **content_type (query filter)**: `linkedinPost`, `linkedinCompanyPost`, `linkedinJobPosting`,
  `linkedinComment`. Job posts = `linkedinJobPosting` (`jobPosting` is rejected with a 400 listing
  the accepted values).
- **Watchlist type → kind**: `competitor` | `partner` | `customer` → `company` · `influencer` |
  `champion` → `profile`.
- **Content-request stages** — terminal success `completed`. Account mapping:
  `account_mapping_ready`, `account_mapping_in_progress`, `account_mapping_ingestion_ready`,
  `account_mapping_ingestion_in_progress`, `account_mapping_failed`. Top-account content:
  `top_account_content_scraping_ready`, `top_account_content_scraping_starting`,
  `top_account_content_scraping_in_progress`, `top_account_content_ingestion_in_progress`,
  `top_account_content_failed`.
- **headcount**: `1-10`, `11-50`, `51-200`, `201-500`, `501-1,000`, `1,001-5,000`, `5,001-10,000`,
  `10,001+` (mind the commas).
- **seniority**: `owner`, `founder`, `c_suite`, `partner`, `vp`, `head`, `director`, `manager`,
  `senior`, `entry`, `intern`.
- **response_format** (watchlists): `concise`, `normalized` (default), `detailed`.
