# What Sillage needs — exact fields & allowed values

The structured target you fill during the interview. These are the real MCP v2 inputs; send valid
values or the call is rejected (an undeclared key is refused with a 422 naming it).

## Persona — `sillage_v2_create_persona` / `sillage_v2_update_persona`

A persona is one **business unit**: the targeting of one team. `create_persona` always adds one (a
workspace without business units refuses a second — "contact Sillage"); `update_persona(persona_id,
…)` changes only the fields you send, and a list you send replaces that list whole. Read current
personas with `sillage_v2_get_personas`.

| Field                 | Type     | Notes                                                                                                                                                                                                                                              |
| --------------------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `job_titles`          | string[] | **Required**, ≥1. Titles to **include**. Send the expanded variant set, not one title.                                                                                                                                                             |
| `locations`           | string[] | **Required**, ≥1, with **at least one country**. Countries or cities, e.g. `"France"`, `"Paris"`. A city-only list saves but account mapping won't run.                                                                                            |
| `excluded_job_titles` | string[] | People whose LinkedIn headline contains one of these words won't become leads — filters noise from a broad match.                                                                                                                                  |
| `seniorities`         | string[] | Allowed values **only**: `owner`, `founder`, `c_suite`, `partner`, `vp`, `head`, `director`, `manager`, `senior`, `entry`, `intern`.                                                                                                               |
| `headcounts`          | string[] | Allowed values **only** (mind the thousands commas): `"1-10"`, `"11-50"`, `"51-200"`, `"201-500"`, `"501-1,000"`, `"1,001-5,000"`, `"5,001-10,000"`, `"10,001+"`. Comma-less variants (`"501-1000"`) or invented buckets (`"5000+"`) are rejected. |
| `industries`          | string[] | e.g. `"SaaS"`, `"FinTech"`, `"E-commerce"`.                                                                                                                                                                                                        |
| `exclusion_criteria`  | string[] | Sentences or keywords for who to leave out — matching leads and companies score lower. ≤50 entries of ≤500 characters; `[]` clears. Put real disqualifying logic here, e.g. "Agencies", "No in-house sales team".                                  |
| `name`                | string   | Shown in the app. Defaults to "<domain> persona". Name each business unit clearly ("Enterprise EMEA").                                                                                                                                             |
| `tag`                 | string   | Short label for your own systems; no effect on targeting; unique per workspace.                                                                                                                                                                    |

Most signal templates are refused (422 "Persona incomplete") until a persona has a job title and a
location — set the persona **before** the signals. Only the champion and job-offer templates need none.

## Lists — `sillage_v2_create_list` + `sillage_v2_add_list_members`

Every workspace starts with Tier 1/2/3 `prospect` lists (no persona, never deletable) and "All my
employees". Any other list a signal watches must be created first.

- `type` (immutable): company lists `competitor`, `partner`, `customer`, `prospect`; people lists
  `influencer`, `champion`. `employee` is the built-in "All my employees" list — never created.
- **Key accounts** live in `customer` / `prospect` lists: `list_lists`, then add them to a tier
  list; `create_list` only for extra lists. With several personas, pass `persona_ids` (none = every
  persona).
- Members: LinkedIn URL or handle (preferred); `domain` as a fallback on company lists only; on
  champion lists, each member may carry `company` (where you know them today). ≤100 per call;
  failures come back in `errors`.

## Signal setups — `sillage_v2_get_signal_templates` → `sillage_v2_create_signal_setup`

The catalogue is data: read it, don't hard-code it. Each `live` template's `fields` say what to
answer. What the interview usually feeds:

- `scopes` — **required on every template**: `account_monitoring` (your monitored key accounts)
  and/or `discovery` (people outside them, priced per signal found). Job-offer templates take
  `account_monitoring` only.
- `keywords` — for the keyword templates (posts, job offers; brand and competitor-brand have
  defaults). Quoting rule in `expansion-playbook.md`.
- `job_titles` — for "Job change" and "Hiring for roles my product serves"; by default
  `use_persona_job_titles: true` already adds the persona's titles, so only add extra ones.
- `list_id` — for the templates that watch a list (`control: list`, its `list_type` says which).
- `persona_id` — required when the workspace has several personas (except champion and job-offer
  templates).

## Where the persona actually bites

Use this to explain _why_ a field matters when the customer resists giving it:

- `job_titles` + `seniorities` + `excluded_job_titles` decide **who gets mapped and who counts as a
  qualified lead** when a signal fires. Too broad → noisy detections; too narrow → missed buyers.
- `locations` + `headcounts` + `industries` decide **which companies and people qualify**. No country
  → no account mapping at all.
- `exclusion_criteria` carries the disqualifying logic the structured fields can't express, and
  down-scores leads and companies that match it.
- The persona is also the context generated content is written against. A thin persona produces
  generic content.
