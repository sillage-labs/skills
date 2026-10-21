# Recipes — copy-paste curl for the common flows

All assume:

```bash
export SILLAGE_API_KEY="sk_live_..."
export SILLAGE="https://api.getsillage.com/api/v3"
auth=(-H "Authorization: Bearer $SILLAGE_API_KEY")
json=(-H "Content-Type: application/json")
```

Every answer is `{ data }`: read results with `jq '.data…'`. Poll politely (a few seconds, backoff).

## Contents

- [Smoke test the key](#smoke-test-the-key)
- [Check setup-state](#check-setup-state-one-call)
- [Persona](#persona)
- [Key accounts list](#key-accounts-list)
- [Set up a signal from a template](#set-up-a-signal-from-a-template)
- [Launch a run, poll, read detections](#launch-a-run-poll-read-detections)
- [Edit loop](#edit-loop--snapshot--diff--patch)
- [Cursor-walk a search](#cursor-walk-a-search)
- [What's running right now](#whats-running-right-now-one-call)

## Smoke test the key

```bash
curl -s -o /dev/null -w '%{http_code}\n' "$SILLAGE/personas" "${auth[@]}"   # 200 = key works; 401 = bad key
```

## Check setup-state (one call)

```bash
curl -s "$SILLAGE/setup-state" "${auth[@]}" \
  | jq '.data.checklist | map_values({status, message, recommended_next_route})'
```

## Persona

```bash
curl -s "$SILLAGE/personas" "${auth[@]}" | jq '.data[] | {id, name, job_titles, locations}'

# none yet: create one (job_titles + locations are what setups need)
pid=$(curl -s -X POST "$SILLAGE/personas" "${auth[@]}" "${json[@]}" -d '{
  "name": "Sales leaders",
  "job_titles": ["Head of Sales","VP Sales"],
  "locations": ["France"],
  "seniorities": ["vp","c_suite"],
  "headcounts": ["51-200","201-500"]
}' | jq -r '.data.id')                                       # also show .meta.warnings, if any

# later: change only what's off
curl -s -X PATCH "$SILLAGE/personas/$pid" "${auth[@]}" "${json[@]}" -d '{"locations":["France","Belgium"]}'
```

## Key accounts list

```bash
lid=$(curl -s -X POST "$SILLAGE/lists" "${auth[@]}" "${json[@]}" \
  -d '{"type":"prospect","name":"Target accounts","tier":1}' | jq -r '.data.id')

# 1–100 per call; partial success: inspect errors[]
curl -s -X POST "$SILLAGE/lists/company/$lid/members" "${auth[@]}" "${json[@]}" -d '{
  "members":[{"linkedin_url":"https://www.linkedin.com/company/acme"},{"domain":"other.com"}]
}' | jq '{added: (.data|length), errors}'
```

Monitoring those accounts is paid: confirm with the user, then send `signal_buckets: ["essential"]`
on `PUT /lists/company/$lid` (whole list) or `PUT /account-monitoring/accounts` (picked accounts).

## Set up a signal from a template

The template's `setup.fields` (JSON Schema) is the contract: build `parameters` from it, never
from memory.

**1 — Pick a live template and read its fields.**

```bash
curl -s "$SILLAGE/signal-templates?page_size=100" "${auth[@]}" \
  | jq '.data[] | {id, key, name, scopes, pricing}'

tid=12   # chosen id
curl -s "$SILLAGE/signal-templates/$tid" "${auth[@]}" \
  | jq '.data.setup.fields | {required, properties: (.properties | map_values({type, enum: (.items.enum // .enum), default, label: ."x-label", list_type: ."x-list-type"}))}'
```

**2 — Build `parameters`.** For each property: required and no default → you must send it
(ask the user if unknown); has a default → leave it out unless the user wants otherwise; `enum` →
only those values. A property with `x-list-type` is `list_id`: it must be one of the workspace's
lists of that type.

```bash
# list_id needed? find a list of that type, or create + fill one (nothing is created for you)
curl -s "$SILLAGE/lists?type=competitor" "${auth[@]}" | jq '.data[] | {id, name, member_count}'
```

**3 — Create.**

```bash
curl -s -X POST "$SILLAGE/signal-setups" "${auth[@]}" "${json[@]}" -d '{
  "signal_template_id": 12,
  "name": "Competitor engagement",
  "persona_id": 3,
  "parameters": {"list_id": 45, "scopes": ["account_monitoring"]},
  "schedule": null
}' | tee /tmp/setup.json | jq 'if .data then .data | {id, status, parameters} else . end'
```

**4 — Handle a `422` field by field.** `errors` names each faulty field by its path; fix only those
and resend:

```bash
jq -r '.errors // {} | to_entries[] | "\(.key): \(.value | join("; "))"' /tmp/setup.json
```

| `errors` key                           | Meaning → fix                                                                                                                                               |
| -------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `parameters.<field>`                   | Missing, wrong type, or not an allowed value → re-read that property in `setup.fields`.                                                                     |
| `parameters.list_id`                   | Missing, unknown, or not a list of the `x-list-type` → pick or create the right list.                                                                       |
| `parameters.schedule`                  | `schedule` sent inside `parameters` → move it to the top level.                                                                                             |
| `persona_id`                           | Several personas and none sent, an unknown id, or (title `Persona incomplete`) the persona lacks `job_titles` / `locations` → `PATCH /personas/{id}` first. |
| _(no `errors`)_ + `Persona incomplete` | The workspace has no persona → `POST /personas`.                                                                                                            |

A `409` means the list belongs only to other personas, or the template can't be set up (archived /
not available yet): pick another list or template.

## Launch a run, poll, read detections

```bash
sid=77   # a signal setup id
# launch returns an ARRAY of runs (directions + optional Discovery)
ids=$(curl -s -X POST "$SILLAGE/signal-runs" "${auth[@]}" "${json[@]}" \
  -d "{\"signal_setup_id\":$sid}" | jq -r '.data[].id')

for id in $ids; do
  while :; do
    st=$(curl -s "$SILLAGE/signal-runs/$id" "${auth[@]}" | jq -r '.data.status')
    echo "run $id: $st"; [[ "$st" == completed || "$st" == failed ]] && break; sleep 5
  done
done

curl -s -X POST "$SILLAGE/signals/count" "${auth[@]}" "${json[@]}" -d "{\"signal_setup_id\":$sid}" | jq '.data.total'
curl -s -X POST "$SILLAGE/signals/query" "${auth[@]}" "${json[@]}" \
  -d "{\"signal_setup_id\":$sid,\"limit\":50}" | jq '.data[] | {signal_type, company_id, excerpt, source_url}'
```

`409` on launch = a run of this setup is still going, or the setup is disabled.

## Edit loop — snapshot → diff → patch

```bash
# snapshot
curl -s "$SILLAGE/personas" "${auth[@]}"
curl -s "$SILLAGE/lists" "${auth[@]}"
curl -s "$SILLAGE/signal-setups?persona_id=3" "${auth[@]}" | jq '.data[] | {id, name, status, parameters}'

# retune keywords (merge: only the sent key changes)
curl -s -X PATCH "$SILLAGE/signal-setups/77" "${auth[@]}" "${json[@]}" \
  -d '{"parameters":{"keywords":["revenue operations"]}}'

# pause / resume, weekly schedule
curl -s -X PATCH "$SILLAGE/signal-setups/77" "${auth[@]}" "${json[@]}" -d '{"status":"disabled"}'
curl -s -X PATCH "$SILLAGE/signal-setups/77" "${auth[@]}" "${json[@]}" -d '{"schedule":"weekly"}'

# watch another list: the list is fixed (409) → create a new setup, then delete the old one
curl -s -X DELETE "$SILLAGE/signal-setups/77" "${auth[@]}"

# remove a list member
curl -s -X DELETE "$SILLAGE/lists/company/45/members/9012" "${auth[@]}"
```

## Cursor-walk a search

Same loop for `signals/query`, `leads/query`, `contents/query`, `content-requests/query`:

```bash
cursor=null
while :; do
  resp=$(curl -s -X POST "$SILLAGE/signals/query" "${auth[@]}" "${json[@]}" \
    -d "$(jq -nc --argjson c "$cursor" '{limit:100} + (if $c then {cursor:$c} else {} end)')")
  echo "$resp" | jq -c '.data[]'
  [[ $(echo "$resp" | jq -r '.meta.cursor.has_more') == true ]] || break
  cursor=$(echo "$resp" | jq '.meta.cursor.next')
done
```

Keep the same filters on every page: a cursor reused with other filters answers `422`.

## What's running right now (one call)

```bash
curl -s "$SILLAGE/requests-status" "${auth[@]}" | jq '.data[] | {type, status, label}'
```
