# Expansion playbook

The craft the customer hired you for: turning one sparse answer into a complete, optimized set.
Most users don't know what would be useful — so **propose, then let them cut**. Always show your
candidates and a one-line rationale; never silently write what they didn't see.

## Contents

- [Job-title expansion](#job-title-expansion)
- [Keyword expansion for keyword_detection](#keyword-expansion-for-keyword_detection)
- [Signal → agent mapping](#signal--agent-mapping)

## Job-title expansion

A persona that lists only "VP Sales" misses most of the buying committee. From one seed title,
generate four directions and read them back for the customer to trim:

1. **Synonyms** — the same role under different names. _VP Sales → VP of Revenue, Chief Revenue
   Officer, Head of Sales, Commercial Director._
2. **The seniority ladder** — the same function above and below. _Sales Director, Head of Sales,
   Sales Manager_ — then ask where the budget actually sits so you set `seniority` correctly.
3. **Adjacent functions that share the pain** — RevOps, Sales Enablement, Sales Operations for a
   sales-tooling product.
4. **Exclusions** — the titles a broad match will wrongly catch. _"Sales" catches Sales
   \_Development_ Reps and retail "Sales Associates"\_ → put those in `exclude_job_title`.

> **Localization:** if the persona targets non-English geographies, keep the seed titles in
> English — Sillage translates titles per language downstream. Your job is the _variant set_, not
> the translation.

**Worked example — "we sell to marketing teams":**
`job_title`: Marketing Director, VP Marketing, CMO, Head of Marketing, Head of Growth, Demand
Generation Manager, Growth Lead · `exclude_job_title`: Marketing Intern, Marketing Assistant ·
`seniority`: head, director, vp, c_suite.

## Keyword expansion for keyword_detection

A `keyword_detection` agent watches LinkedIn posts. Users typically type their _product name_ and
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

## Signal → agent mapping

Map each buying signal the customer named (Wave 4 / Wave 5) to the agent that catches it. The
Sillage MCP `create_agent` exposes these types:

| The customer says…                                 | Agent type          | What it needs                         |
| -------------------------------------------------- | ------------------- | ------------------------------------- |
| "When someone posts about \<pain\> / our category" | `keyword_detection` | expanded `tracking_keywords`          |
| "When a lead/champion changes jobs or is promoted" | `job_update`        | nothing — name only; tracks persona-matched people |
| "Track our competitors' companies"                 | `competitor`        | a competitor watchlist (auto-created) |
| "Track our partners"                               | `partner`           | a partner watchlist                   |
| "Track our existing customers (expansion)"         | `customer`          | a customer watchlist                  |
| "Track these creators our buyers follow"           | `influencer`        | an influencer-profile watchlist       |
| "Track our champions / advocates"                  | `champion`          | a champion-profile watchlist          |

Pick the **one or two highest-signal** agents to start — usually a `keyword_detection` agent off
the expanded keywords plus one watchlist agent off their competitors or influencers. More agents
is not better; signal quality is. Add the rest once the first detections come back and the
customer trusts the output.
