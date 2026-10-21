# Expansion playbook

The craft the customer hired you for: turning one sparse answer into a complete, optimized set.
Most users don't know what would be useful — so **propose, then let them cut**. Always show your
candidates and a one-line rationale; never silently write what they didn't see.

## Contents

- [Job-title expansion](#job-title-expansion)
- [Keyword expansion](#keyword-expansion)
- [Signal → template mapping](#signal--template-mapping)
- [Populating lists](#populating-lists)
- [Account curation](#account-curation)

## Job-title expansion

A persona that lists only "VP Sales" misses most of the buying committee. From one seed title,
generate four directions and read them back for the customer to trim:

1. **Synonyms** — the same role under different names. _VP Sales → VP of Revenue, Chief Revenue
   Officer, Head of Sales, Commercial Director._
2. **The seniority ladder** — the same function above and below. _Sales Director, Head of Sales,
   Sales Manager_ — then ask where the budget actually sits so you set `seniorities` correctly.
3. **Adjacent functions that share the pain** — RevOps, Sales Enablement, Sales Operations for a
   sales-tooling product.
4. **Exclusions** — the titles a broad match will wrongly catch. _"Sales" catches Sales
   \_Development_ Reps and retail "Sales Associates"\_ → put those in `excluded_job_titles`.

> **Localization:** if the persona targets non-English geographies, keep the seed titles in
> English — Sillage translates titles per language downstream. Your job is the _variant set_, not
> the translation.

**Worked example — "we sell to marketing teams":**
`job_titles`: Marketing Director, VP Marketing, CMO, Head of Marketing, Head of Growth, Demand
Generation Manager, Growth Lead · `excluded_job_titles`: Marketing Intern, Marketing Assistant ·
`seniorities`: head, director, vp, c_suite.

## Keyword expansion

"Mentioned keywords on LinkedIn" (`mentioned-keywords-on-linkedin`) watches LinkedIn posts. Users typically type their _product name_ and
stop — which catches almost nothing, because buyers don't post your brand, they post their
**pain** and their **triggers**. Derive keywords from four veins:

1. **Pain language** — the words a buyer uses to describe the problem before they know you exist.
   _Cold-email tool → "low reply rate", "deliverability", "emails going to spam", "booking demos"._
2. **Category & competitor names** — the space they're shopping in. _"Outreach", "Salesloft",
   "Apollo", "sequencing tool"._
3. **Trigger phrases** — language around the moment of need. _"hiring SDRs", "scaling outbound",
   "building a sales team", "new sales motion"._
4. **Job-to-be-done verbs** — what they're trying to do. _"book more meetings", "fill the
   pipeline", "improve open rates"._

**Quoting discipline (this controls noise):** a **bare** keyword matches broadly (high recall, more
noise); a **"quoted"** keyword matches the exact phrase only (high precision, less noise). Start
broad on niche terms, quote the generic ones. _`deliverability`_ (bare, rare enough) vs.
_`"sales team"`_ (quoted, otherwise everywhere). Tell the customer which you quoted and why, and
tune after the first run.

> Propose 8–12 candidates across the four veins, grouped, with rationale. Let the customer delete
> the ones that feel off — that cut is where their domain knowledge earns its keep.

## Signal → template mapping

Map each buying signal the customer named (Wave 4 / Wave 5) to the catalogue template that catches
it. Recognise templates by their stable `key`; always confirm with `get_signal_templates` (only
`status: live` can be set up, and its `fields` are the real questions). The live catalogue today:

| The customer says…                                                 | Template `key`                                     | What it needs                                                           |
| ------------------------------------------------------------------ | -------------------------------------------------- | ----------------------------------------------------------------------- |
| "When someone posts about \<pain\> / our category"                 | `mentioned-keywords-on-linkedin`                   | expanded `keywords`                                                     |
| "When someone mentions us"                                         | `mentioned-my-brand`                               | nothing — `keywords` default to your domain                             |
| "When someone mentions a competitor"                               | `mentioned-competitor-brand-on-linkedin`           | `keywords`, or a competitor list (keywords default to its names)        |
| "When people engage with our team's posts"                         | `engaged-with-my-content-on-linkedin`              | nothing — uses the built-in "All my employees" list                     |
| "When prospects engage with our competitors"                       | `engaged-with-competitors-on-linkedin`             | a `competitor` list                                                     |
| "When prospects engage with our partners"                          | `engaged-with-my-partners-on-linkedin`             | a `partner` list                                                        |
| "When prospects engage with creators our buyers follow"            | `engaged-with-influencers-in-my-space-on-linkedin` | an `influencer` list                                                    |
| "When a champion changes jobs or is promoted"                      | `champion-joined-a-new-company-or-promoted`        | a `champion` list (members can carry `company`)                         |
| "When decision-makers join or get promoted at our target accounts" | `job-change`                                       | nothing, persona job titles by default (known contacts: champion)       |
| "When a target company's job offers mention \<stack\>"             | `mentioned-keywords-in-their-job-offers`           | `keywords` (roles, tools, stacks); account monitoring only              |
| "When a target company hires for the roles we serve"               | `hiring-for-roles-my-product-serves`               | persona job titles by default, or `job_titles`; account monitoring only |

Everything else in the catalogue (funding, tech adoption, headcount growth, website visits…) is
`coming_soon` — say so plainly rather than inventing a workaround. Every setup also needs `scopes`:
`account_monitoring` (your key accounts) and/or `discovery` (beyond them, priced per signal found).

Start with **3 high-signal** setups (`signal_setups` reads `warning` below 3), usually the keyword
template off the expanded keywords, `job-change`, and one list template off their competitors or
influencers. More setups is not
better; signal quality is. Add the rest once the first detections come back and the customer trusts
the output.

## Populating lists

A list signal (competitor / partner / influencer / champion) is only as good as the list you hand it.
Critically, **nothing is created or discovered for you** — the list must exist before the signal, and
the signal detects interactions with the entities you supply, nothing more. It will not mine the
ingested content to tell you who a prospect's partners or tech vendors are.

So populate a list the same way you expand keywords and titles: **propose 8–12 named candidates
from your own market knowledge, grouped with a one-line rationale, then let the customer cut.** For a
UK-hospitality payments seller you already know the rivals (Dojo, Zonal, SumUp…) and the ecosystem
(Deliveroo, OpenTable, Lightspeed…) — offer that list; don't go looking for it in the data.

Two hard rules, both from watching this go wrong:

- **Never regex-mine the corpus to "find" partners or a tech stack.** It's slow, expensive, and the
  data isn't shaped for it (company-page posts are largely empty). Propose from knowledge instead.
- **Never infer expected signal yield from content volume.** "This account has 4,000 posts, so the
  Partner signal will fire 20 times" is fabricated — yield depends on real interactions, not corpus
  size. Don't promise a number, and never contradict a check you just ran to justify an action.

Add members by **LinkedIn company URL** where you can — a raw domain bounces on ambiguous or
unresolvable companies (`multiple_domain_matches`, `domain_not_resolved`; real snag: `meandu.com`,
`zonal.co.uk`, `dojo.tech` had to be re-added by URL). Confirm the URL up front and skip the round-trip.

## Account curation

A target account only yields signals if it (or its people) are **active on LinkedIn**. Expect a large
share of any raw target list to be silent — in practice a handful of accounts drive most detections.
Prefer active, multi-site brands that post regularly over dormant ones. But **don't delete an account
just because `get_contents` shows zero** — confirm it's genuine inactivity, not an account that is
still being prepared, unmonitored, or has no mapped people (see `sillage-manage-workspace` →
"Troubleshoot — few or zero signals").
