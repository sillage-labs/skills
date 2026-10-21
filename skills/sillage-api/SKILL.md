---
name: sillage-api
description: >
  Drives a Sillage workspace over the v3 REST API when the Sillage MCP can't be installed (the
  MCP-less fallback): teaches Claude how to authenticate with an `sk_live_` key and call every endpoint
  (personas, lists, signal templates, signal setups, signal runs, signals, account monitoring,
  leads, companies, contents, credits) to run the same setup and edit loops the MCP skills
  describe. Use when there is no MCP connection, when a client is on an editor without MCP support,
  or when you must script Sillage from curl/HTTP. Use when you hear: "no MCP", "can't install the
  MCP", "use the API instead", "call the Sillage API", "curl Sillage", "REST fallback", "sk_live",
  "set up the workspace over HTTP".
metadata:
  owner: pf@getsillage.com
  version: 2.0.0
  model-tier: sonnet
  provider: Sillage REST API v3 (https://api.getsillage.com/api/v3)
  pairs-with: [sillage-onboarding, sillage-manage-workspace, sillage-help]
---

# Sillage API — drive a workspace over REST when there's no MCP

This is the **fallback transport**. The Sillage skills normally drive the workspace through the
[MCP server](https://api.getsillage.com/api/mcp/v2). When a client **can't install the MCP** — an
editor without MCP support, a locked-down environment, a pure-HTTP/curl script — the same work is
reachable over the plain **v3 REST API**. This skill teaches Claude how.

**This skill is transport only.** The _strategy_ — how to sharpen a persona, which signal templates
fit, which keywords catch buying intent, how to reconcile a live workspace — still lives in:

- `sillage-onboarding` — decide _what_ the targeting should be (interview + expansion).
- `sillage-manage-workspace` — the setup loop and the edit/write-safety rules, described against MCP tools.
- `sillage-help` — the mental model and glossary (persona, list, signal template, signal setup, run, detection).

Read those for the _what_ and _why_. Come here for the _how_ over HTTP. Their rules (confirm before
write, poll before read, a setup's list and persona are fixed) hold identically here.

## When to use

- The MCP is not connected and can't be, but you have (or can generate) a workspace API key.
- You're scripting Sillage from curl, a CI job, or any HTTP client.
- You need to translate an MCP-tool instruction into its REST equivalent → `reference/mcp-to-rest-map.md`.

## When NOT to use

- The MCP **is** available → use it and the MCP skills; it's higher-level and safer.
- You haven't decided the targeting yet → `sillage-onboarding` first.
- You just want to understand the product → `sillage-help`.

## The model you are driving

1. **Personas** (business units) say who the customer sells to. A workspace can have several.
2. **Lists** hold the companies or people signals work on: `customer` / `prospect` (the key
   accounts, which can be monitored), `competitor` / `partner` / `influencer` / `champion`, and the
   one `employee` list Sillage creates for you.
3. **Signal templates** are the catalogue (`GET /signal-templates`, `live` ones by default). Each
   carries `setup.fields`, a JSON Schema saying what to fill in.
4. **Signal setups** are a template set up for one persona, with its `parameters` (and `list_id`
   when the template watches a list). A setup finds nothing until it runs.
5. **Signal runs** execute a setup (`POST /signal-runs`, or `schedule: "weekly"`); the
   **signals** (detections) they write are read with `POST /signals/query`.

## The six things that trip up every REST caller (read before any call)

1. **Always v3.** Every call goes to `https://api.getsillage.com/api/v3` with
   `Authorization: Bearer sk_live_...`; the workspace comes from the key, never the URL. `v1` is
   legacy; `v2` is frozen and `/v2/agents` answers with `Deprecation`, `Sunset: Thu, 01 Apr 2027 00:00:00 GMT`
   and `Link: rel="deprecation"` headers (replaced by signal setups). `sk_live_` keys are created at
   <https://www.getsillage.com/app/settings/api-keys> and shown once; `mk_live_` is a partner key →
   `sillage-partner-api`. No key yet? The walkthrough is in `reference/conventions.md`.
2. **Everything answers `{ data }`.** Read results under `.data`. Pages add `meta.pagination`
   (`page`/`page_size` plain lists) or `meta.cursor` (`next`, `has_more`) on every `POST …/query`
   search except `POST /companies/query` (capped, no cursor) and on `/usage`: send `meta.cursor.next` back as `cursor` until `has_more` is `false`.
   Advisory text sits in `meta.warnings` — show it to the user.
3. **Setups follow their template.** `POST /signal-setups` takes
   `{signal_template_id, parameters, name?, schedule?, persona_id?}`; `parameters` is validated
   against that template's `setup.fields`. `parameters.list_id` is required when the template's
   `setup.fields.required` lists it (content engagement defaults to "All my employees"); **nothing
   is created for you**: find or create the list first. The
   persona must have job titles and locations, else `422 Persona incomplete`.
4. **Setups change by `PATCH` merge.** Send only what changes: a key overwrites, `null` removes it
   (refused on a parameter with a default), the rest is kept. `status: "enabled"|"disabled"`
   pauses/resumes. There is no `PUT`. A different `list_id` or `persona_id` → `409`: set the signal
   up again and delete the old one.
5. **Async writes return `202` — then poll.** A run launch answers `202` with an **array** of runs
   (one per direction, plus a Discovery run when `scopes` has `discovery`). Poll each `GET /signal-runs/{id}` until `status` is
   `completed` or `failed`. `GET /requests-status` shows everything in flight.
6. **Errors are RFC 9457 `application/problem+json`, `type: about:blank`.** `422` = invalid input,
   with `errors` keyed by field path (`parameters.list_id`, `persona_id`) — fix that field, don't
   retry blindly. `409` conflict, `402` out of credits, `403` plan doesn't allow it, `429` wait
   `Retry-After` seconds. Ids are numeric and environment-specific: re-resolve, never hard-code.

## Setup loop — stand up a workspace over REST

Start with `GET /setup-state`: its `checklist` (`persona`, `accounts`, `signal_setups`, `contents`)
gives each dimension a `status`, a `message` and a `recommended_next_route`.

1. **Persona** — `GET /personas`; create with `POST /personas` (`job_titles`, `locations` at
   least; `403` if one exists and business units aren't enabled) or fix with
   `PATCH /personas/{persona_id}`.
2. **Key accounts** — `POST /lists` `{type:"customer"|"prospect", name}` →
   `POST /lists/company/{list_id}/members` `{members:[{linkedin_url}|{domain}]}` (≤100 per call,
   partial success: check `errors`). Monitor them (paid) with `signal_buckets` on the list or
   `PUT /account-monitoring/accounts`.
3. **Signals** — `GET /signal-templates` → pick a `live` template → read `setup.fields` → create
   any list it needs → `POST /signal-setups`. Recipe: `reference/recipes.md` ("Set up a signal
   from a template").
4. **Run** — `POST /signal-runs` `{signal_setup_id}` → poll each run → `POST /signals/query`
   `{signal_setup_id}` (and `POST /signals/count`).

Full detail per call: `reference/endpoint-catalog.md`; copy-paste curl: `reference/recipes.md`.

## Edit loop — change a live workspace over REST

Never write blind. **Snapshot → diff → patch the one axis that's off**, as
`sillage-manage-workspace` prescribes.

1. **Snapshot:** `GET /personas`, `GET /lists`, `GET /signal-setups` (filter `?persona_id=`).
2. **Diff** each axis (persona / lists / setups) → match / mismatch / missing.
3. **Patch** with the right write:
   - Persona → `PATCH /personas/{persona_id}` with only the changed fields.
   - List members → `POST …/members` to add, `DELETE …/members/{member_id}` to remove.
   - Setup keywords / scopes / pause / rename / schedule → `PATCH /signal-setups/{id}`.
   - Setup on another list or persona → `POST /signal-setups` anew, then `DELETE` the old one.
   - Then launch a run if the change should surface new results.

## Close with a summary

```
## What I changed
<persona / lists / setups — before → after, in plain language>

## What's running now
<setups enabled · runs launched (run ids) · schedules>

## Waiting on
<anything still 202/in_progress, which endpoint to poll, and when to re-check>
```

## Reference

- `reference/conventions.md` — auth, keys, the envelope, pagination, async/poll, RFC 9457 errors,
  rate limits, deprecation headers.
- `reference/endpoint-catalog.md` — every v3 endpoint, grouped, with method, path, body and write class.
- `reference/mcp-to-rest-map.md` — each MCP tool → its REST endpoint(s).
- `reference/recipes.md` — copy-paste curl: setup from a template, runs, the edit loop, cursor walks.

## Canonical docs (fetch when in doubt)

This skill mirrors the official docs as of its version. If something looks stale, the live source wins:

- **API docs:** <https://www.getsillage.com/docs/api/v3>
- **OpenAPI spec (JSON):** <https://api.getsillage.com/api/v3/docs/spec> · Swagger UI `/api/v3/docs`
- **Get an API key:** <https://www.getsillage.com/app/settings/api-keys>
