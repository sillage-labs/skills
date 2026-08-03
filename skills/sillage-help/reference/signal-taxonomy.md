# Signal & content taxonomy

The full vocabulary of what Sillage detects and collects. Two layers: **detections** (the events an
agent produces) and **contents** (the raw material behind them). Which detections you can produce
depends on which agent types you run.

## Which agent produces what — the closed list

You can create **nine** Agent types: `keyword_detection`, `job_posting_keyword_detection`,
`job_update`, `competitor`, `partner`, `customer`, `influencer`, `champion`, and
`champion_tracking`. That is the whole creatable set — there is no "employee engagement" Agent or
auto-derived Agent. Don't propose a type that doesn't exist or promise a signal Sillage can't emit.

| Detection group          | Produced by                                                                                      |
| ------------------------ | ------------------------------------------------------------------------------------------------ |
| Keyword (LinkedIn posts) | `keyword_detection`                                                                              |
| Keyword (job postings)   | `job_posting_keyword_detection`                                                                  |
| Career moves             | `job_update`                                                                                     |
| Watchlist engagement     | the matching watchlist agent (`competitor` / `partner` / `customer` / `influencer` / `champion`) |
| Champion career moves    | `champion_tracking`                                                                              |

## Detection types, grouped

Detections are keyed by a signal type. Grouped by what they mean:

**Keyword**

- `keywordDetection` — a LinkedIn post matched one of an agent's tracking keywords.
- `jobPostingKeywordDetection` — a job posting from a tracked company matched one of a
  `job_posting_keyword_detection` agent's keywords (title or description).

**Career moves** (from a `job_update` agent)

- `newJob` — a tracked person started a new role.
- `recentlyPromoted` — a tracked person was promoted.

**Champion career moves** (from a `champion_tracking` Agent)

- `championJobChange` — a profile on a champion watchlist changed jobs, was promoted, or left.

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

> Career, job-posting-keyword, and watchlist signals depend on the matching Agent type being set up
> and run. Setup recommends at least 3 Agents; prioritize `job_update`, `competitor`, `customer`, then
> `keyword_detection` when those signals fit.

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
  (`get_contents`), which also carries the person→company link. The default `normalized` format is a
  clean projection (text, author, engagement, mentions); `detailed` is a deprecated alias of it.

> **`get_contents` unfiltered is a superset, not your target accounts.** With no filter it returns the
> whole workspace corpus — largely person-authored posts (`company_id: null`) and content from
> companies beyond your target list — so a small target-account share is **expected, not a defect**. Filter by `company_id` /
> `company_domain` for a target account's material. And don't spin up a sub-agent to regex the corpus
> for a company's partners or tech stack — the data isn't shaped for that (company-page bodies are
> often empty), and it isn't what a watchlist agent needs.
