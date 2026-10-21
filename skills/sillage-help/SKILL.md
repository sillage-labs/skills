---
name: sillage-help
description: >
  Explains the whole logic of a Sillage workspace and the vocabulary behind it: how business units
  (personas), key-account lists, account monitoring, the signal catalogue, signal setups, runs,
  detections and content fit together, and what every term means. The reference the other skills
  assume you already have. Use when the model of the product is unclear, before setting up or
  editing a workspace, or when a result doesn't make sense. Use when you hear: "how does Sillage
  work", "what's a signal setup", "what does monitoring cost", "glossary".
metadata:
  owner: pf@getsillage.com
  version: 2.0.0
  model-tier: sonnet
  provider: Sillage MCP v2 (https://api.getsillage.com/api/mcp/v2), REST API v3
  pairs-with: [sillage-onboarding, sillage-manage-workspace]
---

# How Sillage works — the logic and the glossary

Sillage watches the public web (mostly LinkedIn) for the moments that mean a company or a person is
about to need what you sell, and turns those moments into leads and ready-to-use content. This skill
is the map of _how the pieces connect_ and _what each word means_. It doesn't do anything — it makes
the doing (with `sillage-onboarding` and `sillage-manage-workspace`) make sense.

## The one thing to understand first

Everything hangs off **one chain**. A weak early stage quietly starves every stage after it:

```
Business unit (persona)       ── who you sell to: job titles, locations, industries…
      │
Key accounts                  ── companies on your customer / prospect LISTS
      │
      ├─ account monitoring ─►  paid per account, per business unit, per month
      │                         → account mapping (the people at each account who match the persona;
      │                           adding companies to a customer / competitor / partner list starts it too)
      │
Signal catalogue (templates)  ── every kind of signal Sillage can detect, with its questions and price
      │
Signal setups                 ── a template you answered, for one business unit (+ a list when it watches one)
      │
      └─ signal run  ─►  weekly on schedule, or launched by hand
                             │
                             ▼
              Signals (detections)  +  Contents   ── what it found, and the raw posts/articles behind it
```

So: **persona → key accounts → monitoring/mapping → setups → runs → signals + content.** No
persona job title or location → most setups are refused. No monitored accounts → account-monitoring
signals have nobody to watch. No run → no signals, however good the setup.

## Templates, setups, runs

- **Signal template** — one entry of the catalogue (`get_signal_templates` / `GET /v3/signal-templates`).
  Its stable `key` (e.g. `job-change`, `mentioned-keywords-on-linkedin`) names it; its `setup.fields`
  say exactly which `parameters` to fill. Only `status: live` templates can be set up — the full list
  is in `reference/signal-taxonomy.md`.
- **Signal setup** — a template you answered (`create_signal_setup`). It belongs to **one business
  unit** (`persona_id`, fixed for life; champion and job-offer setups may have none) and, for templates
  that watch a list (competitors, partners, influencers, champions, your employees), to **one list you
  name** (`list_id`, fixed for life; no list is ever created for you, and the content-engagement
  template defaults to "All my employees"). The same template can be set up several times (e.g. one per keyword
  set). Edits merge (`configure_signal_setup`); to change the list or persona, create a new setup and
  delete the old one.
- **Signal run** — one execution. A setup with `schedule: "weekly"` runs by itself every week; a manual
  one (`schedule: null`) runs only when you call `launch_signal_run`. A run fans out per key account and
  ends `completed` or `failed`; interaction signals start one run per direction (inbound / outbound).

## Where a signal looks — `discovery` vs `account_monitoring`

Every setup answers `scopes`, and it changes both what you find and what you pay:

| Scope                | Looks at                                        | Paid                                              |
| -------------------- | ----------------------------------------------- | ------------------------------------------------- |
| `account_monitoring` | your **key accounts** (customer/prospect lists) | per monitored account, per month — not per signal |
| `discovery`          | everyone **outside** your key accounts          | per signal found, at the template's price         |

Creating or editing a setup charges nothing. Monitoring is switched on per account (or per whole
list), not per setup.

## Signal buckets and account monitoring

A **bucket** (`essential`, `standard`, `advanced`) is the tier that prices account monitoring: a
cost per key account, per month, per business unit. Tiers are cumulative. **Only `essential` exists
in practice today** — every priced template sits in it. Neither the MCP nor the REST API lists the buckets
themselves: `get_signal_templates` shows each template's `pricing` (bucket + credits) and
`get_account_monitoring` the price per account and the monthly cost of what you monitor.

An account's **monitoring status**, per business unit:

| Status                | Means                                                                          |
| --------------------- | ------------------------------------------------------------------------------ |
| `monitored`           | Paid month running (or the workspace isn't billed).                            |
| `not_covered`         | Should be monitored but the renewal couldn't be paid — top up, retried hourly. |
| `monitored_elsewhere` | Monitored in another business unit only.                                       |
| `not_monitored`       | Not monitored here (`covered_until` shows when a stopped month ends).          |

Monitoring costs credits and is charged right away: always tell the user the cost and ask before
turning it on. Stopping never refunds the month already paid.

## The two flows people confuse

- **Account mapping** = resolving a _company_ to its _people_ — the employees whose profiles match the
  persona. Adding companies to a customer, competitor or partner list starts it; on key accounts,
  monitoring pays for it. No mapping → no decision-makers to track there.
- **Content** = the company page's own LinkedIn activity. It exists whether or not anyone is mapped
  (when the plan includes content).

So _"we only get company keywords, never the decision-makers"_ = content flows, mapping is empty. The
fix is monitoring the account (and a persona with job titles + locations), not more keywords.
`sillage-manage-workspace` has the "few or zero signals" checklist.

## When signals come back — lead with the outreach brief

1. **Rank accounts by signal density** — a handful dominate; those are the hot accounts.
2. **Name the person and the signal that qualifies them** — "Luigi, GM — 12 posts about a new site =
   active expansion".
3. **Give a per-account approach angle grounded in that signal.** Read `get_signal_playbook` for the
   signal's template `key` before interpreting it.

Keep config tuning secondary — surface the brief first.

## Is a workspace set up? — the setup-state checklist

`get_setup_state` (`GET /v3/setup-state`) returns a `checklist` of four dimensions, each with a
`status` (`ok` / `warning` / `blocked`), a plain `message` and, when there's a next step,
`recommended_next_tool`:

| Dimension       | Key fields                        | Done when                                   |
| --------------- | --------------------------------- | ------------------------------------------- |
| `persona`       | `job_titles_set`, `locations_set` | both true — they unblock mapping and setups |
| `accounts`      | `count`                           | ≥ 1 account on a customer/prospect list     |
| `signal_setups` | `count`, `missing_priority_types` | the priority template `key`s are set up     |
| `contents`      | `has_contents`                    | content generated (`blocked` = not in plan) |

A key not in `missing_priority_types` already runs — never set it up again. Every dimension `ok` =
live.

## When NOT to use

- You want to _do_ something → `sillage-onboarding` (shape targeting) or `sillage-manage-workspace`
  (write and edit).
- You need exact fields/enums → `sillage-manage-workspace`'s `reference/tool-map.md` or
  `get_signal_templates`; this skill is the mental model, not the API surface.

## Where to go next

- **Glossary** — every term, defined: `reference/glossary.md`.
- **Signal catalogue & content types**: `reference/signal-taxonomy.md`.
