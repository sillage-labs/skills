# Tool map — the MCP v2 tools, grouped

Every `sillage_v2_*` tool you use to set up or edit a workspace, with its key params and its write
class. **Write class is the thing to get right:**

- **READ** — safe, no change.
- **CREATE** — makes a new object.
- **PATCH** — changes only what you send; everything else is kept.
- **APPEND** — adds to a collection; existing items are kept.
- **DESTRUCTIVE** — removes; not reversible.
- **TRIGGER** — enqueues async work; poll a status tool afterwards.
- **BILLED** — spends credits; state the cost and get a yes first.

Every tool refuses an argument it does not declare (422 naming it) — never pad calls with guessed keys.

## Contents

- [Setup / status](#setup--status)
- [Personas (business units)](#personas-business-units)
- [Lists](#lists)
- [Account monitoring](#account-monitoring)
- [Signal catalogue & setups](#signal-catalogue--setups)
- [Runs & detections](#runs--detections)
- [Accounts, people & content](#accounts-people--content)
- [Prompts & resources](#prompts--resources)

## Setup / status

| Tool                  | Class | Key params | Notes                                                                                                                                                                                                                                                                                                                             |
| --------------------- | ----- | ---------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `get_setup_state`     | READ  | —          | `checklist.{persona, accounts, signal_setups, contents}`, each with `status` (ok/warning/blocked), `message`, and `recommended_next_tool` when there is a next step. `signal_setups.missing_priority_types` = which of `job-change`, `engaged-with-competitors-on-linkedin`, `mentioned-keywords-on-linkedin` aren't set up yet; other templates: `get_signal_templates(tags: ["not_set_up"])`. `persona` reports `job_titles_set` / `locations_set`. |
| `get_requests_status` | READ  | `page?`    | Everything in flight: `data[]` typed `account_mapping`, `key_account_content`… or `signal`, each `pending` / `in_progress` with a plain-language `label`. Empty = nothing running.                                                                                                                                                |

## Personas (business units)

| Tool             | Class  | Key params                                                                                                                                                       | Notes                                                                                                                          |
| ---------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| `get_personas`   | READ   | `page?`, `page_size?`                                                                                                                                            | Every active persona with its `id` and targeting. Empty list = no persona yet.                                                 |
| `create_persona` | CREATE | `job_titles[]`, `locations[]` (both required), `name?`, `excluded_job_titles[]`, `headcounts[]`, `industries[]`, `seniorities[]`, `exclusion_criteria[]`, `tag?` | Always adds a persona. A workspace without business units refuses a second one ("contact Sillage"). Returns its `id`.          |
| `update_persona` | PATCH  | `persona_id` + any field above                                                                                                                                   | Only sent fields change; a list you send replaces that list whole. `job_titles` / `locations`, when sent, still need ≥1 entry. |

Allowed values: `seniorities` = `owner`, `founder`, `c_suite`, `partner`, `vp`, `head`, `director`,
`manager`, `senior`, `entry`, `intern`; `headcounts` = `"1-10"`, `"11-50"`, `"51-200"`, `"201-500"`,
`"501-1,000"`, `"1,001-5,000"`, `"5,001-10,000"`, `"10,001+"` (keep the commas).

## Lists

| Tool                 | Class       | Key params                                                                                 | Notes                                                                                                                                                                                                             |
| -------------------- | ----------- | ------------------------------------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `list_types`         | READ        | —                                                                                          | `type` → `kind`: `competitor`/`partner`/`customer`/`prospect` → company; `influencer`/`champion`/`employee` → profile.                                                                                            |
| `create_list`        | CREATE      | `type`, `name`, `description?`, `persona_ids?`, `signal_buckets?`, `tier?`                 | `type` immutable. `employee` refused (the "All my employees" list is built in). `signal_buckets` and `tier` (1–3) only on customer/prospect lists; `["essential"]` is **BILLED**. Created empty.                  |
| `list_lists`         | READ        | `type?`, `persona_id?`, `include_archived?`, `page?`                                       | Each list: `type`, `kind`, `member_count`, `monitored_count`, `signal_buckets`, `mode`, `persona_ids`.                                                                                                            |
| `get_list`           | READ        | `kind`, `list_id`                                                                          | Includes `is_deletable` / `not_deletable_reason`. A smart list (`mode`) is managed in the app only.                                                                                                               |
| `update_list`        | PATCH       | `kind`, `list_id`, `name?`, `description?`, `signal_buckets?`, `archived?`, `persona_ids?` | `signal_buckets: ["essential"]` monitors every account, now and later (**BILLED**); `null` stops. `persona_ids` replaces the attachment; `[]` = every persona. Archiving follows the delete rules.                |
| `delete_list`        | DESTRUCTIVE | `kind`, `list_id`                                                                          | Refused (409) while a signal setup reads it, when it's the only list of its type and tier for its persona (or the only one attached to no persona), or for "All my employees".                                   |
| `list_members`       | READ        | `kind`, `list_id`, `q?`, `signal_buckets?`, `monitoring_status?`, `persona_id?`, `page?`   | Member `id` = workspace company id (company lists) or lead id (people lists). Customer/prospect members carry `monitoring[]` per persona (`monitored`, `not_covered`, `monitored_elsewhere`, `not_monitored`).    |
| `add_list_members`   | APPEND      | `kind`, `list_id`, `members:[{linkedin_url}\|{linkedin_handle}\|{domain}, company?]`       | ≤100 per call, idempotent. `domain` on company lists only. `company` on champion lists only. Partial success: read `errors[].code` (`multiple_domain_matches`, `domain_not_resolved`, …) — a 200 can add nothing. |
| `remove_list_member` | DESTRUCTIVE | `kind`, `list_id`, `member_id`                                                             | `member_id` from `list_members`. This is how a key account is removed.                                                                                                                                            |

## Account monitoring

| Tool                     | Class         | Key params                                                                            | Notes                                                                                                                                                                                    |
| ------------------------ | ------------- | ------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `get_account_monitoring` | READ          | `persona_id?`                                                                         | `is_billed` (`false` = free, no cost to state), `monitored_count`, `not_covered_count`, `price_per_account_credits`, `monthly_cost_credits`, `renewal_shortfall_credits`, `by_persona[]`. An account is paid once per persona.           |
| `set_account_monitoring` | PATCH, BILLED | `company_ids[]` (≤1,000), `signal_buckets` (`["essential"]` \| `null`), `persona_id?` | Charged right away; stopping doesn't refund the month. Accounts must sit on a list attached to that persona, else they come back in `refused` (still a 200). Whole list → `update_list`. |

## Signal catalogue & setups

| Tool                     | Class       | Key params                                                                                      | Notes                                                                                                                                                                                                                                                                                      |
| ------------------------ | ----------- | ----------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `get_signal_templates`   | READ        | `signal_template_id?`, `persona_id?`, `category?`, `subjects?`, `scopes?`, `tags?`, `name?`     | The catalogue. Only `status: live` can be set up. Each template: stable `key`, environment-specific `id`, `scopes`, `pricing`, `tags` (`popular`, `not_set_up`), and `fields` (`key`, `required`, `options`, `default`, `hint`, `control`, `list_type`, `only_with_discovery`, `applied`). |
| `create_signal_setup`    | CREATE      | `signal_template_id`, `parameters` (required, may be `{}`), `persona_id?`, `name?`, `schedule?` | `parameters.scopes` is required on every template. List templates need `parameters.list_id`. `persona_id` required with several personas (except champion and the two job-offer templates). Persona without a job title + location → 422. `schedule: "weekly"` runs once right away.       |
| `get_signal_setups`      | READ        | `signal_setup_id?`, `persona_id?`, `page?`, `response_format?`                                  | Each setup: `signal_template`, `persona_id`, `parameters` as applied (defaults filled), `schedule`, `status` (`enabled`/`disabled`), `last_detected_at`. Some older signals aren't listed but still count in `get_setup_state`.                                                            |
| `configure_signal_setup` | PATCH       | `signal_setup_id` + one of `name`, `status`, `parameters`, `schedule`                           | `parameters` merged into saved ones; `null` removes a key. `status: "disabled"` pauses, `"enabled"` resumes. `list_id` and persona can't change.                                                                                                                                           |
| `delete_signal_setup`    | DESTRUCTIVE | `signal_setup_id`                                                                               | Stops it, frees its list. Confirm the setup by name with the user first.                                                                                                                                                                                                                   |

## Runs & detections

| Tool                  | Class       | Key params                                                                                                                                           | Notes                                                                                                                                                                                                             |
| --------------------- | ----------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `launch_signal_run`   | TRIGGER     | `signal_setup_id`, `lookback_days?`                                                                                                                  | Returns `data[]`, one run per direction, each with an `id`. `lookback_days` (integer 1–180) only on account-monitoring runs of keyword, job-offer, competitor, partner and influencer signals; refused elsewhere. |
| `get_signal_run`      | READ        | `id` (the run id)                                                                                                                                    | `status`: `pending` → `in_progress` → `completed` \| `failed`. Skipped accounts in `metadata.failed.dropped_account_ids`.                                                                                         |
| `list_signals`        | READ        | `limit?`, `cursor?`, `signal_setup_id?`, `signal_run_id?`, `signal_type?`, `lead_id?`, company filters, `signal_date_from/to`, `detected_at_from/to` | The detection feed, newest first. Cursor pagination: pass `meta.cursor.next` back as `cursor` while `meta.cursor.has_more`.                                                                                       |
| `get_signal`          | READ        | `id`                                                                                                                                                 | One detection, payload in `details`.                                                                                                                                                                              |
| `delete_signal`       | DESTRUCTIVE | `id`                                                                                                                                                 | Removes one detection. Rarely needed.                                                                                                                                                                             |
| `get_signal_playbook` | READ        | `signal_template_key?`                                                                                                                               | What a signal means and who to contact. Read the setup's template `key` via `signal_setup_id`. Same content as `sillage://playbooks`.                                                                             |

## Accounts, people & content

| Tool                    | Class | Key params                                                                                     | Notes                                                                                                                                    |
| ----------------------- | ----- | ---------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------- |
| `list_company_mappings` | READ  | `company_ids?`, `persona_id?`, `latest?`, `page?`                                              | One mapping per persona per company, a new one each re-map. Current mapping of a company: `company_ids` + `persona_id` + `latest: true`. |
| `get_company_mapping`   | READ  | `mapping_id`                                                                                   | The mapped people (each `id` is a lead id).                                                                                              |
| `get_company`           | READ  | `company_id`                                                                                   | Workspace company id (same as list member / lead / mapping). Enrichment fields are null until enriched.                                  |
| `get_lead`              | READ  | `lead_id`                                                                                      | The person behind a signal or a people-list member: profile, `experiences`, current `company`.                                           |
| `get_contents`          | READ  | `id?`, `limit?`, `cursor?`, company filters, `content_type?`, `lead_id?`, `created_at_from/to` | The corpus. `normalized` (default) or `concise`. 403 = plan has no content for key accounts.                                             |
| `get_content_requests`  | READ  | `id?`, `limit?`, `cursor?`, `status?`, `type?`, company filters                                | The underlying jobs, with `failed_reason`. No `status` filter = every status, completed included.                                        |

## Prompts & resources

- **Prompts:** `sillage_v2_onboarding` (guided setup), `sillage_v2_targeting_interview` (structured
  targeting interview), `sillage_v2_list_signal_setup` (set up a signal that watches a list).
- **Resources:** `sillage://setup-state` (same checklist as `get_setup_state`), `sillage://playbooks`.
