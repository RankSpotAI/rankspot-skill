# RankSpot SEO Skill

Programmatic SEO and AI visibility for agents. See how ChatGPT, Perplexity and Google answer questions about your brand, track competitors, score keywords, analyse backlinks, find forum threads worth joining, mine "People Also Ask" questions, plan work as actions, and generate articles, all through the RankSpot API.

## Install

```bash
npx skills add RankSpotAI/rankspot-skill --skill rankspot
```

Then set your API key (get it from **Settings, then API Keys** in the [RankSpot dashboard](https://rankspot.ai)):

```bash
export RANKSPOT_API_KEY=your_key_here
```

No other dependencies. The skill uses `curl` and `jq`.

## Capabilities

| Capability              | What it does                                                                       |
|-------------------------|------------------------------------------------------------------------------------|
| **Workspace**           | Brand, domain and business context. Read this first.                                |
| **AI Visibility**       | Tracked prompts, captured answers, cited pages, fanout queries, scored summary       |
| **Actions**             | Every piece of planned work, typed. `write_article` actions generate articles.       |
| **Articles**            | AI-generated articles with full HTML, publish dates and index status                 |
| **Keywords**            | Track, score, filter and semantically cluster keywords                               |
| **Backlinks**           | Competitor and own backlinks with domain rank, spam score and outreach status        |
| **Competitors**         | Brands you track, plus brands RankSpot discovered in AI answers                      |
| **Forum Opportunities** | Reddit and Quora threads worth joining for AI visibility                             |
| **People Also Ask**     | PAA questions mined from search results for your keywords                            |
| **Categories**          | Group actions and articles                                                           |
| **Research**            | Live Google search and page fetch. Costs credits per call.                           |
| **Search Console**      | Clicks, impressions, CTR, position, index status, indexing requests                  |

## Commands

```bash
# Read the workspace: brand, domain, what the business does
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/workspace | jq .

# Score AI visibility over a period (both dates required)
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/ai-visibility/summary?startDate=2026-08-01&endDate=2026-08-31" | jq .

# Pages the AI engines cited, most cited first
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/ai-visibility/citations?type=new" | jq '.data.items[]'

# Searches the engines ran while answering: content gaps in their own words
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/ai-visibility/fanouts?type=new" | jq '.data.items[]'

# Add a competitor, or promote one RankSpot already discovered
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Ahrefs", "domain": "ahrefs.com"}' \
  https://api.rankspot.ai/v1/competitors | jq .

# Keywords with no content planned yet, best score first
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/keywords?type=new&sortBy=compositeScore&sortOrder=desc" | jq .

# Semantically related keywords for clustering
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/keywords/<id>/cluster | jq .

# Competitor backlinks by domain authority
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/backlinks?type=competitors&sortBy=domainFromRank&sortOrder=desc" | jq .

# Forum threads to engage with
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/forum-opportunities?status=new" | jq .

# People Also Ask questions
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/people-also-ask?status=new" | jq .

# Plan an article, then generate it
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"type": "write_article", "title": "How to Do Keyword Research", "keywordIds": ["<id1>", "<id2>"]}' \
  https://api.rankspot.ai/v1/actions | jq .
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/actions/<action-id>/generate | jq .

# Get a generated article with full HTML
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/articles/<id> | jq -r .contentHtml
```

## AI Visibility

RankSpot asks your tracked prompts on ChatGPT, Perplexity, Google AI Overview and Google AI Mode on a daily schedule, then analyses the answers.

| Score            | What it measures                                                     |
|------------------|-----------------------------------------------------------------------|
| `visibilityScore`| Share of answers that named your brand, as a mean across platforms    |
| `shareOfVoice`   | Your mentions as a share of all brand mentions                        |
| `citationShare`  | Citations pointing at your domain as a share of all citations         |
| `categoryRank`   | Your position in the brand leaderboard                                |

Every score is nullable, and null means no data in the period, never zero. Check `responsesCounted` before quoting any of them. There is no built-in period comparison: call the summary twice with two periods of equal length.

List endpoints take optional `startDate` and `endDate` as `YYYY-MM-DD`. Omit both for the full history.

## Actions

An action is one piece of planned work. Its `type` decides which target it needs:

| Type                                        | Required target |
|---------------------------------------------|-----------------|
| `write_article`, `add_prompt`, `other`      | none            |
| `update_article`, `publish_article`, `request_indexing` | `articleId` |
| `earn_link`                                 | `backlinkId`    |
| `get_cited`, `reply_thread`, `record_video` | `citationId`    |
| `track_competitor`                          | `competitorId`  |

Only `write_article` produces an article, via `POST /actions/:id/generate`. Closing an action marks the citation or backlink behind it handled, and reopening it puts that evidence back on the worklist.

## Keyword Scoring

RankSpot enriches each keyword with several signals. Sort by the one that fits your goal:

| Signal             | Description                                                      |
|--------------------|------------------------------------------------------------------|
| `compositeScore`   | Weighted blend of opportunity and AI relevance (best default sort) |
| `opportunityIndex` | High volume plus low competition                                  |
| `aiScore`          | Relevance to your specific business                               |
| `searchVolume`     | Monthly search volume                                             |
| `competitionIndex` | 0 to 100 competition level                                        |

## Backlink Analysis

```bash
# Competitor backlinks filtered to a specific domain
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/backlinks?type=competitors&domainFrom=producthunt" | jq .

# Your own backlinks
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/backlinks?type=mine" | jq .
```

Each backlink includes `domainFromRank`, `dofollow`, `backlinkSpamScore`, `anchor` and `firstSeen`.

## API Reference

- Base URL: `https://api.rankspot.ai/v1`
- MCP server: `https://api.rankspot.ai/mcp`
- Interactive docs: `https://api.rankspot.ai/docs`
- Auth: `Authorization: Bearer <api-key>`
- Pagination: `?offset=0&limit=20` (max 100)
- List endpoints wrap results in a `data` envelope. Single resources return the object directly.

## Get an API Key

Sign up at [rankspot.ai](https://rankspot.ai). Your API key is in **Settings, then API Keys** after signup.
