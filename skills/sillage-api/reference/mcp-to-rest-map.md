# MCP tool → REST endpoint map

Every tool the Sillage MCP registers (names below drop the `sillage_v2_` prefix), with its REST
equivalent. Paths are relative to `https://api.getsillage.com/api/v3`; the MCP calls these same
routes under the hood. A tool not in this table doesn't exist — older skill versions named agent,
watchlist and target-account tools that are gone.

## Setup / meta

| MCP tool              | REST                                                                           |
| --------------------- | ------------------------------------------------------------------------------ |
| `get_setup_state`     | `GET /setup-state` — same checklist; the tool adds `recommended_next_tool`.    |
| `get_requests_status` | `GET /requests-status` — everything still being prepared.                      |
| `get_signal_playbook` | **No endpoint.** Reference text (also the MCP resource `sillage://playbooks`). |
| `list_types`          | **No endpoint.** Fixed type → kind pairs (see glossary "List").                |

Rate limits have no tool: read the `X-RateLimit-*` headers on any response.

## Business units (personas)

| MCP tool         | REST                                                         |
| ---------------- | ------------------------------------------------------------ |
| `get_personas`   | `GET /personas` (+ `GET /personas/{persona_id}`)             |
| `create_persona` | `POST /personas` — adds a business unit                      |
| `update_persona` | `PATCH /personas/{persona_id}` — only the fields sent change |

## Lists

| MCP tool             | REST                                                                            |
| -------------------- | ------------------------------------------------------------------------------- |
| `create_list`        | `POST /lists`                                                                   |
| `list_lists`         | `GET /lists`                                                                    |
| `get_list`           | `GET /lists/{kind}/{list_id}`                                                   |
| `update_list`        | `PUT /lists/{kind}/{list_id}` (incl. `signal_buckets` = monitor the whole list) |
| `delete_list`        | `DELETE /lists/{kind}/{list_id}`                                                |
| `list_members`       | `GET /lists/{kind}/{list_id}/members`                                           |
| `add_list_members`   | `POST /lists/{kind}/{list_id}/members` `{members}`                              |
| `remove_list_member` | `DELETE /lists/{kind}/{list_id}/members/{member_id}`                            |

## Account monitoring

| MCP tool                 | REST                                                                |
| ------------------------ | ------------------------------------------------------------------- |
| `get_account_monitoring` | `GET /account-monitoring` — totals and monthly cost                 |
| `set_account_monitoring` | `PUT /account-monitoring/accounts` — monitor / stop accounts (paid) |

## Signal catalogue & setups

| MCP tool                 | REST                                                                                    |
| ------------------------ | --------------------------------------------------------------------------------------- |
| `get_signal_templates`   | `GET /signal-templates` / `GET /signal-templates/{id}`                                  |
| `get_signal_setups`      | `GET /signal-setups` (`?persona_id=`) / `GET /signal-setups/{id}`                       |
| `create_signal_setup`    | `POST /signal-setups` `{signal_template_id, parameters, persona_id?, schedule?, name?}` |
| `configure_signal_setup` | `PATCH /signal-setups/{id}` (merge; `null` removes a key)                               |
| `delete_signal_setup`    | `DELETE /signal-setups/{id}`                                                            |

## Runs & signals

| MCP tool            | REST                                                                                 |
| ------------------- | ------------------------------------------------------------------------------------ |
| `launch_signal_run` | `POST /signal-runs` `{signal_setup_id}` → `202`, one run per direction (+ Discovery) |
| `get_signal_run`    | `GET /signal-runs/{id}` (+ `GET /signal-runs` to list)                               |
| `list_signals`      | `POST /signals/query` (cursor)                                                       |
| `get_signal`        | `GET /signals/{id}`                                                                  |
| `delete_signal`     | `DELETE /signals/{id}`                                                               |

## Accounts, people & content

| MCP tool                | REST                                                          |
| ----------------------- | ------------------------------------------------------------- |
| `list_company_mappings` | `GET /company-mappings`                                       |
| `get_company_mapping`   | `GET /company-mappings/{mapping_id}`                          |
| `get_company`           | `GET /companies/{id}`                                         |
| `get_lead`              | `GET /leads/{id}`                                             |
| `get_contents`          | `POST /contents/query` (cursor) / `GET /contents/{id}`        |
| `get_content_requests`  | `POST /content-requests/query` / `GET /content-requests/{id}` |

## Things REST adds beyond the MCP

- **Leads & companies**: `POST /leads/query`, `POST /leads/count`, `PATCH /leads/{id}`,
  `POST /companies/query`.
- **Signal counts**: `POST /signals/count`.
- **List history**: `GET /lists/{kind}/{list_id}/membership-logs`, `POST /lists/{kind}/{list_id}/restore`.
- **Billing**: `GET /credits/balance`, `GET /usage` (cursor).

Gone from v3 (no route, no tool): agents, watchlists, the target account list, `enrich-company-mapping`,
signal buckets, the singular `/persona` and signal create/update. Key accounts are customer/prospect list members;
mapping follows monitoring.
