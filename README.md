# RankSpot SEO Skill

Programmatic SEO intelligence for AI agents. Track competitors, score keywords, analyse backlinks, find forum link-building opportunities, mine "People Also Ask" questions, and manage AI-generated content — all through the RankSpot API.

## Quick Start

```bash
export RANKSPOT_API_KEY=your_key_here
```

No installation required. The skill uses `curl` and `jq`.

Get your API key from **Settings → API Keys** in the [RankSpot dashboard](https://rankspot.ai).

## Capabilities

| Capability              | What it does                                                              |
|-------------------------|---------------------------------------------------------------------------|
| **Competitors**         | Add competitor domains; RankSpot auto-discovers their keywords + backlinks |
| **Keywords**            | Track, score, filter, and semantically cluster keywords                   |
| **Backlinks**           | Analyse competitor and your own backlinks with domain rank + spam score   |
| **Forum Opportunities** | Track forum threads for link-building outreach                            |
| **People Also Ask**     | Mine PAA questions for FAQ and content enrichment                         |
| **Topics**              | Create content briefs that trigger AI article generation                  |
| **Articles**            | Retrieve AI-generated articles with full HTML content                     |
| **Categories**          | Organise topics and articles into named groups                            |

## Commands

```bash
# Add a competitor
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Ahrefs", "domain": "ahrefs.com"}' \
  https://api.rankspot.ai/v1/competitors | jq .

# List keywords sorted by composite score
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/keywords?sortBy=compositeScore&sortOrder=desc" | jq .

# Get semantically related keywords for clustering
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/keywords/<id>/cluster | jq .

# List competitor backlinks by domain authority
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/backlinks?type=competitors&sortBy=domainFromRank&sortOrder=desc" | jq .

# List forum link-building opportunities
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/forum-opportunities?status=new" | jq .

# List People Also Ask questions
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/people-also-ask?status=new" | jq .

# Create a content topic (triggers AI article generation)
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "How to Do Keyword Research", "keywordIds": ["<id1>", "<id2>"]}' \
  https://api.rankspot.ai/v1/topics | jq .

# Get a generated article with full HTML
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/articles/<id> | jq '.data.contentHtml'
```

## Keyword Scoring

RankSpot enriches each keyword with multiple signals — sort by the one that fits your goal:

| Signal           | Description                                            |
|------------------|--------------------------------------------------------|
| `compositeScore` | Weighted blend of opportunity + AI relevance (best default sort) |
| `opportunityIndex` | High volume + low competition = high opportunity     |
| `aiScore`        | AI-generated relevance for your specific business      |
| `searchVolume`   | Monthly search volume                                  |
| `competitionIndex` | 0–100 competition level                              |

## Backlink Analysis

```bash
# Competitor backlinks filtered to a specific domain
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/backlinks?type=competitors&domainFrom=producthunt" | jq .

# Your own backlinks
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/backlinks?type=mine" | jq .
```

Each backlink includes `domainFromRank`, `dofollow`, `backlinkSpamScore`, `anchor`, and `firstSeen`.

## API Reference

- Base URL: `https://api.rankspot.ai/v1`
- Interactive docs: `https://api.rankspot.ai/docs`
- Auth: `Authorization: Bearer <api-key>`
- Pagination: `?offset=0&limit=20` (max 100)

## Get an API Key

Sign up at [rankspot.ai](https://rankspot.ai). Your API key is in **Settings → API Keys** after signup.
