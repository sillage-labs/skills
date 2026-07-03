# MCP tool → REST endpoint map

Every `sillage_v2_*` MCP tool referenced in `sillage-manage-workspace` and the other skills, with its
REST equivalent. Use this to translate any MCP-written instruction into an HTTP call. Method + path are
relative to `https://api.getsillage.com/api/v2`.

## Setup / meta

| MCP tool | REST |
| --- | --- |
| `get_setup_state` | **No endpoint.** Derive the 4 flags from reads — see the table in `endpoint-catalog.md`. |
| `get_rate_limit` | **No endpoint.** Read the `X-RateLimit-Limit` / `X-RateLimit-Remaining` headers on any response. |

## Persona

| MCP tool | REST |
| --- | --- |
| `get_persona` | `GET /persona` |
| `upsert_persona` | `PUT /persona` (replace-whole — GET → merge → PUT) |

## Target accounts (TAL)

| MCP tool | REST |
| --- | --- |
| `read_top_account_list` (view `accounts`) | `GET /top-account-list/accounts` |
| `read_top_account_list` (view `not_found`) | `GET /top-account-list/accounts/not-found` |
| `add_top_accounts` | `POST /top-account-list/accounts` (merge) — *not* `POST /top-account-list`, which wipes-and-replaces |
| `remove_top_accounts` | `POST /top-account-list/accounts/remove` (batch) or `DELETE /top-account-list/accounts/{id}` |
| `get_top_account_list_status` | `GET /top-account-list/status` (+ `GET /top-account-list/count`) |
| `get_top_accounts` | `GET /top-accounts` (activity superset, not the TAL) |

## Coverage / company mapping

| MCP tool | REST |
| --- | --- |
| `enrich_company` | `POST /enrich-company-mapping` → `202 {request_id}` |
| `get_account_mapping_stage` | `GET /account-mapping/{request_id}/stage` |
| `list_company_mappings` | `GET /company-mappings` (gives `mapping_id`) |
| `get_company_mapping` | `GET /company-mappings/{mapping_id}` (with `profiles[]`) |

## Watchlists

| MCP tool | REST |
| --- | --- |
| `list_watchlist_types` | **No endpoint.** The type→kind pairs are fixed: competitor/partner/customer → company; influencer/champion → profile. |
| `create_watchlist` | `POST /watchlists` |
| `list_watchlists` | `GET /watchlists` |
| `get_watchlist` | `GET /watchlists/{kind}/{watchlist_id}` |
| `update_watchlist` | `PUT /watchlists/{kind}/{watchlist_id}` |
| `delete_watchlist` | `DELETE /watchlists/{kind}/{watchlist_id}` |
| `add_watchlist_entities` | `POST /watchlists/{kind}/{watchlist_id}/entities` |
| `list_watchlist_entities` | `GET /watchlists/{kind}/{watchlist_id}/entities` |
| `remove_watchlist_entity` | `DELETE /watchlists/{kind}/{watchlist_id}/entities/{entity_id}` |

## Agents

| MCP tool | REST |
| --- | --- |
| `create_agent` | `POST /agents` |
| `get_agents` | `GET /agents` (list) / `GET /agents/{agent_id}` (one) |
| `configure_agent` | `PUT /agents/{agent_id}` (name / enabled / parameters) |
| `bind_agent_watchlist` | `PUT /agents/{agent_id}` (`watchlist_id` [+ `watchlist_kind`]; both null = unbind) |
| `delete_agent` | `DELETE /agents/{agent_id}` |

> REST folds `configure_agent` **and** `bind_agent_watchlist` into one `PUT /agents/{agent_id}` — send
> only the fields you're changing.

## Signals — runs & results

| MCP tool | REST |
| --- | --- |
| `launch_signal_run` | `POST /workspace/signal-runs` → array of `{signal_request_id}` |
| `get_signal_run` | `GET /workspace/signal-runs/{id}` (+ `GET /workspace/signal-runs` for all in-flight) |
| `list_signals` | `GET /workspace/signals` (**cursor**-paginated, rich filters) + `GET /workspace/signals/count` |
| `list_signal_types` | **No endpoint.** The enum is fixed — see the `signal_type` list on `POST /workspace/signals` in the catalog. |
| `add_signal` | `POST /workspace/signals` |
| `update_signal` | `PATCH /workspace/signals/{id}` |
| `delete_signal` | `DELETE /workspace/signals/{id}` |

## Content

| MCP tool | REST |
| --- | --- |
| `get_contents` | `GET /contents` (or `POST /contents/query` for long identifier lists). REST defaults `response_format=detailed`; MCP defaulted `normalized`. |
| `get_content_requests` | `GET /content-requests` (or `POST /content-requests/query`) |

## Things the REST API adds beyond the MCP surface

- `GET /requests-status` — one-shot "everything in flight across all three pipelines."
- `POST /top-account-list` — full destructive TAL replace (the MCP only appends/removes).
- v1 leads (`GET/PATCH /api/v1/workspace/leads`) — no MCP tool and no v2 endpoint.
