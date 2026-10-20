---
name: sillage-partner-api
description: >
  Drives the Sillage Partner API v3: provision and operate Sillage workspaces for your end customers
  with a partner master key (mk_live_): create workspaces, set personas, build lists of key
  accounts and competitors, set up signals from the signal template catalogue, launch signal runs,
  and read back signals, leads, mappings and content. Explains the Sillage model (persona → lists →
  signal templates → signal setups → runs → signals) from the partner's seat. Use when integrating
  Sillage as a partner platform, operating workspaces on behalf of customers, or calling any
  /api/v3/partner endpoint. Use when you hear: "partner API", "master key", "mk_live", "provision a
  workspace", "operate workspaces for my customers".
metadata:
  owner: pf@getsillage.com
  version: 2.0.0
  model-tier: sonnet
  provider: Sillage Partner API v3 (https://api.getsillage.com/api/v3/partner)
  pairs-with: [sillage-help, sillage-api, sillage-onboarding, sillage-manage-workspace]
---

# Sillage Partner API — operate customer workspaces with a master key

This skill is for **partner platforms** that embed Sillage: one `mk_live_` master key, many
workspaces, each owned by one of your end customers. You provision the workspace, seed its targeting,
and read back the signals and content — all over REST, on their behalf.

It is the partner counterpart of [`sillage-api`](../sillage-api) (which drives **one** workspace as
the customer, with an `sk_live_` key). The routes are the same, prefixed with
`/api/v3/partner/workspaces/{partner_workspace_id}`, and answer the same way. Official reference:
<https://www.getsillage.com/partner-doc/v3> · live OpenAPI: `https://api.getsillage.com/api/v3/partner/docs`
(Swagger) and `/api/v3/partner/docs/spec` (raw JSON).

## How Sillage works — the model you are provisioning

Everything builds toward a **signal**: a reason to reach out to an account right now.

1. **Persona** (business unit) — who your customer sells to: job titles, locations, seniority,
   industry, headcount. A workspace can hold several; a setup needs `job_titles` and `locations`.
2. **Lists** — `customer` / `prospect` lists are the key accounts (they can be **monitored**,
   paid per account and persona); `competitor`, `partner`, `influencer`, `champion` lists feed the
   signals that watch them; the one `employee` list is created by Sillage.
3. **Account mapping & content** — for accounts added to a company list Sillage finds the decision makers (monitoring pays for it) matching
   the persona and collects their posts, comments and job postings in the background.
4. **Signal templates** — the catalogue (`GET …/signal-templates`). Each template's `setup.fields`
   (JSON Schema) says which `parameters` to send.
5. **Signal setups** — a template set up for one persona with its parameters (and `list_id` when it
   watches a list). Nothing runs until you launch a run or set `schedule: "weekly"`.
6. **Signal runs → signals** — a completed run writes detections, each linked to the content, the
   company and, when a person is involved, the lead.

Deeper product model: [`sillage-help`](../sillage-help).

## The eight things that trip up every partner integration

1. **The key alone is the identity.** `Authorization: Bearer $SILLAGE_MASTER_KEY` (`mk_live_…`).
   `{partner_workspace_id}` is the string **you chose** at creation (URL-encode it); every other id
   in requests and responses is numeric and environment-specific — never store them across
   environments. Auth check: `GET /api/v2/partner/me` (no v3 equivalent).
2. **Everything answers `{ data }`.** Plain lists page with `page`/`page_size` (`meta.pagination`);
   every `POST …/query` search pages by cursor: send `meta.cursor.next` back as `cursor` until
   `meta.cursor.has_more` is `false`. Advice comes in `meta.warnings`.
3. **Validation errors are `422`** RFC 9457 problem documents (`type: about:blank`) with `errors`
   keyed by field path (`parameters.list_id`, `persona_id`). A malformed `partner_workspace_id` is
   a `422` too; an unknown one is a `404`. `402` no credits, `403` the plan doesn't allow it (contact
   Sillage rather than retrying), `409` conflict, `429` wait `Retry-After` seconds.
4. **Setups follow their template.** `POST …/signal-setups` `{signal_template_id, parameters,
name?, schedule?, persona_id?}`. `parameters.list_id` is required when the template's
   `setup.fields.required` lists it (content engagement defaults to "All my employees"), and **no
   list is created for you**. An incomplete persona → `422 Persona incomplete`.
5. **Setups change by `PATCH` merge.** Send only what changes; `null` removes a key; `status:
"disabled"|"enabled"` pauses/resumes. The list and the persona are fixed (`409`): set the signal
   up again, then delete the old setup.
6. **Runs are async and fan out.** `POST …/signal-runs` `{signal_setup_id}` → `202` with an
   **array** of runs (one per direction, plus Discovery when `scopes` has `discovery`, which is paid).
   Poll each `GET …/signal-runs/{id}` to `completed` / `failed`. A second launch while one runs → `409`.
7. **Monitoring costs credits.** `signal_buckets: ["essential"]` on a customer/prospect list (or
   `PUT …/account-monitoring/accounts`) starts paid monitoring right away. Confirm with your customer.
8. **Keyword matching is semantic, not exact-phrase.** Quotes don't force literal matches; near-synonyms
   multiply signal counts without finding new content. One keyword per concept.

## Provision a workspace — the order matters

Paths below are relative to `/api/v3/partner/workspaces/{partner_workspace_id}` (`…`).

1. **Create** — `POST /api/v3/partner/workspaces` `{partner_workspace_id, name, domain,
owner_email}` → `201`. One live workspace per domain (`409` otherwise: add a persona instead).
2. **Persona** — `POST …/personas` with at least `job_titles` and `locations`.
3. **Key accounts** — `POST …/lists` `{type:"prospect"|"customer", name}` →
   `POST …/lists/company/{list_id}/members` `{members:[{linkedin_url}|{domain}]}` (≤100 per call,
   partial success: always check `errors`) → monitor them (paid) with `signal_buckets`.
4. **Other lists** the chosen templates need (competitors, influencers, champions…), same calls.
5. **Signal setups** — `GET …/signal-templates` → read `setup.fields` → `POST …/signal-setups`.
   Field-by-field recipe: [`sillage-api` → recipes](../sillage-api/reference/recipes.md#set-up-a-signal-from-a-template).
6. **Run** — `POST …/signal-runs` `{signal_setup_id}` → poll each returned run.

Check progress at any time with `GET …/setup-state` (a checklist with `recommended_next_route`)
and `GET …/requests-status` (everything in flight).

## Read what Sillage produced

- **Signals** — `POST …/signals/query` (cursor) + `POST …/signals/count`, `GET …/signals/{id}`.
  Filter by `signal_setup_id`, `signal_run_id`, `signal_type`, company (ids, domain, LinkedIn), dates.
- **Leads** — `POST …/leads/query`, `GET …/leads/{id}` from a signal's `lead_id`.
- **Decision makers** — `GET …/company-mappings` → `GET …/company-mappings/{mapping_id}`.
- **Content** — `POST …/contents/query`, `content_type` in snake_case (`linkedin_post`,
  `linkedin_job_posting`…); the payload sits under `details`.
- **Credits spent** — `GET …/usage` (cursor in the query string).

Full routes, bodies and error model: **[reference/endpoints.md](reference/endpoints.md)**.

## Recent API changes (v3, late 2026)

Factual changes an integration written against v2 must absorb:

- **New base path:** `/api/v3/partner/workspaces/{partner_workspace_id}/…`. Every answer is
  enveloped in `{ data }`; searches page by `meta.cursor`.
- **Agents → signal setups.** `/v2/partner/…/agents` sends `Deprecation`, `Sunset: Thu, 01 Apr 2027 00:00:00 GMT` and
  `Link: rel="deprecation"` headers; use `…/signal-templates` + `…/signal-setups`, and launch runs with `signal_setup_id`
  (no `signal_key`). Signals and runs carry `signal_setup_id` instead of `agent_id`.
- **Watchlists → lists.** `…/lists` with `name` (not `title`) and members under `/members`
  `{members:[…]}`; new types `prospect` and `employee`.
- **Top account list → customer/prospect lists + account monitoring.** The v3 surface has no
  top-account-list routes.
- **Persona → personas.** Several per workspace, plural field names (`job_titles`, `locations`,
  `excluded_job_titles`…), `PATCH` instead of replace-whole `PUT`.
- **Workspace create** takes `owner_email` (was `end_customer_email`) and a required `domain`;
  pausing is `PATCH {archived:true}`; `DELETE` is reversible with `POST …/restore`.
- **Validation errors answer `422`** on every v3 route.

## Still v2 only

`GET /api/v2/partner/me`, `GET /api/v2/partner/workspaces/{id}/ping`, the cross-workspace `GET /api/v2/partner/usage` (v3 only has
per-workspace `…/usage`), the top account list
(`…/top-account-list/*`) and the paid mapping launch (`…/enrich-company-mapping`) have no v3 route.

## When NOT to use

- Operating **one** workspace as the end customer (an `sk_live_` workspace key) → [`sillage-api`](../sillage-api).
- Deciding what the targeting should be (persona wording, which templates, keywords) →
  [`sillage-onboarding`](../sillage-onboarding); product model questions → [`sillage-help`](../sillage-help).
