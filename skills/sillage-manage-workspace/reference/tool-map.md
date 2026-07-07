# Tool map — the MCP v2 tools, grouped

Every `sillage_v2_*` tool you use to set up or edit a workspace, with its key params and its write
class. **Write class is the thing to get right:**

- **READ** — safe, no change.
- **CREATE** — makes a new object.
- **PUT (replace-whole)** — overwrites the entire object; read first, send the complete object.
- **APPEND** — adds to a collection; existing items are kept.
- **DESTRUCTIVE** — removes; not reversible.
- **TRIGGER** — enqueues async work; poll a status tool afterwards.

## Setup / meta

| Tool                  | Class | Key params | Notes                                                                                                                                                                                                                      |
| --------------------- | ----- | ---------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `get_setup_state`     | READ  | —          | `{persona_set, list_uploaded, ingestion_complete, has_contents}`.                                                                                                                                                          |
| `get_rate_limit`      | READ  | —          | Quota check; surface `retry_after` on 429 instead of hammering.                                                                                                                                                            |
| `get_requests_status` | READ  | `page?`    | Everything in flight, one call: flat `requests[]` typed `account_mapping` \| `top_account_content` \| `signal`, each with `status` (`pending`\|`in_progress`) and a plain-language `label`. Empty array = nothing running. |

## Persona

| Tool             | Class         | Key params                                                                                                        | Notes                                                          |
| ---------------- | ------------- | ----------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| `get_persona`    | READ          | —                                                                                                                 | Returns the current ICP, or null if none.                      |
| `upsert_persona` | PUT (replace) | `job_title[]`, `exclude_job_title[]`, `seniority[]`, `headcount[]`, `industry[]`, `location[]`, `additional_info` | Replaces the whole persona. Send only these documented fields. |

Exact allowed values (seniority enum, headcount ranges) are in `sillage-onboarding`'s
`reference/what-sillage-needs.md` — the single source of truth for persona fields.

## Target accounts (TAL)

| Tool                          | Class           | Key params                                | Notes                                                                                                         |
| ----------------------------- | --------------- | ----------------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| `read_top_account_list`       | READ            | `view: accounts \| not_found`             | **The real target list.** Use this to know what you're targeting.                                             |
| `get_top_accounts`            | READ            | `limit?`                                  | Superset — every company with any activity trace, **not** the TAL.                                            |
| `add_top_accounts`            | APPEND, TRIGGER | `accounts:[{domain}\|{linkedin_url}]`     | Prefer `domain`. Enqueues ingestion → poll status.                                                            |
| `remove_top_accounts`         | DESTRUCTIVE     | `ids:[int]` **or** `identifiers:[string]` | Exactly one of the two: numeric ids from `read_top_account_list`, or domains / LinkedIn URLs (≤100 per call). |
| `get_top_account_list_status` | READ            | —                                         | Ingestion state: `queued` → `processing` → `completed` \| `failed`.                                           |

## Coverage / company mapping

| Tool                        | Class   | Key params                                                           | Notes                                                                                                                                                                |
| --------------------------- | ------- | -------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `enrich_company`            | TRIGGER | exactly one of `{domain}` \| `{linkedin_url}` \| `{linkedin_handle}` | Builds coverage (people). Prefer domain; on an ambiguous domain (`409`), retry with the LinkedIn URL. Idempotent — retry reuses the request. Returns a `request_id`. |
| `get_account_mapping_stage` | READ    | `id` (the request id)                                                | Stage of one mapping request.                                                                                                                                        |
| `list_company_mappings`     | READ    | `page?, page_size?`                                                  | Lists mappings; gives you each `mapping_id`.                                                                                                                         |
| `get_company_mapping`       | READ    | `mapping_id`                                                         | The mapped people. Accepts the `request_id` from `enrich_company` directly as `mapping_id`. Only readable once the request is `completed`.                           |
| `get_company`               | READ    | `company_id`                                                         | One company's enrichment (name, domain, LinkedIn, headcount, industries, locations…). Fields are null until enriched.                                                |

## Watchlists

| Tool                      | Class       | Key params                                                                       | Notes                                                                                                                                                         |
| ------------------------- | ----------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `list_watchlist_types`    | READ        | —                                                                                | The valid `type` → `kind` pairs.                                                                                                                              |
| `create_watchlist`        | CREATE      | `type`, `title`, `description?`                                                  | `type` immutable; `kind` derived.                                                                                                                             |
| `list_watchlists`         | READ        | —                                                                                | All lists with counts.                                                                                                                                        |
| `get_watchlist`           | READ        | `watchlist_id`                                                                   |                                                                                                                                                               |
| `update_watchlist`        | PUT         | `watchlist_id`, `title?`, `description?`                                         | Title/description only; type can't change.                                                                                                                    |
| `delete_watchlist`        | DESTRUCTIVE | `watchlist_id`                                                                   |                                                                                                                                                               |
| `add_watchlist_entities`  | APPEND      | `kind`, `watchlist_id`, `entities:[{linkedin_url}\|{linkedin_handle}\|{domain}]` | ≤100 per call. Idempotent on (list, entity). LinkedIn URL/handle preferred; `domain` accepted as a fallback on **company** lists only (422 on profile lists). |
| `list_watchlist_entities` | READ        | `watchlist_id`                                                                   |                                                                                                                                                               |
| `remove_watchlist_entity` | DESTRUCTIVE | `watchlist_id`, entity id                                                        |                                                                                                                                                               |

## Agents

| Tool                   | Class       | Key params                                                           | Notes                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| ---------------------- | ----------- | -------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `create_agent`         | CREATE      | `name`, `type`, `tracking_keywords?`, `watchlist_id?`                | **Eight** types: `keyword_detection`, `job_posting_keyword_detection` (watches your tracked companies' job postings for the keywords — both keyword types need `tracking_keywords`, optional `max_posts_to_scrape` / `start_date`), `job_update` (no params at create — name + type only), `competitor`, `partner`, `customer`, `influencer`, `champion`. Watchlist types auto-create + bind a list unless you pass `watchlist_id` (its type must match). Created enabled. |
| `get_agents`           | READ        | `agent_id?`, `response_format?`                                      | List, or one agent. `concise` (default) omits `parameters` — ask for `detailed` when you need the keywords. Reads can return `type: "unconfigured"` (agent not set up yet).                                                                                                                                                                                                                                                                                                |
| `configure_agent`      | PUT         | `agent_id`, `tracking_keywords?`, `start_date?`, `enabled?`, `name?` | Rename, change keywords, pause/resume. Changing any watch parameter requires sending the **full** `tracking_keywords` list — it replaces the existing one.                                                                                                                                                                                                                                                                                                                 |
| `bind_agent_watchlist` | PUT         | `agent_id`, `watchlist_id`                                           | Link/swap the bound list; both null = unlink.                                                                                                                                                                                                                                                                                                                                                                                                                              |
| `delete_agent`         | DESTRUCTIVE | `agent_id`                                                           |                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |

## Signals — runs & results

| Tool                | Class   | Key params                   | Notes                                                                                                                                                                                                                               |
| ------------------- | ------- | ---------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `launch_signal_run` | TRIGGER | `agent_id`, `lookback_days?` | Agent goes and looks. Returns `runs[]`, each with its own `signal_request_id` — keyword/job → 1 run; watchlist → 2 (inbound + outbound). `lookback_days` is a **top-level integer** (1–180, default 90) — not nested, not a string. |
| `get_signal_run`    | READ    | `signal_request_id`          | Stage `running` → `completed` / `completed_partial` / `failed`; per-job counts.                                                                                                                                                     |
| `list_signals`      | READ    | `page?`, `page_size?`        | The detection feed, offset-paginated. No filter params — filter client-side.                                                                                                                                                        |

> Signals are **read-only** through the MCP — there are no `list_signal_types`, `add_signal`,
> `update_signal`, or `delete_signal` tools (older skill versions referenced them; they were never
> shipped). The detection taxonomy lives in `sillage-help`'s `reference/signal-taxonomy.md`.

## Content

| Tool                   | Class | Key params                                                                                     | Notes                                                                                                                                                                                         |
| ---------------------- | ----- | ---------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `get_contents`         | READ  | `company_domain?[]`, `company_id?`, `person_id?`, `date_from?`, `date_to?`, `response_format?` | The corpus. `response_format`: `normalized` (default — clean typed projection), `concise` (no `data`), `detailed` (**deprecated alias of `normalized`**).                                     |
| `get_content_requests` | READ  | `type?`, `company_domain?[]`, `stage?`                                                         | The underlying request/job records with timing. `stage` takes comma-separated values (e.g. `'pending,in_progress'`); defaults to all non-completed — pass `'completed'` to see finished ones. |
