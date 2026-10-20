---
name: sillage-manage-workspace
description: >
  Stands up a new Sillage workspace and safely edits an existing one through the MCP — the write and
  edit engine behind targeting. Runs the setup loop (persona → key-account lists → monitoring →
  signal setups from the catalogue → runs) and the edit loop (read current state → diff → patch),
  respecting the write rules that make each change safe. Use when applying confirmed targeting,
  changing a live workspace, or fixing a half-configured one. Use when you hear: "apply my
  confirmed targeting", "onboard this account", "edit my persona", "add a business unit", "update my targeting",
  "add a competitor", "add target accounts", "change my keywords", "pause this signal", "re-run my
  signals", "fix my workspace", "reconcile my setup".
metadata:
  owner: pf@getsillage.com
  version: 2.0.0
  model-tier: sonnet
  provider: Sillage MCP v2 (https://api.getsillage.com/api/mcp/v2)
  pairs-with: [sillage-onboarding, sillage-help]
---

# Manage Workspace — set up and edit a Sillage workspace through the MCP

This is the **hands that write**. `sillage-onboarding` decides _what_ the targeting should be (interview +
expansion); this skill puts it into the workspace and changes it later without breaking it. If the
model of the product is unclear, read `sillage-help` first — this skill assumes you know what a
persona (business unit), a list, a key account, monitoring, a signal template, and a signal setup are.

There are two jobs, one skill: **set up** a fresh workspace, and **edit** a live one. Editing is the
harder half, because some choices are frozen once made — get that wrong and you have to rebuild.

## Why this skill exists (and is not the MCP server prompt)

The MCP already has the tools and its server instructions script a happy-path onboarding sequence. What
it does **not** carry is the **edit/reconcile loop** and the **write-safety rules**: which fields are
frozen, what costs credits, what is merged vs replaced. Those are this skill's job. Everything here
operates on **one already-connected workspace** (via `sk_live_` or OAuth) — the MCP can't create a
workspace, only run and edit one. Tools are named `sillage_v2_<name>`; this skill drops the prefix.

## When NOT to use

- You haven't decided the targeting yet → run `sillage-onboarding` first to shape a sharp persona and
  pick signals, then come here to write them.
- You just want to read results ("show me this week's content", "list my signals") → call the read
  tool directly; don't run a setup loop.
- You want to understand the product, not change it → `sillage-help`.

## Non-negotiables (read before any write)

Full detail in `reference/write-semantics.md`.

1. **Persona edits are patches; new personas are deliberate.** `update_persona` changes only the
   fields you send (a list you send replaces that list whole). `create_persona` always adds a new
   business unit — only when the user asks for another one, never to "replace" the current one.
2. **Only the default lists exist.** Every workspace starts with Tier 1/2/3 `prospect` lists (no
   persona, never deletable) and "All my employees". Any other list a signal watches needs an
   existing `list_id` (`list_lists`, or `create_list` + `add_list_members`).
3. **A signal setup's list and persona are frozen.** `configure_signal_setup` merges answers, but
   refuses a new `list_id` or `persona_id`. To change either: create the new setup, then delete the old.
4. **Monitoring and discovery cost credits.** Monitoring an account (`signal_buckets: ["essential"]`)
   is charged per account, per persona, per month, right away. A `weekly` schedule with `discovery`
   in `scopes` spends credits on its first run immediately. State the cost and get a yes first
   (`get_account_monitoring` → `is_billed: false` means monitoring is free: no cost to state).
5. **Confirm before writing, poll before reading.** Read back what you're about to change and get a
   yes. After a write that enqueues work, poll `get_requests_status` / `get_signal_run` before you
   read results. A `403` means the feature isn't in the workspace's plan — surface it, don't retry.

## Setup — standing up a workspace

Drive it off `get_setup_state`: each `checklist` dimension (`persona`, `accounts`, `signal_setups`,
`contents`) has a `status` and, when there is a next step, a `recommended_next_tool` — act through it.

1. `get_setup_state` — see what's missing.
2. **Persona** → `create_persona` (or `update_persona` if one exists) with the confirmed ICP;
   `job_titles` and `locations` (with at least one country) are required.
3. **Key accounts** → `list_lists`, then `add_list_members` to a Tier 1/2/3 `prospect` list (LinkedIn
   URL/handle, domain fallback, ≤100 per call). Adding them starts account mapping. `create_list`
   only for another type (competitor, partner, champion, customer…) or an extra list (`persona_ids`
   with several personas). Read `errors` in the response: a 200 can add nothing.
4. **Monitoring** → after stating the cost (`get_account_monitoring`), `update_list` with
   `signal_buckets: ["essential"]` for the whole list, or `set_account_monitoring` per account.
   Monitoring pays for the mapping and turns on content and signals for those accounts.
5. **Signal setups** → `get_signal_templates` (pass `persona_id`), pick `live` templates.
   `checklist.signal_setups.missing_priority_types` lists which of the 3 priority keys
   (`job-change`, `engaged-with-competitors-on-linkedin`, `mentioned-keywords-on-linkedin`) aren't
   set up yet; a priority key absent from it already runs. For any other template, check
   `get_signal_templates(tags: ["not_set_up"])`. The same template can still be set up again for
   another persona or keyword set. For list templates, make the list first (rule 2). Then
   `create_signal_setup` per signal.
6. **Run** → `launch_signal_run(signal_setup_id)`, poll `get_signal_run` to `completed` / `failed`,
   then read `list_signals` / `get_contents`.

Re-run `get_setup_state` at the end: every dimension should read `ok`, except `contents`, which
reads `blocked` when the plan has no content. Full tool list in
`reference/tool-map.md`.

## Edit — changing a live workspace

Never write blind into a live workspace. Snapshot, diff, then patch the one axis that's off.

1. **Snapshot the current state:**
   ```
   get_setup_state()            # the checklist
   get_personas()               # every business unit and its targeting
   list_lists()                 # lists, member/monitored counts, persona_ids
   get_signal_setups()          # setups, their parameters, status, last_detected_at
   get_account_monitoring()     # what is paid for, what is not covered
   ```
2. **Diff each axis** — persona, key accounts, monitoring, lists, signal setups — and name each as
   **match / mismatch / missing**.
3. **Patch only what's off**, with the right write for that axis (`update_persona`, `add_list_members`
   / `remove_list_member`, `update_list` / `set_account_monitoring`, `configure_signal_setup`, or
   recreate-then-delete for a frozen field). Then launch a run if the change should surface results.

The per-axis table and the recipes (widen the persona, add a business unit, add a competitor, tune
keywords, pause a signal, watch another list) are in `reference/editing-playbook.md`.

## Troubleshoot — few or zero signals

A run that completes with nothing is usually **not** a lookback problem. Stop at the first hit:

1. **Still preparing?** `get_requests_status` — anything listed for those accounts is not ready yet.
   Explain the wait instead of asserting absence.
2. **Monitoring / coverage.** Is the account monitored for this persona (`list_members` →
   `monitoring[].status`; `not_covered` = balance too low to renew)? Are its people mapped
   (`list_company_mappings` with `company_ids`, `persona_id`, `latest: true`)? No mapped people → any
   people-dependent signal finds nobody.
3. **Persona.** A persona with no job titles or no country can't be mapped — fix it with
   `update_persona`.
4. **Setup scope.** Keywords too narrow or over-quoted, list empty, wrong `scopes`, or the wrong
   template for the signal. Check `parameters` via `get_signal_setups`.
5. **Did the run finish?** `get_signal_run` must be `completed`; results come from `list_signals`
   (filter by `signal_setup_id`), not the run.
6. **Account is simply quiet.** Check `get_contents` filtered to the company. An inactive account
   won't produce signals; that's the account, not the setup.
7. **Only then, lookback.** `lookback_days` (1–180) applies only to account-monitoring runs of keyword,
   job-offer, competitor, partner and influencer signals — reach for it last.

## Close with a summary

After a setup or an edit, report plainly what changed and what's now running:

```
## What I changed
<persona / lists / monitoring / signal setups — before → after, in plain language>

## What's running now
<setups enabled · runs launched · accounts monitored and their monthly cost>

## Waiting on
<anything still being prepared, and when to re-check>
```
