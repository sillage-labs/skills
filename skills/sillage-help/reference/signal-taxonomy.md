# Signal catalogue & content taxonomy

Three layers: the **catalogue** (the signal templates you can set up), the **detections** a setup
produces, and the **contents** behind them. Always read the live catalogue with
`get_signal_templates` (`GET /v3/signal-templates`) — this page is a snapshot to reason with.

## The catalogue — signal templates

Each template carries:

- `key` — stable identifier (`job-change`, …). Use it to name a signal; `id` is what
  `create_signal_setup` takes. `missing_priority_types` in setup state and `get_signal_playbook` use
  the `key`.
- `category` — `my_properties`, `my_network`, `competition`, `people`, `market`, `internal_intents`.
- `status` — only `live` can be set up; `coming_soon` templates are listed only with
  `include_coming_soon` and are refused on create.
- `scopes` — where it can look: `discovery` (outside your key accounts) and/or `account_monitoring`
  (your key accounts).
- `pricing` — `account_monitoring` (credits per key account per month, with its `signal_bucket`) and
  `discovery` (credits per signal found); either may be `null`.
- `setup.fields` — the JSON Schema of the `parameters` to send (labels, hints, defaults, which list type
  a `list_id` must be).
- `tags` — `popular`, and `not_set_up` (computed: the workspace has no setup of it yet).

### Live templates (snapshot)

| `key`                                              | Category    | Watches a list   |
| -------------------------------------------------- | ----------- | ---------------- |
| `engaged-with-my-content-on-linkedin`              | my_network  | employee (yours) |
| `mentioned-my-brand`                               | my_network  | —                |
| `engaged-with-my-partners-on-linkedin`             | my_network  | partner          |
| `champion-joined-a-new-company-or-promoted`        | my_network  | champion         |
| `engaged-with-competitors-on-linkedin`             | competition | competitor       |
| `mentioned-competitor-brand-on-linkedin`           | competition | —                |
| `job-change`                                       | people      | —                |
| `hiring-for-roles-my-product-serves`               | people      | —                |
| `mentioned-keywords-in-their-job-offers`           | market      | —                |
| `mentioned-keywords-on-linkedin`                   | market      | —                |
| `engaged-with-influencers-in-my-space-on-linkedin` | market      | influencer       |

Prices are not listed here: each template's `pricing` (and the bucket it sits in) gives them, and
they can differ per partner. Read them live, never quote a fixed number.

- **Nothing outside this list can be set up.** About 25 more templates are `coming_soon` (website
  visits, funding, headcount growth, tech adoption, CRM re-engagement…) — don't promise them.
- **No `customer` signal exists.** Customers are a key-account list, not a watched list.
- **List-watching templates need an existing list** of the named type (`list_id`). It is never
  created for you, and it detects interactions with the entities **you put on it** — it never
  discovers who a company's competitors or partners are.
- **Job-offer templates** (`hiring-for-…`, `mentioned-keywords-in-their-job-offers`) only run on key
  accounts today; their `scopes` is accepted but marked not honoured.
- **Champion and job-offer** setups are the only ones that don't need a complete persona.

## Detection types

Detections (`list_signals` / `POST /v3/signals/query`) carry `signal_setup_id` (→ its template) and a
`signal_type`, written snake_case on v3:

- **Keyword** — `keyword_detection` (a LinkedIn post matched), `job_posting_keyword_detection` (a job
  posting matched).
- **Career moves** — `new_job`, `recently_promoted`, `champion_job_change`.
- **List engagement** (inbound = a key-account person engages the watched entity; outbound = the
  watched entity engages a key account) — `competitor_inbound_comment` / `_outbound_comment`, same for
  `partner_`, `influencer_`, `champion_`; plus `lead_liked_competitor_content`,
  `lead_commented_competitor_content`.
- **Your own content** — `lead_liked_own_content`, `lead_commented_own_content`.
- **Raw engagement / jobs** — `linkedin_comment`, `linkedin_reaction`, `job_posting`,
  `job_posting_insight`, `job_posting_hiring_manager`.

## Content types

`get_contents` (`POST /v3/contents/query`) returns the raw corpus, filterable by company, person, date
and `content_type`:

- `linkedin_post` — a post by a person.
- `linkedin_company_post` — a post by a company page. Its body **often arrives empty**: unreliable for
  text-mining brand/tech/partner mentions.
- `linkedin_comment`, `linkedin_reaction`.
- `recently_promoted` — the career record behind a promotion.
- `linkedin_job_posting` — a job opening.
- `web_article`, `press_release`, `blog_post`.

## Detections vs contents — which to read

- **"What happened"** (the event feed, per setup / type) → signals.
- **"The material"** (posts/articles for a company or person) → contents.

> **Contents unfiltered is a superset, not your key accounts** — largely person-authored posts and
> companies beyond your lists, so a small key-account share is **expected**. Filter by company. Don't
> regex the corpus for a company's partners or tech stack — company-page bodies are often empty, and
> it isn't what a list-watching signal needs.
