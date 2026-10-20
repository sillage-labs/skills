# Write semantics — how each write behaves, and how not to break things

Reading is safe. Writing is where a workspace gets damaged (or billed). This is the exact behavior of
every write and the safe pattern for each.

## Contents

- [Edit in place — recreate only for frozen fields](#edit-in-place--recreate-only-for-frozen-fields)
- [Personas — patch one, add another deliberately](#personas--patch-one-add-another-deliberately)
- [Lists: only the defaults exist](#lists-only-the-defaults-exist)
- [Monitoring — billed, per persona](#monitoring--billed-per-persona)
- [Signal setups — merge patch, frozen list and persona](#signal-setups--merge-patch-frozen-list-and-persona)
- [Signal runs — trigger, then poll](#signal-runs--trigger-then-poll)
- [Errors you'll see](#errors-youll-see)

## Edit in place — recreate only for frozen fields

Most changes have an in-place tool; use it. Deleting and recreating a signal setup mints a new id,
loses its history (`last_detected_at`, the stats behind it) and can re-spend credits on a first
`weekly` run. Recreate **only** to change what is frozen:

- a setup's **list** (`parameters.list_id`) or its **persona** → create the new setup, then
  `delete_signal_setup` the old one (it frees the list);
- a list's **type** or **tier** → create a new list, move members, re-point setups the same way.

If a user asks to "delete everything and redo it," push back and offer the in-place edit.

## Personas — patch one, add another deliberately

- `update_persona(persona_id, …)` is a **patch**: only the fields you send change. But each list
  field you send **replaces that list whole** — to add a title, send the full current `job_titles`
  plus the new one (read it from `get_personas`).
- `create_persona` **always adds** a business unit; it never replaces one. Use it only when the user
  asks for another persona. A workspace without business units refuses it ("contact Sillage") and
  nothing changes — don't fall back to overwriting the existing persona.
- `job_titles` and `locations` are required (≥1 entry each), and `locations` needs at least one
  **country**: a city-only persona saves but can't be mapped. Most signal templates are refused
  (422 "Persona incomplete") until the persona has both.
- `tag` must be unique among the workspace's active personas.
- Unknown keys are refused (422), not silently dropped.

```
personas = get_personas()
target = personas.data.find(p => p.name == "Enterprise EMEA")
update_persona({ persona_id: target.id,
                 job_titles: [...target.job_titles, "CRO", "GTM Engineer"] })
```

## Lists: only the defaults exist

- Every workspace starts with Tier 1/2/3 `prospect` lists (no persona, never deletable) and "All my
  employees", which "Engaged with my content on LinkedIn" uses when you leave `list_id` out. Any
  other list a signal watches must exist first, with the right `list_type`.
- Key accounts live in **customer** and **prospect** lists: run `list_lists` and add them to a tier
  list; `create_list` only for other types or extra lists. Adding accounts starts mapping. Attach a list to business units with
  `persona_ids`; no persona (or `[]`) means it belongs to **every** persona.
- `add_list_members` is an **idempotent append**, ≤100 per call, **partial success**: added members
  in `data`, failures in `errors` with a `code` (`multiple_domain_matches`, `domain_not_resolved`,
  `resolution_failed`, `wrong_entity_type`, `invalid_identifier`). A 200 can carry an empty `data` —
  always read `errors` and tell the user which ones failed.
- **Lead with the LinkedIn URL, not the domain.** An ambiguous domain bounces
  (`multiple_domain_matches`) and you end up supplying the URL anyway. `domain` is accepted on
  company lists only.
- **Champion lists only:** each member may carry `company` (`name` and/or `domain`) — where you know
  the contact today, used to detect their next move. Rejected (422) on any other list.
- `delete_list` / archiving is refused (409) while a setup reads the list, when it is the only list
  of its type and tier for its persona (or the only one attached to no persona), or for "All my
  employees". `get_list` → `is_deletable`.
- Smart lists (`mode` set) are managed in the app: no member writes through the MCP.

## Monitoring — billed, per persona

- Adding an account to a list starts its mapping; monitoring (`signal_buckets: ["essential"]`) pays
  for it and turns on content and signals. It costs credits **per account, per persona, per month**,
  charged **right away**; stopping (`null`) doesn't refund the month. Show `get_account_monitoring`'s
  price and get a yes first; `is_billed: false` means no cost to state.
- Whole list → `update_list(signal_buckets)` (applies to members added later too). Some accounts →
  `set_account_monitoring(company_ids, signal_buckets, persona_id?)`; accounts not on a list of that
  persona come back in `refused`, still with a 200.
- A list attached to several personas is monitored, and charged, in each of them. A monitored list
  attached to no persona is charged in every active persona.
- An account whose month can't be paid reads `not_covered` — a balance problem, not a setup problem.

## Signal setups — merge patch, frozen list and persona

- `create_signal_setup` takes `signal_template_id` (the environment-specific `id`, never reused from
  elsewhere — recognise templates by `key`), `parameters` (required, may be `{}`), and optionally
  `persona_id`, `name`, `schedule`.
- `parameters.scopes` is required on every template: `account_monitoring` (your monitored key
  accounts) and/or `discovery` (outside them, priced per signal found — see the template's
  `pricing`). Fields marked `only_with_discovery` are accepted only when `scopes` has `discovery`.
  The two job-offer templates accept `account_monitoring` only.
- Defaults are filled server-side and read back in `parameters` (e.g. brand keywords = the workspace
  domain; competitor-brand keywords = your competitor lists' names; `use_persona_job_titles: true`).
  Check the returned `parameters` instead of assuming.
- `persona_id`: optional with one persona, required with several (except champion and the two
  job-offer templates). The same template can be set up several times, per persona or per keyword set.
- `schedule` is top-level (`"weekly"` or `null`), never inside `parameters`. `"weekly"` runs once
  immediately — with `discovery` that first run spends credits at once.
- `configure_signal_setup` **merges** `parameters`: send only what changes; `null` removes a key, and
  answers that stop applying after a change must be sent as `null`. `status: "disabled"` pauses,
  `"enabled"` resumes. A new `list_id` is refused (409); `persona_id` isn't an argument (422).
- A wrong answer is refused with the path of the field (e.g. `parameters.scopes`) — fix that field
  and retry, don't rewrite the whole call.

## Signal runs — trigger, then poll

- `launch_signal_run(signal_setup_id)` returns `data[]`, one run per direction (keyword → one,
  interaction → inbound and/or outbound, champion → one). Poll each run's `id` with `get_signal_run`
  until `completed` or `failed`.
- `lookback_days` is a **top-level integer** (1–180, default 90), accepted only on account-monitoring
  runs of keyword, job-offer, competitor, partner and influencer signals; refused elsewhere.
- Re-running is safe (detections are deduplicated). A champion run launched while the previous one
  is still going returns an error.
- Read results from `list_signals` (filter `signal_setup_id`), not the run.

## Errors you'll see

| Error                                         | Meaning & response                                                                                                 |
| --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------ |
| `422` naming a field (`parameters.scopes`, …) | Bad or missing answer, or an undeclared argument (e.g. `persona_id` on `configure_signal_setup`): fix that field.   |
| `422` "Persona incomplete"                    | The persona lacks a job title or a location — `update_persona` first.                                              |
| `409` on a setup                              | Archived template, a list attached only to other personas, or an attempt to change the list.                       |
| `409` on a list                               | Still read by a setup, last list of its type/tier for its persona (or with no persona), or "All my employees".     |
| `404` / not found                             | Wrong id, or an id from another environment. Discover it with the matching read tool.                              |
| `403`                                         | Feature not in the plan, or a second persona without business units — **surface it, don't retry.**                 |
| `429`                                         | Rate limit (60 requests/minute) — back off, poll every 10–30 s.                                                    |
| `API key revoked or invalid`                  | The `sk_live_` key is stale — regenerate it and reconnect the MCP.                                                 |
