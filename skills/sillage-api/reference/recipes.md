# Recipes — copy-paste curl for the common flows

All assume:

```bash
export SILLAGE_API_KEY="sk_live_..."
export SILLAGE="https://api.getsillage.com/api/v2"
auth=(-H "Authorization: Bearer $SILLAGE_API_KEY")
json=(-H "Content-Type: application/json")
```

Poll politely (a few seconds, backoff). Every write below is safe to retry.

## Smoke test the key

```bash
curl -s "$SILLAGE/persona" "${auth[@]}"      # 200 {data:...|null} = key works; 401 = bad key
```

## Check setup-state (one call)

```bash
curl -s "$SILLAGE/setup-state" "${auth[@]}"   # {persona_set, list_uploaded, ingestion_complete, has_contents}
```

## Setup loop, end to end

**1 — Persona (GET → merge → PUT).** Read first, merge your change into the full object, then replace:

```bash
curl -s "$SILLAGE/persona" "${auth[@]}"     # inspect current data, merge locally, then:
curl -s -X PUT "$SILLAGE/persona" "${auth[@]}" "${json[@]}" -d '{
  "job_title": ["Head of Sales","VP Sales"],
  "exclude_job_title": ["Intern"],
  "seniority": ["vp","c_suite"],
  "headcount": ["51-200","201-500"],
  "industry": ["Software"],
  "location": ["France"],
  "additional_info": "SaaS teams selling to revenue orgs"
}'
```

**2 — Target accounts (add, then poll to completed).**

```bash
curl -s -X POST "$SILLAGE/top-account-list/accounts" "${auth[@]}" "${json[@]}" -d '{
  "accounts": [{"domain":"acme.com"}, {"linkedin_url":"https://www.linkedin.com/company/other"}]
}'                                           # 202 accepted

# poll until completed | failed
while :; do
  s=$(curl -s "$SILLAGE/top-account-list/status" "${auth[@]}" | jq -r .state)
  echo "$s"; [[ "$s" == completed || "$s" == failed ]] && break; sleep 5
done
curl -s "$SILLAGE/top-account-list/accounts/not-found" "${auth[@]}"   # what didn't match
```

**3 — Coverage / people (enrich per account, poll each).**

```bash
rid=$(curl -s -X POST "$SILLAGE/enrich-company-mapping" "${auth[@]}" "${json[@]}" \
  -d '{"domain":"acme.com"}' | jq -r .request_id)                    # 202 -> request_id

while :; do
  st=$(curl -s "$SILLAGE/content-requests/$rid" "${auth[@]}" | jq -r .stage)
  echo "$st"; [[ "$st" == completed || "$st" == account_mapping_failed ]] && break; sleep 5
done

# read the people
mid=$(curl -s "$SILLAGE/company-mappings" "${auth[@]}" | jq -r '.data[0].id')
curl -s "$SILLAGE/company-mappings/$mid" "${auth[@]}" | jq '.profiles'
```

**4 — Watchlist (create + add members).**

```bash
wid=$(curl -s -X POST "$SILLAGE/watchlists" "${auth[@]}" "${json[@]}" \
  -d '{"type":"competitor","title":"Direct competitors"}' | jq -r '.data.id')
# competitor -> kind "company"
curl -s -X POST "$SILLAGE/watchlists/company/$wid/entities" "${auth[@]}" "${json[@]}" -d '{
  "entities":[{"linkedin_url":"https://www.linkedin.com/company/rival"},{"domain":"rival.com"}]
}' | jq '{added:.data, errors:.errors}'      # 200; inspect errors[] for per-entity failures
```

**5 — Agent (create).**

```bash
# keyword agent
curl -s -X POST "$SILLAGE/agents" "${auth[@]}" "${json[@]}" -d '{
  "name":"Buying-intent keywords","type":"keyword_detection",
  "parameters":{"tracking_keywords":["\"sales enablement\"","RevOps"]}
}' | jq '.data.id'

# watchlist agent bound to the list from step 4
curl -s -X POST "$SILLAGE/agents" "${auth[@]}" "${json[@]}" \
  -d "{\"name\":\"Competitor watch\",\"type\":\"competitor\",\"watchlist_id\":$wid}"

# job-change agent (no params)
curl -s -X POST "$SILLAGE/agents" "${auth[@]}" "${json[@]}" -d '{"name":"Job changes","type":"job_update"}'
```

**6 — Launch a run, poll, read detections.**

```bash
aid=123   # an agent_id from GET /agents
# launch returns an ARRAY (keyword->1, watchlist->2)
ids=$(curl -s -X POST "$SILLAGE/workspace/signal-runs" "${auth[@]}" "${json[@]}" \
  -d "{\"agent_id\":$aid,\"parameters\":{\"lookback_days\":90}}" | jq -r '.[].signal_request_id')

for id in $ids; do
  while :; do
    st=$(curl -s "$SILLAGE/workspace/signal-runs/$id" "${auth[@]}" | jq -r .stage)
    echo "run $id: $st"
    [[ "$st" == completed || "$st" == completed_partial || "$st" == failed ]] && break; sleep 5
  done
done

curl -s "$SILLAGE/workspace/signals/count?agent_id=$aid" "${auth[@]}"
curl -s "$SILLAGE/workspace/signals?agent_id=$aid&limit=50" "${auth[@]}" | jq '.data'
```

## Edit loop — snapshot → diff → patch

```bash
# snapshot
curl -s "$SILLAGE/persona" "${auth[@]}"
curl -s "$SILLAGE/agents" "${auth[@]}"
curl -s "$SILLAGE/watchlists" "${auth[@]}"
curl -s "$SILLAGE/top-account-list/accounts?page_size=100" "${auth[@]}"

# patch: retune an agent's keywords + keep it enabled (one call)
curl -s -X PUT "$SILLAGE/agents/123" "${auth[@]}" "${json[@]}" \
  -d '{"parameters":{"tracking_keywords":["\"revenue operations\""]}}'

# patch: pause an agent
curl -s -X PUT "$SILLAGE/agents/123" "${auth[@]}" "${json[@]}" -d '{"enabled":false}'

# patch: rebind an agent to a different list (kind derived from agent type)
curl -s -X PUT "$SILLAGE/agents/123" "${auth[@]}" "${json[@]}" -d '{"watchlist_id":456}'

# patch: remove target accounts
curl -s -X POST "$SILLAGE/top-account-list/accounts/remove" "${auth[@]}" "${json[@]}" \
  -d '{"accounts":[{"domain":"stale.com"}]}'
```

## Cursor-paginate the whole detection feed

```bash
cursor=""
while :; do
  q="limit=100"; [[ -n "$cursor" ]] && q="$q&cursor=$cursor"
  resp=$(curl -s "$SILLAGE/workspace/signals?$q" "${auth[@]}")
  echo "$resp" | jq -c '.data[]'
  more=$(echo "$resp" | jq -r '.meta.has_more')
  cursor=$(echo "$resp" | jq -r '.meta.next_cursor')
  [[ "$more" == "true" ]] || break
done
```

## What's running right now (one call)

```bash
curl -s "$SILLAGE/requests-status" "${auth[@]}" | jq '{total:.meta.in_flight_total, data}'
```

## Long company-identifier lists — use the POST body variant

```bash
curl -s -X POST "$SILLAGE/contents/query" "${auth[@]}" "${json[@]}" -d '{
  "company_domain": ["a.com","b.com","c.com"],
  "content_type": ["linkedinPost"],
  "response_format": "normalized",
  "page_size": 100
}'
```
