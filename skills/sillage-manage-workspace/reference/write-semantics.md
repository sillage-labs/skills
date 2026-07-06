# Write semantics — how each write behaves, and how not to break things

Reading is safe. Writing is where a workspace gets damaged. This is the exact behavior of every write
class and the safe pattern for each.

## Edit in place — never delete-and-recreate

The most damaging habit is tearing something down to "refresh" it. There is almost never a reason to.

- **Agents:** to change keywords, name, or enabled state, use `configure_agent` — **not**
  `delete_agent` + `create_agent`. Recreating mints a **new agent id**, and for watchlist types it
  auto-spawns a **new watchlist id**, orphaning the list you populated (real churn seen in the wild:
  agent `1416 → 2191`, watchlist `17 → 18 → 19`). You lose the agent's continuity for no gain.
- **Persona:** re-running `upsert_persona` with the same values is a no-op churn — it overwrites the
  object with itself. To change it, read → merge → PUT once; don't "set it up again."
- **Target accounts:** removing and re-adding an account is not a "refresh." It re-enqueues list
  ingestion (which returns `completed` quickly) but is **not** a shortcut to fresher content or
  coverage — people-mapping is a separate `enrich_company` step that a re-add does not run for you. It
  churns state for no benefit; enrich or re-run the agent instead.

If a user asks to "delete everything and redo it," push back and offer the in-place edit.

## Persona — replace-whole (PUT)

`upsert_persona` **overwrites the entire persona** on every call. It is not a patch. Send a partial
object and you drop every field you left out.

**Safe pattern — always read, merge, write:**

```
current = get_persona()
# merge your change into the FULL object
next = { ...current, job_title: [ ...current.job_title, "CRO", "GTM Engineer" ] }
upsert_persona(next)   # complete object, documented fields only
```

Send **only** the documented persona fields (`job_title`, `exclude_job_title`, `seniority`,
`headcount`, `industry`, `location`, `additional_info`). Don't invent extra keys — an unknown field
(e.g. `name`) has been observed to return 200 and **wipe the persona to empty**. There is one persona
per workspace; there's no id to target and no "personas" collection.

## Target account list — append and remove

- `add_top_accounts` **appends**. Re-adding an account already on the list is harmless; there's no
  "replace the whole list" tool.
- `remove_top_accounts` is **destructive** — pass exactly one of `ids` (numeric, read them from
  `read_top_account_list` first; don't guess) or `identifiers` (domains / LinkedIn URLs, when you
  don't have the ids).
- Identify accounts by **`domain`** when you can — it matches more accurately than a LinkedIn URL.
- Adding accounts **enqueues ingestion**. Poll `get_top_account_list_status` to `completed` before you
  expect coverage or content for them.

## Watchlist entities — idempotent append, LinkedIn-first

- `add_watchlist_entities` takes **LinkedIn URL or handle** (preferred), up to 100 per call. On
  **company** lists only, `domain` is accepted as a fallback when no LinkedIn identifier is known —
  sending a domain to a profile list returns 422. It is **idempotent on (list, entity)**: entities
  already present are skipped, so re-runs are safe.
- **Lead with the LinkedIn URL, not the domain.** An ambiguous or un-enrichable domain bounces with a
  resolution error (seen with `meandu.com`, `zonal.co.uk`, `dojo.tech`) and you end up supplying the
  URL anyway — so resolve it up front rather than paying a failed round-trip first.
- A watchlist's `type` is **immutable**; its `kind` (company vs profile) is derived from the type.
  Company lists take companies; profile lists take people.
- Adding an entity to a watchlist does **not** build coverage for it. If you need the people at a
  watchlisted company, run `enrich_company` with that company's **domain** separately — entities
  added by LinkedIn identifier alone show "No coverage" until you do.

## Coverage — trigger, then poll; doesn't refresh itself

- `enrich_company` is a **trigger**: it starts a mapping and returns immediately. The mapping record
  only becomes readable once its stage is `completed` — reading it earlier returns "not found", which
  is normal, not an error.
- It's **idempotent**: retrying a failed or recent enrichment reuses the same request rather than
  creating a duplicate.
- Coverage is **not** rebuilt when the persona changes. After you widen or reshape the persona,
  re-trigger `enrich_company` on the accounts you want re-mapped against the new ICP.
- **Reading the people:** the `request_id` returned by `enrich_company` is accepted directly as
  `get_company_mapping`'s `mapping_id`. To browse every existing mapping (and their ids), go through
  `list_company_mappings`.

## Signal runs — trigger, then poll

- `launch_signal_run` **enqueues** work and returns `runs[]`, each with its own `signal_request_id`
  (keyword/job agents → one run; watchlist agents → two). Poll `get_signal_run` per run to a
  terminal stage (`completed`, `completed_partial`, `failed`) before drawing conclusions.
- `lookback_days` is a **top-level integer** (1–180, default 90): `{ agent_id, lookback_days: 90 }` —
  and the right JSON type (`90`, not `"90"`; strings are rejected). Only the REST endpoint nests it
  inside `parameters`.
- A run launched immediately after `create_agent` can be rejected while the agent's keywords are still
  being indexed — if a fresh keyword agent's first run errors on input, wait a short moment and launch
  again.
- Read results from `list_signals` (the event feed) and `get_contents` (the corpus). Run counters and
  the detection feed are two different reads — trust `list_signals` for what was actually found.

## Agents — configure vs bind

- `configure_agent` changes **keywords, `start_date`, name, and enabled** (pause = `enabled:false`,
  resume = `enabled:true`). Changing any watch parameter requires sending the **full**
  `tracking_keywords` list — it replaces the existing one.
- `bind_agent_watchlist` changes **which list** a watchlist agent watches; passing both fields null
  unlinks it.

## Errors you'll see, and what they mean

| Error                                                            | Meaning & response                                                                                               |
| ---------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------- |
| `Invalid input. Re-check field names, types, and allowed values` | Bad body — check the param names/types/enums against `tool-map.md`, fix, retry.                                  |
| `Resource not found. Re-check the identifier`                    | Wrong id, or the record isn't ready yet (e.g. a mapping mid-flight). Discover the id via the matching read tool. |
| `API key revoked or invalid`                                     | The `sk_live_` key is stale — regenerate it and reconnect the MCP.                                               |
| `403` (feature not enabled)                                      | The workspace doesn't have that feature — **surface it, don't retry.**                                           |
| `429` (rate limit)                                               | Back off; respect `retry_after` from `get_rate_limit`.                                                           |
