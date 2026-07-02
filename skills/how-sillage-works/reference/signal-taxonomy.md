# Signal & content taxonomy

The full vocabulary of what Sillage detects and collects. Two layers: **detections** (the events an
agent produces) and **contents** (the raw material behind them). Which detections you can produce
depends on which agent types you run.

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

**Watchlist cross-engagement** (from watchlist agents; each has an inbound and outbound comment form)

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

> In practice the keyword agent is the workhorse. Career, hiring, watchlist, and research signals
> depend on the matching agent type being set up and run. Start with one or two high-signal agents and
> add types as the first detections come back.

## Content types

`get_contents` returns the raw corpus behind detections, filterable by company, person, date and type:

- `linkedinPost` — a post by a tracked person.
- `linkedinCompanyPost` — a post by a company page.
- `linkedinComment` — a comment.
- `linkedinReaction` — a reaction.
- `jobPosting` — a job opening.
- `webArticle` — an article found by research.
- `pressRelease` — a press release.
- `blogPost` — a blog post.

## Detections vs contents — which to read

- Want **"what happened"** (the event feed, by agent / signal type) → detections (`list_signals`).
- Want **"the material"** (posts/articles for a company or person, by type) → contents
  (`get_contents`), which also carries the person→company link. Read contents in its **detailed**
  form when you need the full author object, not just an excerpt.
