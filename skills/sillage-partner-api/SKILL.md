---
name: sillage-partner-api
description: >
  Drives the Sillage Partner API — provision and operate Sillage workspaces for your end customers
  with a partner master key (mk_live_): create workspaces, set personas, add top accounts, manage
  watchlists and agents, launch signal runs, and read back mappings, signals, and content. Explains
  the Sillage model (persona → top accounts → account mapping → content → agents → signals) from the
  partner's seat. Use when integrating Sillage as a partner platform, operating workspaces on behalf
  of customers, or calling any /api/v2/partner endpoint. Use when you hear: "partner API", "master
  key", "mk_live", "provision a workspace", "operate workspaces for my customers".
metadata:
  owner: pf@getsillage.com
  version: 1.0.0
  model-tier: sonnet
  provider: Sillage Partner API (https://api.getsillage.com/api/v2/partner)
  pairs-with:
    [sillage-help, sillage-api, sillage-onboarding, sillage-manage-workspace]
---

# Sillage Partner API — operate customer workspaces with a master key

This skill is for **partner platforms** that embed Sillage: one `mk_live_` master key, many
workspaces, each owned by one of your end customers. You provision the workspace, seed its targeting,
and read back the signals and content — all over REST, on their behalf.

It is the partner counterpart of [`sillage-api`](../sillage-api) (which drives **one** workspace as
the customer, with an `sk_live_` key). The concepts are identical; only the auth and the addressing
change. Official reference: <https://www.getsillage.com/partner-doc> · live OpenAPI:
`https://api.getsillage.com/api/v2/partner/docs` (Swagger) and `/docs/spec` (raw JSON).

## How Sillage works — the model you are provisioning

Everything in Sillage builds toward one thing: a **signal** — a reason to reach out to an account
right now. Every concept belongs to one of three layers:

```
SETUP    what you configure          Persona + Top Account List (+ Watchlists, Agents)
CONTENT  what Sillage collects       company posts, job postings, decision-makers' posts & comments
SIGNAL   what you get back           a match between an agent's setup and a piece of content
```

The chain, in order — each stage feeds the next, and a weak early stage starves everything after it:

1. **Persona** — who your customer sells to: job titles, seniority, location, industry, headcount.
2. **Top Account List (TAL)** — the companies they want to win. This list is the universe Sillage
   works on; mapping, listening, and detection happen **only** on these accounts.
3. **Account mapping** — for each top account, Sillage resolves the real company, then finds the
   **key decision makers (KDM)** inside it who match the persona. Runs in the background; everything
   downstream depends on it (an unmapped account produces no people-level signals).
4. **Content collection** — for every mapped account, Sillage listens at two levels: **company
   level** (the account's own posts and job postings) and **people level** (posts and comments from
   the mapped decision makers).
5. **Agents** — saved detection rules, each watching the content for one kind of event: keyword
   mentions, job changes, keywords in job postings, or interactions with a **watchlist** (a list you
   supply of competitors, customers, partners, influencers, or champions).
6. **Signal runs → signals** — an agent finds nothing until you **launch a run**. A completed run
   writes **signals** ("Salesforce's Head of Sales posted about sales efficiency on July 1st"), each
   linked to the underlying content, the company, and — when a person is involved — the lead.

Deeper product model and glossary: [`sillage-help`](../sillage-help) and
<https://www.getsillage.com/docs/getting-started>.

## The eight things that trip up every partner integration

1. **The key alone is the identity.** `Authorization: Bearer $SILLAGE_API_KEY` (`mk_live_…`) on
   `https://api.getsillage.com/api/v2/partner/*`. The `{partner_workspace_id}` in every workspace
   path is the opaque id **you chose** at creation — not a numeric id. All numeric ids in responses
   are environment-specific; never store them across environments. Auth check: `GET /api/v2/partner/me`.
2. **Activate before you seed.** Enrichment and content collection require the workspace to be
   provisioned on Sillage's side (credits + features). A `402` problem document means no credits, a
   `403` means the feature is not enabled — contact Sillage rather than retrying. Add top accounts
   **after** the workspace is fully activated: accounts ingested before activation can miss their
   company-level content (remove + re-add them once active).
3. **Persona is replace-whole, and `location` is required in practice.** `PUT /persona` replaces
   the entire persona: omitted fields become unset and unknown fields are silently dropped —
   always `GET`, merge, `PUT` the full object, and read it back. Account mapping needs
   `location` (country names) to scope its search; a persona without it makes enrichment requests
   fail instead of producing profiles.
4. **Everything that matters is asynchronous.** Adding accounts, enriching a company, launching a
   run — all return `202 Accepted`. Poll to a terminal state before reading results: mapping and
   collection progress via `GET /content-requests` (⚠️ its default filter **hides `completed`** —
   pass `stage=completed` explicitly), runs via `GET /signal-runs/{id}`.
5. **Agents never run on their own.** Creating an agent (any of the 8 types) starts nothing:
   trigger it with `POST /signal-runs` — which works for **every** agent type and returns an
   **array** of runs (watchlist agents start two: inbound + outbound). Judge a run by its polled
   stage and the signals it wrote, not by the agent's `last_run` field.
6. **Listing signals and contents are POST endpoints.** `POST /signals/query` (cursor pagination —
   follow `meta.next_cursor`; `page`/`page_size` are ignored; default window is the last **90
   days**, max 180) and `POST /contents/query` (page pagination). If an older integration calls
   `GET /signals` or `GET /contents`, migrate it — those routes are gone.
7. **Keyword matching is semantic, not exact-phrase.** Quotes don't force literal matches, and
   near-synonyms multiply signal counts without finding new content. Use one keyword per concept.
8. **The API's own error messages are the source of truth for enums.** A `400`/`422` problem
   document lists the accepted values for `type`, `content_type`, `stage`, etc. When a doc page and
   the validator disagree, trust the validator.

## Provision a workspace — the order matters

Persona before accounts (mapping reads it), accounts before agents (agents read the content):

1. **Create** — `POST /workspaces` `{partner_workspace_id, end_customer_email, name, domain}` →
   confirm with `GET …/ping`.
2. **Persona** — `PUT …/persona` with the full object, including `location`.
3. **Watchlists** (if the customer tracks competitors/customers/partners/influencers/champions) —
   `POST …/watchlists` `{type, title}` → `POST …/watchlists/{kind}/{id}/entities` (≤100 per call,
   LinkedIn URL/handle preferred; the response is partial-success — always check its `errors` array).
4. **Top accounts** — `POST …/top-account-list/accounts` `{accounts:[{domain}|{linkedin_url}]}` →
   poll `GET …/top-account-list/status` to `completed` → check `GET …/top-account-list/accounts/not-found`
   **and** verify the accounts you care about are actually present (a mismatch can be silent) —
   re-add misses by `linkedin_url`.
5. **Agents** — `POST …/agents` `{name, type, parameters}`; watchlist-type agents bind the
   `watchlist_id` you pass, or auto-create an empty list.
6. **Run** — `POST …/signal-runs` `{agent_id, signal_key: "keyword", parameters: {}}` → poll each
   returned `signal_request_id` to `completed` / `completed_partial`.

## Read what Sillage produced

- **Decision makers (KDM)** — `GET …/company-mappings` (list; each has `status`
  `in_progress`/`complete`) → `GET …/company-mappings/{mapping_id}` for the enriched profiles
  (position, LinkedIn, email, phone when found). One extra profile lookup: `GET …/leads/{id}` from a
  signal's `lead_id`.
- **Signals** — `POST …/signals/query` (+ `/signals/count`, `GET /signals/{id}`). Filter by agent,
  run, company (id, domain, LinkedIn handle/URL), and date windows.
- **Content** — `POST …/contents/query`, filterable by `content_type`: `linkedinPost`,
  `linkedinCompanyPost`, `linkedinJobPosting`, `linkedinComment` (use `linkedinJobPosting` for job
  posts — `jobPosting` is not a valid value). Each item carries the normalized payload in `data`
  (text, link, engagement, author…; absent fields are omitted, not null).
- **Processing status** — `GET …/content-requests` (`type` = `account_mapping` |
  `top_account_content`), the observable state machine behind mapping and collection.

Full routes, parameters, response shapes, enums, and error model:
**[reference/endpoints.md](reference/endpoints.md)**.

## Recent API changes (mid-2026)

Factual changes a partner integration written earlier may need to absorb:

- Signals listing moved to **`POST /signals/query`** (cursor-paginated); `GET /signals` was removed.
- Contents listing moved to **`POST /contents/query`**; `GET /contents` was removed.
- The polling route `GET /api/v2/account-mapping/{request_id}/stage` was **removed** — poll
  `GET …/content-requests/{id}` instead.
- Agent creation accepts **8 types** (including `job_posting_keyword_detection`, matched against the
  job postings of your top accounts), and `POST /signal-runs` fires **any** agent type.
- Signals now include `source_url`, `author`, and `excerpt` when resolvable.
- `GET …/leads/{id}` was added to resolve a signal's `lead_id` into a profile.

## When NOT to use

- Operating **one** workspace as the end customer (an `sk_live_` workspace key) → [`sillage-api`](../sillage-api).
- Deciding what the targeting should be (persona wording, keyword strategy) →
  [`sillage-onboarding`](../sillage-onboarding); product model questions → [`sillage-help`](../sillage-help).
