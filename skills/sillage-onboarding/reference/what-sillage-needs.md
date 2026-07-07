# What Sillage needs — exact fields & allowed values

The structured target you fill during the interview. These are the real MCP v2 inputs; send valid
values or the call is rejected.

## Persona — `sillage_v2_upsert_persona`

PUT semantics: **every call fully replaces the previous persona.** To preserve fields, read first
with `sillage_v2_get_persona`, then send the complete object. All fields optional; omit one to
leave it unconstrained.

| Field               | Type     | Notes                                                                                                                                                                                                                                              |
| ------------------- | -------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `job_title`         | string[] | Titles to **include**. Send the expanded variant set, not one title.                                                                                                                                                                               |
| `exclude_job_title` | string[] | Titles to **exclude** — filters noise from a broad include match.                                                                                                                                                                                  |
| `seniority`         | string[] | Allowed values **only**: `owner`, `founder`, `c_suite`, `partner`, `vp`, `head`, `director`, `manager`, `senior`, `entry`, `intern`.                                                                                                               |
| `headcount`         | string[] | Allowed values **only** (mind the thousands commas): `"1-10"`, `"11-50"`, `"51-200"`, `"201-500"`, `"501-1,000"`, `"1,001-5,000"`, `"5,001-10,000"`, `"10,001+"`. Comma-less variants (`"501-1000"`) or invented buckets (`"5000+"`) are rejected. |
| `industry`          | string[] | e.g. `"SaaS"`, `"FinTech"`, `"E-commerce"`.                                                                                                                                                                                                        |
| `location`          | string[] | Countries or cities, e.g. `"France"`, `"Germany"`, `"Paris"`.                                                                                                                                                                                      |
| `additional_info`   | string   | Free-form ICP criteria that don't fit the structured fields — e.g. "B2B only, must already have an in-house SDR team". Put real qualifying logic here, not fluff.                                                                                  |

## Agents — `sillage_v2_create_agent`

Required: `name`, `type`. Type is one of `keyword_detection`, `job_posting_keyword_detection`,
`job_update`, `competitor`, `partner`, `customer`, `influencer`, `champion`.

- **`keyword_detection`** — monitors LinkedIn posts. Requires `tracking_keywords` (string[]).
  Optional: `max_posts_to_scrape` (int), `start_date` (ISO date, ignores older posts). Quoting rule
  for keywords lives in `expansion-playbook.md`.
- **`job_posting_keyword_detection`** — monitors the **job postings your tracked companies publish**
  (title and description) for your keywords. Same parameters as `keyword_detection`: requires
  `tracking_keywords`, optional `max_posts_to_scrape`, `start_date`. The signal is hiring intent —
  keywords here should be the roles/stacks a buying company hires for, not the pain language of posts.
- **`job_update`** — detects job changes and promotions (`newJob`, `recentlyPromoted`) among the
  workspace's **mapped contacts**. Workspace-wide; takes **no parameters at creation** — `name` and
  `type` only.
- **`competitor` / `partner` / `customer`** — company watchlist agents. A watchlist of the matching
  type is implicitly created and bound, unless you pass an existing `watchlist_id` (its type must
  equal the agent type).
- **`influencer` / `champion`** — profile watchlist agents, same binding behavior.

Returned agent is created enabled and structurally complete — verify `id` and `enabled: true`.

## Watchlists — `sillage_v2_create_watchlist`

Only needed if you want to build and populate a list _before_ binding it to an agent (otherwise
`create_agent` makes one for you). Required: `type` (`competitor`/`partner`/`customer` → company
list; `influencer`/`champion` → profile list) and `title`. Then add entities with
`sillage_v2_add_watchlist_entities`. Type is immutable once set.

## Where the persona actually bites

Use this to explain _why_ a field matters when the customer resists giving it:

- `job_title` + `seniority` + `exclude_job_title` decide **who counts as a qualified lead** when an
  agent fires. Too broad → noisy detections; too narrow → missed buyers.
- `headcount` + `industry` + `location` decide **which companies qualify**. A signal from an
  out-of-ICP company gets down-rated, not surfaced.
- `additional_info` carries the qualifying logic the structured fields can't express, and feeds
  **content generation** — the persona is the context the generated emails and messages are
  written against. A thin persona produces generic content.
