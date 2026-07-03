# Editing playbook — reconcile a live workspace, one axis at a time

Editing is a read-diff-patch loop, never a blind write. Snapshot the current state, judge each axis,
then apply the single right fix. Read `write-semantics.md` first — it's what keeps each patch safe.

## Step 1 — snapshot

```
get_persona()            # the current ICP
get_agents()             # agents: types, enabled, keywords
list_watchlists()        # lists and how populated they are
read_top_account_list()  # the real target accounts (NOT get_top_accounts)
get_setup_state()        # the four readiness flags
```

## Step 2 — diff each axis

For each axis, call it **match** (leave it), **mismatch** (fix it), or **missing** (add it):

| Axis            | Read with               | Mismatch looks like…                                       | The one fix                                                                               |
| --------------- | ----------------------- | ---------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| Persona         | `get_persona`           | wrong/narrow titles, seniority, industry, geo              | merge into the full object → `upsert_persona`; then re-`enrich_company` affected accounts |
| Target accounts | `read_top_account_list` | missing target companies, or off-ICP ones present          | `add_top_accounts` (append) / `remove_top_accounts` (by id)                               |
| Coverage        | `list_company_mappings` | target companies with no mapped people                     | `enrich_company` by **domain**, poll to `completed`                                       |
| Watchlists      | `list_watchlists`       | a needed list absent or empty                              | `create_watchlist` (right type) + `add_watchlist_entities`                                |
| Agents          | `get_agents`            | needed signal has no agent / agent paused / stale keywords | `create_agent`, or `configure_agent` (keywords/enabled), or `bind_agent_watchlist`        |

## Step 3 — patch, then make it surface

Apply the fix, then re-trigger whatever turns the change into results:

- Persona or coverage change → `enrich_company` the affected accounts (coverage doesn't auto-refresh).
- Keyword/agent change → `launch_signal_run` and poll `get_signal_run`.
- Confirm with `get_setup_state` that the readiness flags are where you expect.

> **Don't delete-and-recreate to "refresh."** Every edit below has an in-place tool — use it. Tearing
> down agents, accounts, or the persona and rebuilding churns ids, orphans auto-spawned watchlists, and
> re-enqueues ingestion for nothing (detail in `write-semantics.md` → "Edit in place").

## Recipes for the common edits

**Widen the persona (surface more decision-makers).** `get_persona` → add the new titles/seniority
into the full arrays → `upsert_persona` → `enrich_company` the target accounts again so their coverage
is rebuilt against the wider ICP. Widening the persona and re-mapping is the lever that surfaces buyers
who were previously invisible — not adding keywords.

**Add a competitor to track.** Is there a `competitor` watchlist? If not, `create_watchlist(type:
competitor)` (or let `create_agent(type: competitor)` make one). `add_watchlist_entities` with the
competitor's LinkedIn URL (a raw domain can bounce on ambiguous companies). To also get the people at
that competitor, `enrich_company` its domain.

**Add / remove target accounts.** `add_top_accounts` (by domain) to append; poll
`get_top_account_list_status`. To drop accounts, read ids from `read_top_account_list` then
`remove_top_accounts(ids)`.

**Tune keywords.** `get_agents(agent_id, detailed)` to see the current set → `configure_agent` with the
new `tracking_keywords`. Quote generic phrases to cut noise, leave niche terms bare (the quoting rule
is in `sillage-onboarding`'s `reference/expansion-playbook.md`). Then `launch_signal_run` to test the new set.

**Pause / resume an agent.** `configure_agent(agent_id, enabled: false)` to pause, `true` to resume.

**Re-point a watchlist agent.** `bind_agent_watchlist(agent_id, watchlist_id)` to swap the list it
watches; both fields null to unlink.

**Fix a half-configured workspace.** Start from `get_setup_state`: the first false flag is the next
job — no persona → set it; no accounts → add them; not ingested → poll (or re-add / re-enrich); no
content → check agents exist and have been run.
