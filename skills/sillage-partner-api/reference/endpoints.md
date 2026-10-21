# Sillage Partner API — endpoint catalog (v3)

Base URL `https://api.getsillage.com` · Auth `Authorization: Bearer $SILLAGE_MASTER_KEY` (partner
master key `mk_live_…` — the key alone identifies the partner) · Every answer `{ data }` · Errors:
RFC 9457 `application/problem+json`, `type: about:blank`, `errors` keyed by field on `422` · IDs:
numeric, environment-specific (except your `partner_workspace_id`) · Dates: ISO 8601, UTC · Live
spec: Swagger UI `/api/v3/partner/docs`, raw JSON `/api/v3/partner/docs/spec`.

Path shorthand: `…` = `/api/v3/partner/workspaces/{partner_workspace_id}` — the string id **you**
assigned at creation, URL-encoded. Each route behaves like its direct-client twin in
[`sillage-api`'s catalog](../../sillage-api/reference/endpoint-catalog.md); only the differences are
spelled out here.

## Contents

- [Workspaces](#workspaces)
- [Personas](#personas)
- [Lists](#lists)
- [Account monitoring](#account-monitoring)
- [Signal templates & setups](#signal-templates--setups)
- [Signal runs](#signal-runs)
- [Signals](#signals)
- [Leads & companies](#leads--companies)
- [Account mapping](#account-mapping)
- [Contents & content requests](#contents--content-requests)
- [Status & usage](#status--usage)
- [Still v2 only](#still-v2-only)
- [Enums](#enums)

## Workspaces

| Method | Path                         | Notes                                                                                                                                                                                                                                                                                                        |
| ------ | ---------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| POST   | `/api/v3/partner/workspaces` | Create. Body (all required): `partner_workspace_id` (1–255, unique among yours, deleted included; no `/ ? #` or spaces), `name`, `domain` (hostname), `owner_email` (gets admin access in the app) → `201`. `409` duplicate id, live workspace on that domain (add a persona instead), or concurrent create. |
| GET    | `/api/v3/partner/workspaces` | List (`page`, `page_size`, `include_archived`, `include_deleted`).                                                                                                                                                                                                                                           |
| GET    | `…`                          | One workspace: `partner_workspace_id`, `name`, `domain`, `archived`, `created_at`.                                                                                                                                                                                                                           |
| PATCH  | `…`                          | `{name?, archived?}`. `archived: true` pauses (no scheduled runs, content or weekly email; API and manual runs keep working). `domain` cannot change.                                                                                                                                                        |
| DELETE | `…`                          | Deletes; stops resolving everywhere. The id stays yours.                                                                                                                                                                                                                                                     |
| POST   | `…/restore`                  | Brings a deleted workspace back.                                                                                                                                                                                                                                                                             |

## Personas

| Method      | Path                      | Notes                                                                                                                                                                                                          |
| ----------- | ------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GET / POST  | `…/personas`              | List · create. Fields: `name`, `job_titles[]`, `excluded_job_titles[]`, `locations[]` (country names), `headcounts[]`, `industries[]`, `seniorities[]`, `exclusion_criteria[]`, `tag`. Unknown fields → `422`. |
| GET / PATCH | `…/personas/{persona_id}` | Read · update the sent fields only.                                                                                                                                                                            |

Signal setups need `job_titles` **and** `locations` on their persona.

## Lists

| Method             | Path                                           | Notes                                                                                                                                                                                                                                                                                |
| ------------------ | ---------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| POST               | `…/lists`                                      | `{type, name, description?, tier?, persona_ids?, signal_buckets?}`. `tier` and `signal_buckets` on `customer`/`prospect` only.                                                                                                                                                       |
| GET                | `…/lists`                                      | `type?`, `persona_id?`, `include_archived?`.                                                                                                                                                                                                                                         |
| GET / PUT / DELETE | `…/lists/{kind}/{list_id}`                     | `kind` = `company` \| `profile`. PUT: `name`, `description`, `archived`, `persona_ids`, `signal_buckets`. DELETE refused while a setup watches the list, for "All my employees", and for the last list of a tier.                                                                                                                             |
| GET / POST         | `…/lists/{kind}/{list_id}/members`             | Add 1–100 `{members:[{linkedin_url}\|{linkedin_handle}\|{domain}]}` (`domain` on company lists only). **Partial success**: `data` = added, `errors[]` per member (`multiple_domain_matches`, `domain_not_resolved`, `resolution_failed`, `wrong_entity_type`, `invalid_identifier`). |
| DELETE             | `…/lists/{kind}/{list_id}/members/{member_id}` | Remove one member.                                                                                                                                                                                                                                                                   |

No partner route for list restore or membership logs.

## Account monitoring

| Method | Path                            | Notes                                                                                                                                                           |
| ------ | ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GET    | `…/account-monitoring`          | Totals and cost (`persona_id?`).                                                                                                                                |
| PUT    | `…/account-monitoring/accounts` | `{company_ids[] (1–1000), signal_buckets: ["essential"] \| null, persona_id?}` — **paid**. Answers `changed`, `already_in_state`, `refused`, `charged_credits`. |

## Signal templates & setups

| Method               | Path                      | Notes                                                                                                                                                                                                                                       |
| -------------------- | ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| GET                  | `…/signal-templates`      | Live catalogue (`include_coming_soon`, `category`, `scopes`, `tags`, `name`, `persona_id`).                                                                                                                                                 |
| GET                  | `…/signal-templates/{id}` | `key`, `status`, `scopes`, `pricing`, `setup.fields` (JSON Schema + `x-label`, `x-hint`, `x-control`, `x-list-type`).                                                                                                                       |
| POST                 | `…/signal-setups`         | `{signal_template_id, parameters, name?, schedule?: "weekly"\|null, persona_id?}` → `201`. `list_id` required when `setup.fields.required` lists it; nothing auto-created. `422` per field / `Persona incomplete`; `409` list of other personas, archived template. |
| GET                  | `…/signal-setups`         | `persona_id?`.                                                                                                                                                                                                                              |
| GET / PATCH / DELETE | `…/signal-setups/{id}`    | PATCH merges `{name?, status?, parameters?, schedule?, persona_id?}`; other list or persona → `409`.                                                                                                                                        |

## Signal runs

| Method | Path                 | Notes                                                                                                                                                                             |
| ------ | -------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| POST   | `…/signal-runs`      | `{signal_setup_id, parameters?: {lookback_days}}` → `202 {data:[runs]}` (directions + optional paid Discovery). `409` already running / disabled, `402` no credits for Discovery. |
| GET    | `…/signal-runs`      | In-flight runs.                                                                                                                                                                   |
| GET    | `…/signal-runs/{id}` | `status` `pending` → `in_progress` → `completed` \| `failed`; `metadata.failed.dropped_account_ids`.                                                                              |

## Signals

| Method | Path              | Notes                                                                                                                                                                                                                                                                           |
| ------ | ----------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| POST   | `…/signals/query` | Cursor (`cursor`, `limit` 1–100). Filters: `signal_setup_id`, `signal_run_id`, `signal_type[]`, `signal_date_from/to`, `detected_at_from/to`, `company_ids[]`, `company_domain[]`, `company_linkedin_handle[]`, `company_linkedin_url[]`, `lead_id`, `countries[]`, `cities[]`. |
| POST   | `…/signals/count` | Same filters → `data.total`.                                                                                                                                                                                                                                                    |
| GET    | `…/signals/{id}`  | `signal_type`, `details`, `detected_at`, `signal_date`, `lead_id`, `company_id`, `signal_setup_id`, `source_url`, `author`, `excerpt`.                                                                                                                                          |
| DELETE | `…/signals/{id}`  | Remove one detection.                                                                                                                                                                                                                                                           |

## Leads & companies

| Method      | Path               | Notes                                                                                     |
| ----------- | ------------------ | ----------------------------------------------------------------------------------------- |
| POST        | `…/leads/query`    | Cursor search (`sort`, `created_at_from/to`, company and location filters).               |
| POST        | `…/leads/count`    | Same filters → `data.total`.                                                              |
| GET / PATCH | `…/leads/{id}`     | Read a signal's `lead_id` · update `{status?, owner_email?}`.                             |
| GET         | `…/companies/{id}` | Firmographics of a `company_id`; fields `null` until enriched. No partner company search. |

## Account mapping

| Method | Path                              | Notes                                                             |
| ------ | --------------------------------- | ----------------------------------------------------------------- |
| GET    | `…/company-mappings`              | `company_ids?`, `persona_id?`, `latest?`; each item has `status`. |
| GET    | `…/company-mappings/{mapping_id}` | One mapping with its people.                                      |

`mapping_id` and `company_id` are different id spaces.

## Contents & content requests

| Method | Path                       | Notes                                                                                                    |
| ------ | -------------------------- | -------------------------------------------------------------------------------------------------------- |
| POST   | `…/contents/query`         | Cursor; `content_type[]`, dates, company filters, `lead_id`, `response_format`. Payload under `details`. |
| GET    | `…/contents/{id}`          | One item.                                                                                                |
| POST   | `…/content-requests/query` | Cursor; `type`, `status[]` (completed included).                                                         |
| GET    | `…/content-requests/{id}`  | `status`, `label`, `persona_ids`.                                                                        |

## Status & usage

| Method | Path                | Notes                                                                                          |
| ------ | ------------------- | ---------------------------------------------------------------------------------------------- |
| GET    | `…/setup-state`     | Checklist (`persona`, `accounts`, `signal_setups`, `contents`) with `recommended_next_route`.  |
| GET    | `…/requests-status` | Everything in flight.                                                                          |
| GET    | `…/usage`           | The workspace's billing events, cursor in the query string; each carries its invoice `status`. |

## Still v2 only

| Method | Path                                                     | Purpose                                                   |
| ------ | -------------------------------------------------------- | --------------------------------------------------------- |
| GET    | `/api/v2/partner/me`                                     | Auth check → `data.{id,name,slug}`.                       |
| GET    | `/api/v2/partner/workspaces/{id}/ping`                   | Key valid + workspace active.                             |
| *      | `/api/v2/partner/workspaces/{id}/top-account-list/*`     | Legacy target account list (v3: customer/prospect lists). |
| POST   | `/api/v2/partner/workspaces/{id}/enrich-company-mapping` | Paid one-company mapping launch.                          |
| GET    | `/api/v2/partner/usage`                                  | Usage across all workspaces (v3: per-workspace `…/usage`). |

`/api/v2/partner/…/agents` answers with `Deprecation`, `Sunset: Thu, 01 Apr 2027 00:00:00 GMT` and
`Link: rel="deprecation"` headers: migrate to
signal setups.

## Enums

- **List type → kind**: `competitor` | `partner` | `customer` | `prospect` → `company` ·
  `influencer` | `champion` | `employee` → `profile`.
- **Signal setup `status`**: `enabled`, `disabled`. **`schedule`**: `"weekly"` or `null`.
- **Template `status`**: `live`, `coming_soon`. **`scopes`**: `discovery`, `account_monitoring`.
- **Run / content-request `status`**: `pending`, `in_progress`, `completed`, `failed`.
- **Account monitoring `status`**: `monitored`, `not_covered`, `monitored_elsewhere`, `not_monitored`.
- **`signal_buckets`**: `["essential"]` or `null`.
- **headcounts**: `1-10`, `11-50`, `51-200`, `201-500`, `501-1,000`, `1,001-5,000`,
  `5,001-10,000`, `10,001+` (mind the commas).
- **seniorities**: `owner`, `founder`, `c_suite`, `partner`, `vp`, `head`, `director`, `manager`,
  `senior`, `entry`, `intern`.
- On a refused value, the `422` names the field: trust the validator over any doc page.
