# Glossary — every Sillage term, defined

The vocabulary the tools and the other skills assume, ordered along the chain (business unit →
signals), then the cross-cutting terms. MCP tool names drop the `sillage_v2_` prefix.

**Contents:** Business unit · List · Key accounts · Account monitoring · Signal bucket · Account
mapping · Signal template · Signal setup · Scope · Schedule · Signal run · Inbound / Outbound ·
Signal · Content · Setup-state checklist · Keys

## Business unit / persona / ICP

Your **Ideal Customer Profile**, one per business unit: `job_titles`, `excluded_job_titles`,
`locations`, `headcounts`, `industries`, `seniorities`, `exclusion_criteria`, plus a `name` and a
`tag`. It decides which people count as leads and is the context content is written against. A
workspace can hold **several personas** (business units), when Sillage has enabled it; each is a
numeric `persona_id` (`get_personas`). `job_titles` + `locations` are the two fields that unblock
account mapping and most signal setups (an incomplete persona → 422). `update_persona` changes only
the fields you send; `create_persona` **adds** a business unit — never use it to replace one. Unknown
fields (including the old singular names `job_title`, `location`) are refused.

## List

A named set of companies or people, with a **type** (fixed once set) that fixes its **kind**:

| Type         | Kind    | Used for                                                        |
| ------------ | ------- | --------------------------------------------------------------- |
| `customer`   | company | **key accounts** — your customers                               |
| `prospect`   | company | **key accounts** — who you want to sell to                      |
| `competitor` | company | watched by the competitor signals                               |
| `partner`    | company | watched by the partner signal                                   |
| `influencer` | profile | watched by the influencer signal                                |
| `champion`   | profile | watched by the champion signal (members may carry `company`)    |
| `employee`   | profile | "All my employees" — comes with the workspace, can't be created |

A list is attached to business units through `persona_ids` (empty = every business unit). `mode` is
`manual` or `smart` (a smart list is filled by rules in the app and may remove members you add).
Lists are created and filled through the API (`create_list`, `add_list_members`) — Sillage never
creates one for you.

## Key accounts

Every company on a `customer` or `prospect` list. This is what `account_monitoring` signals look at,
what setup state counts as `accounts`, and what contents are generated for. Older docs call it the
Target Account List (TAL): there is no separate account list any more — add accounts to a customer or
prospect list (`add_list_members`). Each member reports its `monitoring` status per business unit.

## Account monitoring

The paid switch that makes Sillage keep watching a key account: per account, per business unit, per
month, in credits, charged right away. Set it with `signal_buckets: ["essential"]` (or `null` to
stop) on a whole list (`update_list`) or on chosen accounts (`set_account_monitoring`); read the
totals and monthly cost with `get_account_monitoring`. A list attached to several business units is
charged in each. Statuses: `monitored`, `not_covered` (renewal unpaid — top up, retried hourly),
`monitored_elsewhere` (another business unit only), `not_monitored`. Stopping never refunds the
current month. Always state the cost and ask before turning it on.

## Signal bucket

The tier that prices account monitoring: `essential`, `standard`, `advanced` (cumulative — a higher
tier includes the lower ones). A template carries the lowest bucket that includes it. Only
`essential` is in use today. Explanation only: no MCP tool or REST route lists buckets — a template's `pricing` names its bucket.

## Account mapping / coverage

Resolving a company to its **people** — the employees whose profiles match the persona. Adding
companies to a customer, competitor or partner list starts it; on key accounts, monitoring pays for
it. Read it with `list_company_mappings` / `get_company_mapping`; in-flight work with
`get_requests_status`. No mapping ("no coverage") → no decision-makers to track there.

## Signal template

One entry of the **signal catalogue**: a kind of signal Sillage can detect, with a stable `key`, a
`category`, the `scopes` it supports, its `pricing` and the `setup.fields` it asks. Only `live` ones
can be set up. See `signal-taxonomy.md`.

## Signal setup

A template you answered, saved for **one business unit** (`persona_id`) and, when the template watches
a list, **one list** (`list_id`) — both fixed for the setup's life. Its `status` is `enabled` or
`disabled`; `last_detected_at` tells when it last found something. Edit with `configure_signal_setup`
(merge: sent keys replace, `null` removes). To change the list or business unit, create a new setup,
then delete the old one. (Replaces the old "agent"; `/v2/agents` sunsets on 2027-04-01.)

## Scope

Where a setup looks, answered in `parameters.scopes`: `account_monitoring` (your key accounts, covered
by monitoring) and/or `discovery` (everyone else, paid per signal found at the template's price).

## Schedule

`schedule: "weekly"` — the setup runs by itself every week (first run on the next tick). `null` —
manual: it runs only when launched.

## Signal run

One execution of a setup (`launch_signal_run` with its `signal_setup_id`). Status `pending` →
`in_progress` → `completed` / `failed`; poll with `get_signal_run`. List-engagement signals start one
run per direction. Re-running is safe — detections are deduplicated. A disabled setup can't be launched.

## Inbound / Outbound

The two directions of a list-engagement signal: **inbound** — a key-account person engages the watched
entity's content; **outbound** — the watched entity engages a key account's content.

## Signal (detection)

One detected event — "this person did this thing" — with its `signal_type`, `signal_setup_id`, lead,
company and source link. Read with `list_signals` / `get_signal`; interpret with
`get_signal_playbook` (by template `key`).

## Content

The raw corpus behind signals: posts, company posts, comments, reactions, job postings, articles.
Available when the plan includes content for key accounts (else 403 / `contents: blocked`). Unfiltered
it's a superset — filter by company. Company-page posts often carry an empty body.

## Setup-state checklist

`get_setup_state`: four dimensions — `persona`, `accounts`, `signal_setups`, `contents` — each `ok` /
`warning` / `blocked` with a message and a `recommended_next_tool`. All `ok` = live.

## Keys — `sk_live_` vs `mk_live_`

- **`sk_live_`** — a **workspace key**: one workspace, read and edit. The MCP's OAuth connection
  resolves to it. It cannot create a workspace.
- **`mk_live_`** — a **partner/master key** to create and operate many customer workspaces
  (`sillage-partner-api`). Not part of the workspace MCP.
