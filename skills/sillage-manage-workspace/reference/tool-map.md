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

| Tool              | Class | Key params | Notes                                                             |
| ----------------- | ----- | ---------- | ----------------------------------------------------------------- |
| `get_setup_state` | READ  | —          | `{persona_set, list_uploaded, ingestion_complete, has_contents}`. |
| `get_rate_limit`  | READ  | —          | Quota check; surface `retry_after` on 429 instead of hammering.   |

## Persona

| Tool             | Class         | Key params                                                                                                        | Notes                                                          |
| ---------------- | ------------- | ----------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| `get_persona`    | READ          | —                                                                                                                 | Returns the current ICP, or null if none.                      |
| `upsert_persona` | PUT (replace) | `job_title[]`, `exclude_job_title[]`, `seniority[]`, `headcount[]`, `industry[]`, `location[]`, `additional_info` | Replaces the whole persona. Send only these documented fields. |

Exact allowed values (seniority enum, headcount ranges) are in `sillage-onboarding`'s
`reference/what-sillage-needs.md` — the single source of truth for persona fields.

## Target accounts (TAL)

| Tool                          | Class           | Key params                               | Notes                                                              |
| ----------------------------- | --------------- | ---------------------------------------- | ------------------------------------------------------------------ |
| `read_top_account_list`       | READ            | `view: accounts \| content \| not_found` | **The real target list.** Use this to know what you're targeting.  |
| `get_top_accounts`            | READ            | `limit?`                                 | Superset — every company with any activity trace, **not** the TAL. |
| `add_top_accounts`            | APPEND, TRIGGER | `accounts:[{domain}\|{linkedin_url}]`    | Prefer `domain`. Enqueues ingestion → poll status.                 |
| `remove_top_accounts`         | DESTRUCTIVE     | `ids:[int]`                              | Remove by ids read from `read_top_account_list`.                   |
| `get_top_account_list_status` | READ            | —                                        | Ingestion state: `queued` → `processing` → `completed` \| `failed`. |

## Coverage / company mapping

| Tool                        | Class   | Key params                    | Notes                                                                           |
| --------------------------- | ------- | ----------------------------- | ------------------------------------------------------------------------------- |
| `enrich_company`            | TRIGGER | `{domain}\|{linkedin_handle}` | Builds coverage (people). Prefer domain. Idempotent — retry reuses the request. |
| `get_account_mapping_stage` | READ    | `id` (the request id)         | Stage of one mapping request.                                                   |
| `list_company_mappings`     | READ    | `page?, page_size?`           | Lists mappings; gives you the `mapping_id` (≠ request id).                      |
| `get_company_mapping`       | READ    | `mapping_id`                  | The mapped people. Only exists once the request is `completed`.                 |

## Watchlists

| Tool                      | Class       | Key params                                                             | Notes                                                                   |
| ------------------------- | ----------- | ---------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `list_watchlist_types`    | READ        | —                                                                      | The valid `type` → `kind` pairs.                                        |
| `create_watchlist`        | CREATE      | `type`, `title`, `description?`                                        | `type` immutable; `kind` derived.                                       |
| `list_watchlists`         | READ        | —                                                                      | All lists with counts.                                                  |
| `get_watchlist`           | READ        | `watchlist_id`                                                         |                                                                         |
| `update_watchlist`        | PUT         | `watchlist_id`, `title?`, `description?`                               | Title/description only; type can't change.                              |
| `delete_watchlist`        | DESTRUCTIVE | `watchlist_id`                                                         |                                                                         |
| `add_watchlist_entities`  | APPEND      | `kind`, `watchlist_id`, `entities:[{linkedin_url}\|{linkedin_handle}\|{domain}]` | ≤100 per call. Idempotent on (list, entity). LinkedIn URL/handle preferred; `domain` accepted as a fallback on **company** lists only (422 on profile lists). |
| `list_watchlist_entities` | READ        | `watchlist_id`                                                         |                                                                         |
| `remove_watchlist_entity` | DESTRUCTIVE | `watchlist_id`, entity id                                              |                                                                         |

## Agents

| Tool                   | Class       | Key params                                            | Notes                                                                                                                                                                                                                     |
| ---------------------- | ----------- | ----------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `create_agent`         | CREATE      | `name`, `type`, `tracking_keywords?`, `watchlist_id?` | Types: `keyword_detection`, `job_update`, `competitor`, `partner`, `customer`, `influencer`, `champion`. `job_update` takes no params at create (name + type only). Watchlist types auto-create + bind a list unless you pass `watchlist_id` (its type must match). Created enabled. |
| `get_agents`           | READ        | `agent_id?`, `response_format?`                       | List, or one agent detailed. `job_update` params: `job_titles[]`, `seniority_levels[]`.                                                                                                                                   |
| `configure_agent`      | PUT         | `agent_id`, `tracking_keywords?`, `enabled?`, `name?` | Rename, change keywords, pause/resume. Keywords/enabled only — can't set job titles here.                                                                                                                                 |
| `bind_agent_watchlist` | PUT         | `agent_id`, `watchlist_id`                            | Link/swap the bound list; both null = unlink.                                                                                                                                                                             |
| `delete_agent`         | DESTRUCTIVE | `agent_id`                                            |                                                                                                                                                                                                                           |

## Signals — runs & results

| Tool                | Class   | Key params                | Notes                                                                                                                                            |
| ------------------- | ------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `launch_signal_run` | TRIGGER     | `agent_id`, `parameters?`                                              | Agent goes and looks. Returns `runs[]`, each with its own `signal_request_id` — keyword/job → 1 run; watchlist → 2 (inbound + outbound). Put per-run options like `lookback_days` **inside `parameters`**, as a **number** (strings are rejected). |
| `get_signal_run`    | READ        | `signal_request_id`                                                    | Stage `running` → `completed` / `completed_partial` / `failed`; per-job counts.                                                                  |
| `list_signals`      | READ        | `page?`, `page_size?`                                                  | The detection feed; paginate with `has_more`. No filter params — filter client-side.                                                             |
| `list_signal_types` | READ        | —                                                                      | The detection taxonomy.                                                                                                                          |
| `add_signal`        | CREATE      | `signal_type`, `interaction_data`, `workspace_company_document_id`     | Manually insert a detection (e.g. from your own source). Optional: `workspace_lead_document_id`, `agent_document_id`.                            |
| `update_signal`     | PUT         | signal id + fields                                                     | Modify an existing detection.                                                                                                                    |
| `delete_signal`     | DESTRUCTIVE | signal id                                                              | Remove a detection; not reversible.                                                                                                              |

## Content

| Tool                   | Class | Key params                                                                                    | Notes                                                                      |
| ---------------------- | ----- | --------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| `get_contents`         | READ  | `company_domain?[]`, `company_id?`, `author_profile_id?`, `content_type?`, `response_format?` | The corpus. `response_format`: `concise` \| `normalized` \| `detailed` — only `detailed` carries the full author/actor object. |
| `get_content_requests` | READ  | `type?`, `company_domain?[]`, `stage?`                                                        | The underlying request/job records with timing. `stage` takes **one** value — comma-joined lists are rejected.                 |
