# Editing playbook — reconcile a live workspace, one axis at a time

Editing is a read-diff-patch loop, never a blind write. Snapshot the current state, judge each axis,
then apply the single right fix. Read `write-semantics.md` first — it's what keeps each patch safe.

## Step 1 — snapshot

```
get_setup_state()            # the checklist + recommended_next_tool
get_personas()               # every business unit and its targeting
list_lists()                 # lists: type, member/monitored counts, persona_ids
get_signal_setups()          # setups: template, parameters, status, last_detected_at
get_account_monitoring()     # what is paid for, what is not covered
```

## Step 2 — diff each axis

For each axis, call it **match** (leave it), **mismatch** (fix it), or **missing** (add it):

| Axis          | Read with                                               | Mismatch looks like…                                          | The one fix                                                                           |
| ------------- | ------------------------------------------------------- | ------------------------------------------------------------- | ------------------------------------------------------------------------------------- |
| Persona       | `get_personas`                                          | wrong/narrow titles, seniority, industry, geo; no country     | `update_persona` (send full arrays for the lists you change)                          |
| Key accounts  | `list_lists` → `list_members` (customer/prospect)       | missing target companies, or off-ICP ones present             | `add_list_members` / `remove_list_member`                                             |
| Monitoring    | `get_account_monitoring`, `list_members` → `monitoring` | key accounts not monitored, or `not_covered`                  | `update_list` / `set_account_monitoring` (billed — confirm); top up for `not_covered` |
| Lists         | `list_lists`                                            | a list a signal needs is absent or empty                      | `create_list` (right type, `persona_ids`) + `add_list_members`                        |
| Signal setups | `get_signal_setups`, `get_setup_state`                  | needed kind missing, setup paused, stale keywords, wrong list | `create_signal_setup`, `configure_signal_setup`, or recreate-then-delete              |

For "missing" signal setups, `checklist.signal_setups.missing_priority_types` lists which of the 3
priority keys (`job-change`, `engaged-with-competitors-on-linkedin`, `mentioned-keywords-on-linkedin`)
aren't set up yet; a priority key absent from it already runs, even if `get_signal_setups` doesn't
show it. For any other template, `get_signal_templates(tags: ["not_set_up"], persona_id)` lists
what that persona hasn't tried yet.

## Step 3 — patch, then make it surface

- Signal setup change → `launch_signal_run(signal_setup_id)`, poll `get_signal_run`.
- Accounts or monitoring change → poll `get_requests_status` until those accounts are no longer listed.
- Confirm with `get_setup_state` that every dimension reads where you expect.

## Recipes for the common edits

**Widen the persona (surface more decision-makers).** `get_personas` → append the new titles /
seniorities to the full arrays → `update_persona(persona_id, job_titles: [...])`. Widening the persona
is the lever that surfaces buyers who were invisible — not adding keywords. Setups with
`use_persona_job_titles: true` pick the new titles up at their next run.

**Add a business unit.** Only when the user asks for a separate team/line. `create_persona` with its
own `job_titles` + `locations`; a 403 means business units aren't on (Sillage enables them on
request). Then attach its key-account lists (`create_list` / `update_list` with `persona_ids`) and set
up its signals with `persona_id`.

**Add a competitor to track.** Find a `competitor` list (`list_lists(type: "competitor")`) or create
one, then `add_list_members` with the competitor's LinkedIn URL. If no setup watches it yet, create
"Engaged with competitors on LinkedIn" (`engaged-with-competitors-on-linkedin`) with that `list_id`.
"Mentioned competitor brand on LinkedIn" defaults its keywords to your competitor lists' names.

**Add / remove target accounts.** `add_list_members` on the customer/prospect list (read `errors`).
New members of a monitored list are monitored (and billed) automatically. To drop one, read its
`id` from `list_members`, then `remove_list_member`.

**Tune keywords.** `get_signal_setups(signal_setup_id)` to see the current `parameters.keywords` →
`configure_signal_setup(signal_setup_id, parameters: { keywords: [...] })` with the full new set.
A bare keyword matches broadly, a "quoted" one the exact phrase only: quote generic phrases, leave
niche terms bare. Then `launch_signal_run` to test it.

**Pause / resume a signal.** `configure_signal_setup(signal_setup_id, status: "disabled")`, then
`"enabled"` to resume. Stop the weekly run without pausing: `schedule: null`.

**Watch another list / move to another persona.** Frozen fields. `create_signal_setup` with the new
`list_id` or `persona_id` and the same answers (copy `parameters` from `get_signal_setups`), confirm
it, then `delete_signal_setup` the old one (confirm it by name with the user).

**Fix a half-configured workspace.** Start from `get_setup_state` and follow each non-`ok`
dimension's `recommended_next_tool` in order: persona → accounts → signal setups → contents.
