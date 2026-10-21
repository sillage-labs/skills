# Endpoint catalog — every v3 REST endpoint, grouped

All paths are relative to `https://api.getsillage.com/api/v3`. All calls carry
`Authorization: Bearer sk_live_...`. Every answer is `{ data }` (pages add `meta.pagination` or
`meta.cursor`). **Write class** is the thing to get right:

- **READ** — safe, no change.
- **CREATE** — makes a new object (`201`).
- **PATCH (merge)** — send only what changes; the rest is kept.
- **PUT** — the fields you send replace theirs (lists: name, description, archive, personas, monitoring).
- **DESTRUCTIVE** — removes.
- **TRIGGER (async, `202`)** — enqueues work; poll after (see `conventions.md`).
- **PAID** — spends credits.

## Contents

- [Setup state & what's running](#setup-state--whats-running)
- [Personas](#personas)
- [Lists](#lists)
- [Account monitoring](#account-monitoring)
- [Signal templates](#signal-templates)
- [Signal setups](#signal-setups)
- [Signal runs](#signal-runs)
- [Signals (detections)](#signals-detections)
- [Leads & companies](#leads--companies)
- [Account mapping](#account-mapping)
- [Contents & content requests](#contents--content-requests)
- [Credits & usage](#credits--usage)
- [Still v2 only](#still-v2-only)

## Setup state & what's running

| Method & path          | Class | Notes                                                                                                                                                                                                                                                                                             |
| ---------------------- | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `GET /setup-state`     | READ  | `data.checklist` with `persona` (`job_titles_set`, `locations_set`), `accounts` (`count`), `signal_setups` (`count`, `missing_priority_types` = template keys worth adding), `contents` (`has_contents`). Each has `status`, `message`, `recommended_next_route` (e.g. `POST /v3/signal-setups`). |
| `GET /requests-status` | READ  | Everything in flight, newest first: items typed `account_mapping` / `key_account_content` / `signal`, each with `status` and a plain `label`. `page_size` 1–500, default 500.                                                                                                                     |

## Personas

A persona is a business unit. Fields: `name`, `job_titles[]`, `excluded_job_titles[]`, `locations[]`,
`headcounts[]`, `industries[]`, `seniorities[]`, `exclusion_criteria[]`, `tag`.

| Method & path                  | Class  | Notes                                                                                                            |
| ------------------------------ | ------ | ---------------------------------------------------------------------------------------------------------------- |
| `GET /personas`                | READ   | Paged list.                                                                                                      |
| `POST /personas`               | CREATE | Unknown fields → `422`. `403` when a persona exists and business units aren't enabled. Advice may come back in `meta.warnings`. A setup needs `job_titles` **and** `locations`. |
| `GET /personas/{persona_id}`   | READ   | One persona.                                                                                                     |
| `PATCH /personas/{persona_id}` | PATCH  | Sent fields replace theirs; at least one field.                                                                  |

Enum values (`seniorities`, `headcounts`) live in `sillage-onboarding`'s `reference/what-sillage-needs.md`.

## Lists

`type` → `kind`: `competitor` / `partner` / `customer` / `prospect` → `company`; `influencer` /
`champion` / `employee` → `profile`. `kind` is part of every single-list path. The one `employee`
list ("All my employees") is created by Sillage: add/remove people, never create or delete it.

| Method & path                                        | Class       | Notes                                                                                                                                                                                                                                                                                                                                           |
| ---------------------------------------------------- | ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `POST /lists`                                        | CREATE      | `{type, name, description?, tier?, persona_ids?, signal_buckets?}`. `tier` (1–3) and `signal_buckets` (`["essential"]` = monitored, **paid**) on `customer`/`prospect` only.                                                                                                                                                                    |
| `GET /lists`                                         | READ        | `type?`, `persona_id?` (that persona's + unattached lists), `include_archived?`, `response_format?`. Items carry `member_count`, `bound_signal_setup_count`, `mode`, `is_deletable`.                                                                                                                                                            |
| `GET /lists/{kind}/{list_id}`                        | READ        | One list.                                                                                                                                                                                                                                                                                                                                       |
| `PUT /lists/{kind}/{list_id}`                        | PUT         | `{name?, description?, archived?, persona_ids?, signal_buckets?}`. `type` never changes. `signal_buckets` monitors/stops the whole list (**paid**).                                                                                                                                                                                             |
| `DELETE /lists/{kind}/{list_id}`                     | DESTRUCTIVE | Refused while a signal setup watches it, for the "All my employees" list, and for the last list of a tier. Reversible with `POST …/restore`.                                                                                                                                                                                                                                                                      |
| `POST /lists/{kind}/{list_id}/restore`               | WRITE       | Bring a deleted list back.                                                                                                                                                                                                                                                                                                                      |
| `GET /lists/{kind}/{list_id}/members`                | READ        | `q?`, `signal_buckets?`, `persona_id?`, `monitoring_status?` (customer/prospect).                                                                                                                                                                                                                                                               |
| `POST /lists/{kind}/{list_id}/members`               | MERGE       | `{members:[{linkedin_url}\|{linkedin_handle}\|{domain}]}`, 1–100. **Partial success:** added in `data`, failures in `errors[]` (`code`: `multiple_domain_matches`, `domain_not_resolved`, `resolution_failed`, `wrong_entity_type`, `invalid_identifier`). `domain` on company lists only. Smart lists refuse writes. Paid on a monitored list. |
| `DELETE /lists/{kind}/{list_id}/members/{member_id}` | DESTRUCTIVE | `member_id` is the company id / lead id the rest of the API uses.                                                                                                                                                                                                                                                                               |
| `GET /lists/{kind}/{list_id}/membership-logs`        | READ        | Who entered and left, when.                                                                                                                                                                                                                                                                                                                     |

## Account monitoring

Key accounts (customer/prospect list members) are monitored per account **and** per persona — paid.
`403` for an app user who isn't a workspace admin.

| Method & path                      | Class     | Notes                                                                                                                                                                                                |
| ---------------------------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `GET /account-monitoring`          | READ      | `persona_id?`. `is_billed`, `price_per_account_credits`, `monitored_count`, `not_covered_count`, `monthly_cost_credits`, `by_persona[]`.                                                             |
| `PUT /account-monitoring/accounts` | PUT, PAID | `{company_ids:[…] (1–1000), signal_buckets: ["essential"] \| null, persona_id?}` (`persona_id` required with several personas). Answers `changed`, `already_in_state`, `refused`, `charged_credits`. |

## Signal templates

| Method & path                | Class | Notes                                                                                                                                                                                                                                                                             |
| ---------------------------- | ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `GET /signal-templates`      | READ  | Live only unless `include_coming_soon=true`. Filters (comma lists): `category`, `subjects`, `scopes`, `tags` (`popular`, `not_set_up`), `name`, `persona_id` (resolves computed defaults).                                                                                        |
| `GET /signal-templates/{id}` | READ  | Fields: `id`, `key` (stable across environments), `name`, `description`, `category`, `status` (`live`/`coming_soon`), `scopes`, `pricing` (`discovery`, `account_monitoring`), `setup` (`fields` JSON Schema with `x-label`, `x-hint`, `x-control`, `x-list-type`, `x-honoured`). |

## Signal setups

| Method & path                | Class       | Notes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| ---------------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `POST /signal-setups`        | CREATE      | `{signal_template_id, parameters, name?, schedule?: "weekly"\|null, persona_id?}`. `parameters` validated against `setup.fields`; omitted ones take their default. `list_id` required when `setup.fields.required` lists it (content engagement defaults to "All my employees"). `persona_id` required with several personas. Errors: `422` per field (`parameters.<field>`, `persona_id`, title `Persona incomplete`), `409` archived/coming-soon template or list attached to other personas, `404` unknown template. `schedule: "weekly"` runs once on the next scheduler tick, then weekly (paid if `scopes` has `discovery`). |
| `GET /signal-setups`         | READ        | `persona_id?` (that persona's + unbound setups). Items: `signal_template`, `status` (`enabled`/`disabled`), `parameters` (as applied), `schedule`, `persona_id`, `last_detected_at`.                                                                                                                                                                                                                                                                                                                                                                                                                               |
| `GET /signal-setups/{id}`    | READ        | One setup.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| `PATCH /signal-setups/{id}`  | PATCH       | `{name?, status?, parameters?, schedule?, persona_id?}`. Parameters merge (`null` removes; refused on a defaulted one). Other `list_id` or `persona_id` → `409`. Boolean `enabled` → `422`.                                                                                                                                                                                                                                                                                                                                                                                                                        |
| `DELETE /signal-setups/{id}` | DESTRUCTIVE | Detections already found stay.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |

## Signal runs

| Method & path           | Class   | Notes                                                                                                                                                                                                                                                                                                                           |
| ----------------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `POST /signal-runs`     | TRIGGER | `{signal_setup_id, parameters?: {lookback_days 1–180}}` → `202 {data:[runs]}`: one per direction for list watchers, plus one Discovery run when `scopes` has `discovery` (paid). `409` a run in progress or setup disabled · `402` no credits for Discovery · `403` workspace blocked · `422` nothing eligible / misconfigured. |
| `GET /signal-runs`      | READ    | In-flight runs, not paged.                                                                                                                                                                                                                                                                                                      |
| `GET /signal-runs/{id}` | READ    | Poll target. `status`: `pending` → `in_progress` → `completed` \| `failed`; `metadata.failed.dropped_account_ids` when accounts were skipped.                                                                                                                                                                                   |

## Signals (detections)

| Method & path          | Class       | Notes                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| ---------------------- | ----------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `POST /signals/query`  | READ        | Cursor search, newest first. Body: `cursor`, `limit` (1–100), `signal_date_from/to`, `detected_at_from/to`, `signal_setup_id`, `signal_run_id`, `signal_type[]`, `company_ids[]`, `company_domain[]`, `company_linkedin_handle[]`, `company_linkedin_url[]`, `lead_id`, `countries[]`, `cities[]`. Items: `signal_type`, `details`, `detected_at`, `signal_date`, `lead_id`, `company_id`, `signal_setup_id`, `source_url`, `author`, `excerpt`. |
| `POST /signals/count`  | READ        | Same filters (no cursor) → `data.total`. A `GET` here hits `/{id}` and answers `422`.                                                                                                                                                                                                                                                                                                                                                            |
| `GET /signals/{id}`    | READ        | One detection.                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `DELETE /signals/{id}` | DESTRUCTIVE | Allowed with `sk_live_`.                                                                                                                                                                                                                                                                                                                                                                                                                         |

## Leads & companies

`company_id` everywhere is the workspace-company id (the one list members and signals carry).

| Method & path           | Class | Notes                                                                                                  |
| ----------------------- | ----- | ------------------------------------------------------------------------------------------------------ |
| `POST /leads/query`     | READ  | Cursor search: `sort` (`created_at:desc` default), `created_at_from/to`, company and location filters. |
| `POST /leads/count`     | READ  | Same filters → `data.total`.                                                                           |
| `GET /leads/{id}`       | READ  | One lead with `status`, `owner`, `crm_records`, `company`.                                             |
| `PATCH /leads/{id}`     | PATCH | `{status?, owner_email?}` (`owner_email: null` unassigns).                                             |
| `POST /companies/query` | READ  | `{name?, domain?, limit? (≤50)}`, capped, no cursor. Resolve a company to its `id`.                    |
| `GET /companies/{id}`   | READ  | Firmographics; every field nullable until enriched.                                                    |

## Account mapping

| Method & path                        | Class | Notes                                                                                                  |
| ------------------------------------ | ----- | ------------------------------------------------------------------------------------------------------ |
| `GET /company-mappings`              | READ  | `company_ids?`, `persona_id?`, `latest?`. Items: `id` (mapping id), `status`, `company`, `persona_id`. |
| `GET /company-mappings/{mapping_id}` | READ  | One mapping with its people.                                                                           |

## Contents & content requests

| Method & path                  | Class | Notes                                                                                                                                                                                                                         |
| ------------------------------ | ----- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `POST /contents/query`         | READ  | Cursor search: `content_type[]` (snake_case, e.g. `linkedin_post`, `linkedin_job_posting`), `created_at_from/to`, company filters, `lead_id`, `response_format` (`normalized` default \| `concise`). Payload under `details`. |
| `GET /contents/{id}`           | READ  | One item.                                                                                                                                                                                                                     |
| `POST /content-requests/query` | READ  | Cursor search: `type`, `status[]` (completed included by default), company filters.                                                                                                                                           |
| `GET /content-requests/{id}`   | READ  | One request: `status`, `label`, `persona_ids`.                                                                                                                                                                                |

## Credits & usage

| Method & path          | Class | Notes                                                                                                                                                                                                                  |
| ---------------------- | ----- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `GET /credits/balance` | READ  | Balance with its `batches` (`expires_at`).                                                                                                                                                                             |
| `GET /usage`           | READ  | Billing events, cursor in the **query string** (`cursor`, `limit`, `created_at_from/to`). Each: `credits`, `product` (`scope`, `signal_bucket`, `signal_template_key`, `signal_setup_id`, `company_id`, `persona_id`). |

## Still v2 only

No v3 route yet — call `https://api.getsillage.com/api/v2/...` for these: the target account list
(`/top-account-list/*`, superseded in v3 by customer/prospect lists) and the paid mapping launch
`POST /enrich-company-mapping`. `/v2/agents` and `/v2/watchlists` have v3 replacements (setups,
lists): migrate, don't build on them.
