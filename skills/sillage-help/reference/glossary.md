# Glossary — every Sillage term, defined

The vocabulary the tools and the other skills assume. Terms are ordered along the chain (persona →
detections), then the cross-cutting ones.

## Persona / ICP

Your **Ideal Customer Profile**: who you sell to, expressed as `job_title`, `exclude_job_title`,
`seniority`, `headcount`, `industry`, `location`, and free-form `additional_info`. It is the frame
that decides **which people count as qualified leads** and **which companies qualify** — and it's the
context your generated content is written against, so a thin persona produces generic content. One
persona per workspace. It is **replaced whole** on every write (PUT), never patched — read it before
you change it. Exact fields and allowed values live in `sillage-onboarding`'s `reference/what-sillage-needs.md`.

## Target Account List (TAL) / target accounts

The companies you **want to sell to** — the list Sillage enriches and generates content for. You
identify accounts by **domain** (preferred, best match accuracy) or LinkedIn URL. The list is
**additive** (you append accounts; you remove them by id) — there is no "replace the whole list" in
the MCP. Read the real TAL with `read_top_account_list`.

> Not to be confused with the result of `get_top_accounts`, which returns **every company that has any
> activity trace** in the workspace (mapped, or with content, or with a request) — a superset, not
> your target list. For "what am I actually targeting", use `read_top_account_list`.

## Coverage / Company Mapping

**Coverage** is whether Sillage has resolved a company to its **people** — the employees whose LinkedIn
profiles match your persona. That resolution is a **company mapping**, built by `enrich_company`. A
mapping gives you each person's profile: handle, LinkedIn URL, headline, position, location, sometimes
their "about". Mappings are **versioned and historical** — a new one is a new version; removing a
company from the front doesn't erase its past mappings.

- **"No coverage"** — the badge/state meaning no mapping exists for an entity yet, so there are no
  people to track for it. Adding an entity to a watchlist doesn't build coverage by itself — it shows
  no coverage until a company mapping is triggered. Coverage comes from `enrich_company` on a
  **domain**.
- Coverage does **not** auto-refresh when you change the persona — re-trigger `enrich_company` to pull
  people against the new ICP.

## Watchlist

A named list of entities you **monitor** — separate from the TAL. It has a **type** (immutable once
set) which fixes its **kind**:

| Type         | Kind    | You're monitoring…                                    |
| ------------ | ------- | ----------------------------------------------------- |
| `customer`   | company | your existing customers (expansion / churn awareness) |
| `competitor` | company | rival companies                                       |
| `partner`    | company | partners                                              |
| `influencer` | profile | creators your buyers follow                           |
| `champion`   | profile | your advocates / internal champions                   |

Company watchlists take company entities; profile watchlists take people. A watchlist is bound to at
most one agent to give that agent its scope; binding is swappable.

## Agent

A **saved search** — one detection rule. Types: `keyword_detection`, `job_update`, and the five
watchlist types above. An agent is created **enabled** and can be paused/renamed/reconfigured. It does
nothing on its own until you **launch a signal run**. For watchlist types, creating the agent will
create and bind a matching watchlist for you unless you hand it an existing `watchlist_id`.

These **seven** types are the whole set — there is no `job_posting` or "engagement" agent, and for a
signal with no agent (hiring, research, CRM) you can't create one. Edit an agent **in place**
(`configure_agent`, `bind_agent_watchlist`); deleting and recreating mints a new agent id — and, for
watchlist types, a new auto-spawned watchlist id — orphaning the list you populated.

## Signal Run

One **execution** of an agent — the agent going out to look. Keyword and job agents produce **one**
run; watchlist agents produce **two** (see inbound/outbound). A run fans out to one job per target
account and reports a stage (`running` → `completed` / `completed_partial` / `failed`) and per-job
counts (matched, written, deduped, errored).

## Inbound / Outbound

Two directions of a watchlist run:

- **Inbound** — a **target** engages the watched entity's content (someone you're prospecting comments
  on your competitor's post).
- **Outbound** — the **watched entity** engages a target's content.

## Signal Detection

A single detected event — "this person did this thing" — keyed by agent and signal type. The event
feed. See `signal-taxonomy.md` for every type.

## Workspace Content

The raw collected corpus behind detections: LinkedIn posts, company posts, comments, reactions, job
postings, articles. Same substrate as detections, read through a different lens — filterable by
company, person, and content type. See `signal-taxonomy.md` for the content types.

Read unfiltered it's a **superset** — largely person-authored posts and content from companies beyond
your target list — so filter by `company_id` / `company_domain` for a target account's material, and
expect its share of the whole corpus to be small. Company-page posts often carry an empty body.

## Setup-state flags

`persona_set`, `list_uploaded`, `ingestion_complete`, `has_contents` — the four booleans from
`get_setup_state` that say whether a workspace is configured and live. All true = ready.

## Keys — `sk_live_` vs `mk_live_`

- **`sk_live_`** — a **workspace key**. It scopes the MCP to one workspace: read and edit that
  workspace's config and results. This is what the public MCP uses under the hood — an OAuth
  connection (the default; no key to paste) resolves to the workspace's `sk_live_` key. It **cannot
  create a workspace**.
- **`mk_live_`** — a **partner/master key** used by Sillage's own provisioning to create and operate
  workspaces on a customer's behalf. It is not part of the workspace MCP and isn't needed to run or
  edit an existing workspace.
