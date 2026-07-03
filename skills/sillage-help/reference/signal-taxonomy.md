# Signal & content taxonomy

The full vocabulary of what Sillage detects and collects. Two layers: **detections** (the events an
agent produces) and **contents** (the raw material behind them). Which detections you can produce
depends on which agent types you run.

## Which agent produces what — the closed list

You can only ever create **seven** agent types: `keyword_detection`, `job_update`, `competitor`,
`partner`, `customer`, `influencer`, `champion`. That is the whole set — there is **no** `job_posting`
agent, no "employee engagement" agent, no auto-derived agent. If a detection group below has no
createable agent next to it, **nothing you can create produces it today**: don't propose an agent that
doesn't exist, and don't promise a signal Sillage can't emit.

| Detection group          | Produced by                                                                                       |
| ------------------------ | ------------------------------------------------------------------------------------------------- |
| Keyword                  | `keyword_detection`                                                                               |
| Career moves             | `job_update`                                                                                      |
| Watchlist engagement     | the matching watchlist agent (`competitor` / `partner` / `customer` / `influencer` / `champion`)  |
| Hiring intent            | **no createable agent** today                                                                     |
| Research (`deepSearch`)  | **no createable agent** today                                                                     |
| CRM                      | **no createable agent** today                                                                     |

## Detection types, grouped

Detections are keyed by a signal type. Grouped by what they mean:

**Keyword**

- `keywordDetection` — a LinkedIn post matched one of an agent's tracking keywords.

**Career moves** (from a `job_update` agent)

- `newJob` — a tracked person started a new role.
- `recentlyPromoted` — a tracked person was promoted.

**Hiring intent**

- `jobPosting` — a company published a job opening.
- `jobPostingInsight` — an enriched read on a posting.
- `jobPostingHiringManager` — the hiring manager behind a posting.

**Raw LinkedIn engagement**

- `linkedinComment` — a comment.
- `linkedinReaction` — a reaction.

**Watchlist cross-engagement** (from watchlist agents; each has an inbound and outbound comment form). A watchlist agent detects interactions with the entities **you put on the list** — it does not discover a company's competitors, partners, or tech vendors from the corpus. You supply the list; the agent watches it.

- `competitorInboundComment` / `competitorOutboundComment`
- `partnerInboundComment` / `partnerOutboundComment`
- `customerInboundComment` / `customerOutboundComment`
- `influencerInboundComment` / `influencerOutboundComment`
- `championInboundComment` / `championOutboundComment`
- `leadLikedCompetitorContent` / `leadCommentedCompetitorContent`
- `leadLikedInfluencerContent` / `leadCommentedInfluencerContent`

**Research**

- `deepSearch` — a buying-intent finding from web research (news, funding, announcements).

**CRM** (present in the taxonomy)

- `hubspotContactDetection`, `salesforceContactDetection` — a match against a connected CRM.

> In practice the keyword agent is the workhorse. Career and watchlist signals depend on the matching
> agent type being set up and run; hiring, research, and CRM signals have **no createable agent** (see
> the table at the top). Start with one or two high-signal agents and add types as the first detections
> come back.

## Content types

`get_contents` returns the raw corpus behind detections, filterable by company, person, date and type:

- `linkedinPost` — a post by a tracked person.
- `linkedinCompanyPost` — a post by a company page. Its body/excerpt **often arrives empty**, so
  company-page posts are unreliable for text-mining brand/tech/partner mentions.
- `linkedinComment` — a comment.
- `linkedinReaction` — a reaction.
- `newJob` / `recentlyPromoted` — the career-move records behind `job_update` detections.
- `jobPosting` / `linkedinJobPosting` — a job opening (generic, or LinkedIn-sourced).
- `webArticle` — an article found by research.
- `pressRelease` — a press release.
- `blogPost` — a blog post.
- `socialPost` — a post from a non-LinkedIn social source.

## Detections vs contents — which to read

- Want **"what happened"** (the event feed, by agent / signal type) → detections (`list_signals`).
- Want **"the material"** (posts/articles for a company or person, by type) → contents
  (`get_contents`), which also carries the person→company link. Read contents in its **detailed**
  form when you need the full author object, not just an excerpt.

> **`get_contents` unfiltered is a superset, not your target accounts.** With no filter it returns the
> whole workspace corpus — largely person-authored posts (`company_id: null`) and content from
> companies beyond your target list — so a small target-account share is **expected, not a defect**. Filter by `company_id` /
> `company_domain` for a target account's material. And don't spin up a sub-agent to regex the corpus
> for a company's partners or tech stack — the data isn't shaped for that (company-page bodies are
> often empty), and it isn't what a watchlist agent needs.
