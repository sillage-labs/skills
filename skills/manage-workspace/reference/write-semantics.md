# Write semantics — how each write behaves, and how not to break things

Reading is safe. Writing is where a workspace gets damaged. This is the exact behavior of every write
class and the safe pattern for each.

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
`headcount`, `industry`, `location`, `additional_info`). Don't invent extra keys — unknown fields
don't help and can cost you the fields you meant to keep. There is one persona per workspace; there's
no id to target and no "personas" collection.

## Target account list — append and remove-by-id

- `add_top_accounts` **appends**. Re-adding an account already on the list is harmless; there's no
  "replace the whole list" tool.
- `remove_top_accounts` is **destructive** — it removes by id. Read the ids from
  `read_top_account_list` first; don't guess them.
- Identify accounts by **`domain`** when you can — it matches more accurately than a LinkedIn URL.
- Adding accounts **enqueues ingestion**. Poll `get_top_account_list_status` to `completed` before you
  expect coverage or content for them.

## Watchlist entities — idempotent append, LinkedIn-only

- `add_watchlist_entities` takes **LinkedIn URL or handle** (up to 100 per call) — there is no domain
  param here. It is **idempotent on (list, entity)**: entities already present are skipped, so re-runs
  are safe.
- A watchlist's `type` is **immutable**; its `kind` (company vs profile) is derived from the type.
  Company lists take companies; profile lists take people.
- Because entities are keyed by LinkedIn (not domain), a watchlisted company has **no domain**, so it
  gets **no coverage** on its own. If you need the people at a watchlisted company, run
  `enrich_company` with that company's **domain** separately.

## Coverage — trigger, then poll; doesn't refresh itself

- `enrich_company` is a **trigger**: it starts a mapping and returns immediately. The mapping record
  only becomes readable once its stage is `completed` — reading it earlier returns "not found", which
  is normal, not an error.
- It's **idempotent**: retrying a failed or recent enrichment reuses the same request rather than
  creating a duplicate.
- Coverage is **not** rebuilt when the persona changes. After you widen or reshape the persona,
  re-trigger `enrich_company` on the accounts you want re-mapped against the new ICP.
- **`request_id` ≠ `mapping_id`.** `enrich_company` / `get_account_mapping_stage` work with the
  request id; to read the actual people you go `list_company_mappings` → get the `mapping_id` →
  `get_company_mapping`.

## Signal runs — trigger, then poll

- `launch_signal_run` **enqueues** work and returns a `signal_request_id`. Poll `get_signal_run` to a
  terminal stage (`completed`, `completed_partial`, `failed`) before drawing conclusions.
- Put per-run options **inside `parameters`** (e.g. `{ agent_id, parameters: { lookback_days: 90 } }`),
  not at the top level.
- A run launched immediately after `create_agent` can be rejected while the agent's keywords are still
  being indexed — if a fresh keyword agent's first run errors on input, wait a short moment and launch
  again.
- Read results from `list_signals` (the event feed) and `get_contents` (the corpus). Run counters and
  the detection feed are two different reads — trust `list_signals` for what was actually found.

## Agents — configure vs bind

- `configure_agent` changes **keywords, name, and enabled** (pause = `enabled:false`, resume =
  `enabled:true`). It does not set job titles.
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
