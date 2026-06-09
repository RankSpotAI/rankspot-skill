---
name: rankspot
description: RankSpot is an SEO intelligence platform. Use this skill when the user wants to research competitors, discover and score keywords, analyze backlinks, find forum link-building opportunities, mine "People Also Ask" questions, plan content topics, retrieve AI-generated articles, or analyse Google Search Console performance data via the RankSpot API.
homepage: https://rankspot.ai
metadata: {"clawdbot":{"emoji":"📈","requires":{"env":["RANKSPOT_API_KEY"]}}}
---

## FIRST TIME READING THIS SKILL? STOP AND READ THIS SECTION TO THE USER.

Before running any commands, explain the following to the user:

**What RankSpot does:**
RankSpot is an SEO intelligence platform that gives you programmatic access to your workspace data. Through its API you can: track competitor domains (RankSpot auto-discovers their keywords and backlinks on a background sync), manage and score your keyword list with AI-driven signals, analyse backlinks from competitors and your own domain, surface forum threads as link-building opportunities, mine "People Also Ask" questions from search results, plan content topics from keyword clusters, retrieve AI-generated articles, and pull Google Search Console performance data (clicks, impressions, CTR, average position) for your connected property.

**Setup:**
Generate an API key from **Settings → API Keys** in the RankSpot dashboard. Each key is scoped to a single workspace.

```bash
export RANKSPOT_API_KEY=your_api_key_here
```

No installation required. All commands use `curl` and `jq`.

**Account:**
Sign up or log in at **https://rankspot.ai**. Your API key is in Settings after signup.

---

## Setup

```bash
export RANKSPOT_API_KEY=your_api_key_here
```

| Property        | Value                                                                                                         |
|-----------------|---------------------------------------------------------------------------------------------------------------|
| **name**        | rankspot                                                                                                      |
| **description** | SEO intelligence: competitor tracking, keyword scoring, backlink analysis, forum opportunities, content topics, AI-generated articles |
| **allowed-tools** | Bash(curl:*), Bash(jq:*)                                                                                    |

---

## API Base URL

All endpoints use: `https://api.rankspot.ai/v1`

All requests require: `Authorization: Bearer $RANKSPOT_API_KEY`

Swagger / interactive docs: `https://api.rankspot.ai/docs`

---

## Validate Your API Key

```bash
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/categories | jq .
```

If you get `{"message":"Missing API key"}` or `{"message":"Invalid API key"}`, the key is wrong. Ask the user to check **Settings → API Keys** in the RankSpot dashboard.

---

## Pagination

All list endpoints return:

```json
{
  "data": {
    "total": 120,
    "offset": 0,
    "limit": 20,
    "count": 20,
    "items": [...]
  }
}
```

Query parameters: `offset` (default 0) and `limit` (default 20, max 100).

Trial/inactive subscriptions are capped at `offset=0`, `limit=20`. Active subscriptions can paginate freely up to `limit=100`.

---

## What You Get

| Capability              | Description                                                                              | Endpoints                                    |
|-------------------------|------------------------------------------------------------------------------------------|----------------------------------------------|
| **Competitors**         | Track competitor domains; RankSpot auto-discovers their keywords + backlinks             | `POST/GET/DELETE /competitors`               |
| **Keywords**            | Discover unplanned keywords (high-score opportunities with no content yet), add, cluster | `POST/GET /keywords`, `GET /:id/cluster`     |
| **Backlinks**           | Find competitor backlinks you don't have yet — the core link gap / prospecting workflow  | `GET/PATCH/DELETE /backlinks`                |
| **Forum Opportunities** | Reddit/Quora threads where people discuss your niche — engage to boost GEO visibility   | `POST/GET/PATCH/DELETE /forum-opportunities` |
| **People Also Ask**     | Mine PAA questions from search results for FAQ and content enrichment                   | `GET/PATCH/DELETE /people-also-ask`          |
| **Topics**              | Create content topics from keyword clusters, trigger AI article generation              | `POST/GET/PATCH/DELETE /topics`, `POST /topics/:id/generate` |
| **Articles**            | Retrieve AI-generated articles (full HTML), update metadata, organise by category       | `GET/PATCH/DELETE /articles`                 |
| **Categories**          | Organise topics and articles into named categories                                      | `POST/GET/PATCH/DELETE /categories`          |
| **Search Console**      | Google Search Console performance data — clicks, impressions, CTR, avg position. Requires GSC connected from the RankSpot dashboard. | `POST /gsc/performance` |

---

## Core Workflow — Discover Untapped Keywords → Plan Content

The most common starting point: find keywords already in your workspace that have no content planned for them yet, pick the best ones, and turn them into articles.

```bash
# 1. LIST unplanned keywords — high compositeScore, no topic assigned yet
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/keywords?type=new&sortBy=compositeScore&sortOrder=desc&limit=50" | jq \
  '.data.items[] | {id, keyword, compositeScore, searchVolume, competitionIndex}'

# 2. GET semantically related keywords for your best seed keyword
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/keywords/<seed-id>/cluster | jq '.data[]'

# 3. CREATE a content topic from the seed + its cluster
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "How to Do Keyword Research in 2026",
    "description": "A step-by-step guide targeting beginners.",
    "keywordIds": ["<seed-id>", "<cluster-id-1>", "<cluster-id-2>"]
  }' \
  https://api.rankspot.ai/v1/topics | jq .
# Topic is created in "planned" status. Trigger generation:
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/topics/<topic-id>/generate | jq .

# 4. Poll until status is "generated" (takes 5–10 minutes)
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/topics/<topic-id> | jq '{status: .data.status, articleId: .data.articleId}'

# 5. FETCH the generated article with full HTML
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/articles/<article-id> | jq '{title: .data.title, contentHtml: .data.contentHtml}'
```

## Core Workflow — Competitor Intelligence

```bash
# 1. ADD a competitor (RankSpot begins discovering their keywords + backlinks async)
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Ahrefs", "domain": "ahrefs.com"}' \
  https://api.rankspot.ai/v1/competitors | jq '.data.id'
# Sync runs every 1–2 weeks. For immediate data contact dan@rankspot.ai.

# 2. LIST competitor keywords you haven't planned content for yet
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/keywords?competitorId=<id>&type=new&sortBy=compositeScore&sortOrder=desc&limit=50" | jq \
  '.data.items[] | {keyword, compositeScore, searchVolume}'

# 3. LIST competitor backlinks you don't have — your link gap
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/backlinks?type=competitors&competitorId=<id>&sortBy=domainFromRank&sortOrder=desc" | jq \
  '.data.items[] | {domainFrom, domainFromRank, dofollow, anchor, urlFrom}'

# 4. LIST Reddit/Quora threads where people discuss your niche
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/forum-opportunities?status=new&limit=50" | jq \
  '.data.items[] | {title, url}'
```

---

## Commands Reference

### Competitors

#### Add a Competitor

```bash
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Ahrefs", "domain": "ahrefs.com"}' \
  https://api.rankspot.ai/v1/competitors | jq .
```

**Request body:**
```json
{ "name": "Ahrefs", "domain": "ahrefs.com" }
```

**Response (201):**
```json
{
  "data": {
    "id": "clx...",
    "name": "Ahrefs",
    "domain": "ahrefs.com",
    "createdAt": "2026-05-31T00:00:00.000Z",
    "updatedAt": "2026-05-31T00:00:00.000Z"
  }
}
```

**Important:** After creation, RankSpot begins discovering keywords and backlinks for this domain on the next background sync. Results do not appear immediately — we re-fetch competitors every 2 weeks. If you need to refetch right away, let us know dan@rankspot.ai

**Errors:**
- `400 competitor-limit-exceeded` — your plan's competitor limit has been reached
- `400 competitor-already-exists` — this domain is already tracked

#### List Competitors

```bash
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/competitors?limit=20&offset=0" | jq .
```

#### Delete a Competitor

```bash
curl -s -X DELETE -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/competitors/<id>
```

Returns `204 No Content`. Associated keywords and backlinks are preserved but their `competitorId` is set to null.

---

### Keywords

#### Add Keywords

```bash
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"keywords": ["seo tools", "keyword research", "link building"]}' \
  https://api.rankspot.ai/v1/keywords | jq .
```

**Response (201):** `{ "data": { "count": 3 } }` — duplicates are silently skipped.

#### List Keywords

The most important filter is `type=new` — these are keywords with no planned topic yet. This is where content opportunities live. Always start here before deciding what to write next.

```bash
# MOST IMPORTANT: unplanned keywords — no content assigned yet, best score first
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/keywords?type=new&sortBy=compositeScore&sortOrder=desc&limit=50" | jq .

# Unplanned keywords from a specific competitor
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/keywords?type=new&competitorId=<id>&sortBy=compositeScore&sortOrder=desc" | jq .

# All keywords sorted by composite score
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/keywords?sortBy=compositeScore&sortOrder=desc&limit=50" | jq .

# Search by substring
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/keywords?keyword=seo&sortBy=searchVolume&sortOrder=desc" | jq .
```

**Query parameters:**

| Parameter    | Values                                              | Default        |
|--------------|-----------------------------------------------------|----------------|
| `keyword`    | Substring filter                                    | —              |
| `competitorId` | UUID — omit to see keywords across all competitors; pass an ID to scope to one | — |
| `sortBy`     | `compositeScore`, `opportunityIndex`, `searchVolume`, `competitionIndex`, `aiScore` | `compositeScore` |
| `sortOrder`  | `asc`, `desc`                                       | `desc`         |
| `type`       | `all`, `new`, `planned`, `processed`, `archived`    | `all`          |
| `limit`      | 1–100                                               | 20             |
| `offset`     | ≥0                                                  | 0              |

**Type meanings:**
- `new` — **no topic or article yet** — these are your untapped opportunities
- `planned` — linked to a topic but article not generated yet
- `processed` — has a generated article (done)
- `all` — all non-archived keywords
- `archived` — soft-deleted

**Keyword fields:**

| Field             | Description                                                              |
|-------------------|--------------------------------------------------------------------------|
| `keyword`         | The keyword string                                                       |
| `searchVolume`    | Monthly search volume                                                    |
| `competitionIndex`| 0–100 competition level                                                  |
| `competition`     | `LOW`, `MEDIUM`, `HIGH`                                                  |
| `opportunityIndex`| 0–100 RankSpot-calculated opportunity (high volume + low competition)    |
| `aiScore`         | 0–100 AI-generated relevance score for your business                     |
| `compositeScore`  | 0–100 weighted blend of opportunity + AI score — the primary sort signal |
| `topicIds`        | Topics this keyword is linked to                                         |
| `articleIds`      | Articles generated from topics this keyword belongs to                   |

#### Archive / Unarchive a Keyword

```bash
# Archive (soft-delete)
curl -s -X DELETE -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/keywords/<id>

# Restore
curl -s -X PATCH -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/keywords/<id>/unarchive
```

Both return `204 No Content`.

#### Get Semantic Keyword Cluster

Returns keywords semantically similar to a seed keyword (cosine threshold 0.87, up to 30 results sorted by compositeScore).

```bash
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/keywords/<id>/cluster | jq .
```

**Response (200):**
```json
{
  "data": [
    { "id": "clx...", "keyword": "start a blog for free" },
    { "id": "clx...", "keyword": "how to create a blog" }
  ]
}
```

Returns an empty array if the seed keyword has not been semantically indexed yet (indexing happens on a background schedule).

Use clusters to group related keywords into a single content topic so the generated article covers a broader semantic range.

---

### Backlinks

Backlinks are primarily used for **link gap analysis**: finding high-authority sites that link to your competitors but not to you. These are your link-building prospects. The default `type=competitors` surfaces exactly this — competitor backlinks you can go after.

#### List Backlinks

```bash
# MOST COMMON: competitor backlinks sorted by domain authority — your link prospecting list
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/backlinks?type=competitors&sortBy=domainFromRank&sortOrder=desc" | jq \
  '.data.items[] | {domainFrom, domainFromRank, dofollow, anchor, urlFrom}'

# Scope to a single competitor for focused gap analysis
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/backlinks?type=competitors&competitorId=<id>&sortBy=domainFromRank&sortOrder=desc" | jq .

# Filter by linking domain — check if a specific site links to competitors
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/backlinks?type=competitors&domainFrom=producthunt" | jq .

# Your own backlinks — audit your existing link profile
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/backlinks?type=mine&sortBy=domainFromRank&sortOrder=desc" | jq .
```

**Query parameters:**

| Parameter    | Values                                                    | Default          |
|--------------|-----------------------------------------------------------|------------------|
| `type`       | `competitors`, `processed`, `mine`, `archived`            | `competitors`    |
| `competitorId` | UUID — omit to see backlinks across all competitors; pass an ID to scope to one. Only applies when `type=competitors`. | — |
| `domainFrom` | Substring filter on the linking domain                    | —                |
| `sortBy`     | `domainFromRank`, `firstSeen`, `backlinkSpamScore`        | `domainFromRank` |
| `sortOrder`  | `asc`, `desc`                                             | `desc`           |
| `limit`      | 1–100                                                     | 20               |
| `offset`     | ≥0                                                        | 0                |

**Type meanings:**
- `competitors` — unprocessed competitor backlinks (status=new) — your active link-building prospects
- `processed` — competitor backlinks you've already acted on
- `mine` — your own backlinks (all statuses)
- `archived` — soft-deleted backlinks

**Backlink fields:**

| Field              | Description                                                  |
|--------------------|--------------------------------------------------------------|
| `domainFrom`       | The linking domain                                           |
| `urlFrom`          | The exact linking URL                                        |
| `urlTo`            | The target URL on your / competitor's domain                 |
| `domainTo`         | The target domain                                            |
| `domainFromRank`   | Domain authority score of the linking domain (higher = stronger) |
| `pageFromRank`     | Page-level authority of the linking page                     |
| `rank`             | Combined authority score                                     |
| `dofollow`         | `true` if the link passes link equity                        |
| `backlinkSpamScore`| Spam score (lower = cleaner link profile)                    |
| `anchor`           | Anchor text used for the link                                |
| `firstSeen`        | When the backlink was first detected                         |
| `attributes`       | `rel` attribute values, e.g. `["noopener", "noreferrer"]`   |
| `competitorId`     | Which competitor this belongs to (null = your own backlink)  |
| `status`           | `new` (not yet acted on) or `processed` (already acted on)  |

#### Update Status (Mark as Processed)

Mark a competitor backlink as processed once you've acted on it (submitted the site, sent outreach, etc.). Processed backlinks move out of `type=competitors` into `type=processed` so your prospect list stays clean.

```bash
# Mark as processed
curl -s -X PATCH -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status": "processed"}' \
  https://api.rankspot.ai/v1/backlinks/<id>

# Mark back to new (if you want to re-engage)
curl -s -X PATCH -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status": "new"}' \
  https://api.rankspot.ai/v1/backlinks/<id>
```

Returns `204 No Content`.

**Note:** When a competitor backlink is verified to be linking to your domain too, RankSpot automatically moves it to `type=mine` and resets its status to `new`.

#### Archive / Unarchive a Backlink

```bash
# Archive
curl -s -X DELETE -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/backlinks/<id>

# Restore
curl -s -X PATCH -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/backlinks/<id>/unarchive
```

---

### Forum Opportunities

Forum opportunities are Reddit threads, Quora questions, and community discussions where people are asking about or discussing a problem your customer solves. Engaging in these threads helps with **GEO (Generative Engine Optimization)** — when AI models like ChatGPT and Perplexity synthesise answers, they pull from these conversations. Being present and helpful in relevant discussions increases the chance your brand gets cited.

These are not primarily for link building — they're for brand visibility in the conversations that AI models learn from.

RankSpot surfaces these automatically; you can also add your own.

#### Create a Forum Opportunity

```bash
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "What SEO tools do you actually use day-to-day?",
    "url": "https://reddit.com/r/SEO/comments/abc123/what_seo_tools_do_you_actually_use",
    "competitorId": "<optional-competitor-id>"
  }' \
  https://api.rankspot.ai/v1/forum-opportunities | jq .
```

**Idempotent:** submitting the same URL twice returns the existing record unchanged.

#### List Forum Opportunities

```bash
# New threads to engage with
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/forum-opportunities?status=new&limit=50" | jq \
  '.data.items[] | {title, url}'

# Threads related to a specific competitor
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/forum-opportunities?competitorId=<id>&status=new" | jq .

# Both new and already engaged
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/forum-opportunities?status=new,processed" | jq .
```

#### Update Status (Mark as Processed)

```bash
curl -s -X PATCH -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status": "processed"}' \
  https://api.rankspot.ai/v1/forum-opportunities/<id> | jq .
```

Status values: `new` | `processed`

#### Archive a Forum Opportunity

```bash
curl -s -X DELETE -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/forum-opportunities/<id>
```

---

### People Also Ask

"People Also Ask" questions are discovered automatically by RankSpot from search results for your tracked keywords. They cannot be created via the API — only read and managed.

#### List Questions

```bash
# All new questions
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/people-also-ask?status=new&limit=50" | jq .

# Multiple status values
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/people-also-ask?status=new,processed" | jq .
```

**Question fields:** `id`, `question`, `status` (`new` | `processed`), `postId` (the keyword post that triggered the question).

#### Mark Question as Processed

```bash
curl -s -X PATCH -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status": "processed"}' \
  https://api.rankspot.ai/v1/people-also-ask/<id> | jq .
```

#### Archive a Question

```bash
curl -s -X DELETE -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/people-also-ask/<id>
```

---

### Topics

Topics are content briefs. Once created, a topic sits in `planned` status until you trigger generation via `POST /topics/:id/generate`. Generation is asynchronous and takes **5–10 minutes** — poll `GET /topics/:id` until `status` changes to `generated`. Only `planned` topics can be edited — `generating` and `generated` topics are locked.

#### Create a Topic

```bash
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "How to Do Keyword Research in 2026",
    "description": "A step-by-step guide for beginners targeting informational intent.",
    "additionalInstructions": "Include a comparison table of top tools. Avoid mentioning Semrush.",
    "categoryId": "<optional-category-id>",
    "keywordIds": ["<id1>", "<id2>", "<id3>"]
  }' \
  https://api.rankspot.ai/v1/topics | jq .
```

**Topic fields in request:**

| Field                   | Required | Description                                                    |
|-------------------------|----------|----------------------------------------------------------------|
| `title`                 | Yes      | Topic/article title (max 500 chars)                           |
| `description`           | No       | What the article should cover (max 2000 chars)                |
| `additionalInstructions`| No       | Custom writing instructions for the AI (max 2000 chars)       |
| `categoryId`            | No       | Assign to a category                                          |
| `keywordIds`            | No       | Link existing keyword IDs — broadens semantic coverage        |

#### List Topics

```bash
# All planned topics
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/topics?status=planned" | jq .

# Search by title/description
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/topics?search=keyword+research" | jq .

# Filter by category
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/topics?categoryId=<id>" | jq .
```

Status values: `planned` | `generating` | `generated`

#### Get a Single Topic

```bash
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/topics/<id> | jq .
```

Returns the topic including linked `keywords` array and `articleId` if generation is complete.

#### Update a Topic (planned only)

```bash
curl -s -X PATCH -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Keyword Research Guide 2026: Tools & Tactics",
    "keywordIds": ["<id1>", "<id2>", "<id4>"]
  }' \
  https://api.rankspot.ai/v1/topics/<id> | jq .
```

Passing `keywordIds` **replaces** the full set of linked keywords. Returns `400` if the topic is `generating` or `generated`.

**Note:** Topic titles must be unique per workspace (case-insensitive). `POST /topics` returns `400` if a topic with the same title already exists.

#### Generate an Article from a Topic

Triggers AI article generation. The topic must be in `planned` status. Generation is asynchronous and takes **5–10 minutes** — poll `GET /topics/:id` until `status` is `generated`, then fetch the article via `articleId`.

```bash
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/topics/<id>/generate | jq .
```

**Response (200):**
```json
{ "success": true }
```

Then poll for completion:

```bash
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/topics/<id> | jq '{status: .data.status, articleId: .data.articleId}'
```

**Errors:**
- `400` — topic is not in `planned` status (already generating or generated)
- `403` — article generation limit reached for your plan

#### Delete a Topic

```bash
curl -s -X DELETE -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/topics/<id>
```

Permanent. The generated article (if any) is preserved; its `topicId` link is removed.

---

### Articles

Articles are AI-generated from topics. They cannot be created directly via the API — use `POST /topics/:id/generate` to trigger generation. Generation takes 5–10 minutes. Once complete the article is accessible here. Content and status are managed by RankSpot; only metadata can be updated via the API.

#### List Articles

```bash
# All articles
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/articles?limit=50" | jq .

# Filter by status
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/articles?status=generated" | jq .

# Filter by category
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/articles?categoryId=<id>" | jq .
```

Status values: `draft` | `generating` | `generated`

**Note:** `contentHtml` is omitted from list responses. Fetch a single article to get the full HTML.

#### Get an Article (with full HTML content)

```bash
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/articles/<id> | jq .
```

**Response (200):**
```json
{
  "data": {
    "id": "clx...",
    "title": "How to Do Keyword Research in 2026",
    "description": "A step-by-step guide for beginners.",
    "contentHtml": "<h1>How to Do Keyword Research...</h1><p>...</p>",
    "status": "generated",
    "slug": "how-to-do-keyword-research-2026",
    "coverImageUrl": "https://cdn.example.com/cover.jpg",
    "categoryId": "clx...",
    "createdAt": "2026-05-31T00:00:00.000Z",
    "updatedAt": "2026-05-31T00:00:00.000Z"
  }
}
```

#### Update Article Metadata

```bash
curl -s -X PATCH -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Keyword Research Guide 2026",
    "slug": "keyword-research-guide-2026",
    "coverImageUrl": "https://cdn.example.com/new-cover.jpg",
    "categoryId": "<id>"
  }' \
  https://api.rankspot.ai/v1/articles/<id> | jq .
```

Updatable fields: `title`, `description`, `slug` (must be unique in workspace), `coverImageUrl`, `categoryId`. Article `contentHtml` and `status` cannot be changed via the API.

#### Delete an Article

```bash
curl -s -X DELETE -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/articles/<id>
```

Permanent. Associated topic and keyword links are preserved.

---

### Categories

Categories help organise topics and articles. Names must be unique within a workspace.

#### Create a Category

```bash
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "SEO Guides"}' \
  https://api.rankspot.ai/v1/categories | jq .
```

**Response (201):**
```json
{
  "data": {
    "id": "clx...",
    "name": "SEO Guides",
    "color": "violet",
    "createdAt": "2026-05-31T00:00:00.000Z",
    "updatedAt": "2026-05-31T00:00:00.000Z"
  }
}
```

#### List Categories

```bash
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/categories" | jq .
```

#### Rename a Category

```bash
curl -s -X PATCH -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Keyword Research"}' \
  https://api.rankspot.ai/v1/categories/<id> | jq .
```

#### Delete a Category

```bash
curl -s -X DELETE -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/categories/<id>
```

Topics and articles that were assigned to this category will have their `categoryId` set to null.

---

## Workflow: Competitor Backlink Gap Analysis

Use this workflow when the user asks to:
- "Show me backlinks my competitors have that I don't"
- "Where are my competitors getting links from?"
- "Find link-building prospects from competitor research"

Competitor backlinks are sites that already link to your competitors — they've decided your niche is worth linking to. These are your warmest prospects.

```bash
# Step 1: Add competitors (if not already tracked)
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Competitor A", "domain": "competitor-a.com"}' \
  https://api.rankspot.ai/v1/competitors | jq '.data.id'
# Sync runs every 1–2 weeks. For immediate data contact dan@rankspot.ai.

# Step 2: Pull their highest-authority backlinks — your link gap list
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/backlinks?type=competitors&competitorId=<id>&sortBy=domainFromRank&sortOrder=desc&limit=100" | jq \
  '.data.items[] | {domainFrom, domainFromRank, dofollow, anchor, urlFrom}'

# Step 3: Check if a specific high-value domain links to any competitor
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/backlinks?type=competitors&domainFrom=producthunt" | jq .

# Step 4: Check your own link profile for comparison
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/backlinks?type=mine&sortBy=domainFromRank&sortOrder=desc" | jq \
  '.data.items[] | {domainFrom, domainFromRank, dofollow}'

# Step 5: After contacting or submitting to a site, mark the backlink as processed
curl -s -X PATCH -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status": "processed"}' \
  https://api.rankspot.ai/v1/backlinks/<id>

# Step 6: Review what you've already acted on
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/backlinks?type=processed&sortBy=domainFromRank&sortOrder=desc" | jq \
  '.data.items[] | {domainFrom, domainFromRank, status}'
```

---

## Workflow: Unplanned Keywords → Content Plan

Use this when the user asks to:
- "What keywords don't I have content for yet?"
- "Show me my best keyword opportunities"
- "Group my keywords into content topics"
- "Build a content calendar from my keyword list"

```bash
# Step 1: List unplanned keywords — the content gap, best score first
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/keywords?type=new&sortBy=compositeScore&sortOrder=desc&limit=100" | jq \
  '.data.items[] | {id, keyword, compositeScore, searchVolume, competitionIndex}'

# Step 2: For your best seed keyword, get its semantic cluster
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/keywords/<seed-id>/cluster | jq '.data[]'

# Step 3: Create a topic with the seed + cluster (broader semantic coverage = better article)
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Complete Guide to [Topic]",
    "description": "Covers [seed keyword] and related searches.",
    "keywordIds": ["<seed-id>", "<cluster-id-1>", "<cluster-id-2>"]
  }' \
  https://api.rankspot.ai/v1/topics | jq .

# Step 4: Trigger generation (topic must be "planned")
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/topics/<topic-id>/generate | jq .

# Step 5: Poll until status is "generated" (takes 5–10 minutes)
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/topics/<topic-id> | jq '{status: .data.status, articleId: .data.articleId}'

# Step 5: Fetch the generated article with full HTML
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/articles/<article-id> | jq '{title: .data.title, contentHtml: .data.contentHtml}'
```

---

## Workflow: GEO Forum Engagement Pipeline

Use this when the user asks to:
- "Find Reddit threads where people are looking for tools like mine"
- "What Quora questions should I answer for GEO visibility?"
- "Where do people discuss problems my product solves?"
- "Track my community engagement for AI search visibility"

Being present and helpful in the conversations that AI models (ChatGPT, Perplexity, Gemini) learn from increases the probability your brand gets cited when those models answer related queries.

```bash
# Step 1: List new forum threads to engage with
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/forum-opportunities?status=new&limit=100" | jq \
  '.data.items[] | {id, title, url}'

# Step 2: Add a thread you found manually (Reddit, Quora, Indie Hackers, etc.)
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "What SEO tools do you actually use?", "url": "https://reddit.com/r/SEO/comments/xyz"}' \
  https://api.rankspot.ai/v1/forum-opportunities | jq .

# Step 3: After leaving a helpful reply, mark as processed
curl -s -X PATCH -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status": "processed"}' \
  https://api.rankspot.ai/v1/forum-opportunities/<id> | jq .

# Step 4: Archive threads that aren't relevant
curl -s -X DELETE -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  https://api.rankspot.ai/v1/forum-opportunities/<id>
```

---

## Workflow: PAA Question Mining for FAQ Sections

Use this when the user asks to:
- "Find questions to answer in my articles"
- "What are people asking about my keywords?"
- "Add FAQ sections to improve SEO"

```bash
# Step 1: Get unprocessed PAA questions
curl -s -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  "https://api.rankspot.ai/v1/people-also-ask?status=new&limit=100" | jq '.data.items[] | .question'

# Step 2: Use the questions as FAQ entries in your article or topic
# (these are real search queries — answer them in your content)

# Step 3: Mark questions as processed after incorporating them
curl -s -X PATCH -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"status": "processed"}' \
  https://api.rankspot.ai/v1/people-also-ask/<id> | jq .
```

---

### Search Console

**Prerequisite:** The user must connect Google Search Console from the RankSpot dashboard: **Integrations → Google Search Console → Connect → select a property**. Without this, all requests return `401`.

#### Get Search Performance Data

```bash
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "startDate": "2024-01-01",
    "endDate": "2024-01-31",
    "dimensions": ["query"]
  }' \
  https://api.rankspot.ai/v1/gsc/performance | jq .
```

**Request body:**

| Field                  | Required | Description                                                                 |
|------------------------|----------|-----------------------------------------------------------------------------|
| `startDate`            | Yes      | Start of date range (`YYYY-MM-DD`). Data is available with ~2–3 day delay. |
| `endDate`              | Yes      | End of date range (`YYYY-MM-DD`). Maximum range: 16 months.                |
| `dimensions`           | No       | Array of dimensions to group by. Defaults to `["query"]`. Combine up to 3. |
| `dimensionFilterGroups`| No       | Filter groups to narrow results (see below).                                |
| `startRow`             | No       | Zero-based row offset for pagination (default: 0).                          |
| `rowLimit`             | No       | Max rows to return, 1–25000. Defaults to GSC API default (1000) when omitted. |

**Dimension values:** `query` · `page` · `country` · `device` · `date` · `searchAppearance`

**Response (200):**
```json
{
  "data": {
    "siteUrl": "https://example.com/",
    "startDate": "2024-01-01",
    "endDate": "2024-01-31",
    "dimensions": ["query"],
    "rows": [
      {
        "keys": ["rankspot seo tool"],
        "clicks": 120,
        "impressions": 980,
        "ctr": 12.24,
        "position": 3.2
      }
    ]
  }
}
```

`ctr` is a percentage (e.g. `12.24` = 12.24%). `position` is rounded to 1 decimal.

**Errors:**
- `401 Google Search Console is not connected` — user needs to connect from the dashboard
- `401 No Search Console property selected` — user connected OAuth but hasn't picked a property yet
- `401 Failed to refresh Google Search Console token` — token was revoked; user must reconnect
- `502` — Google Search Console API returned an error (message is forwarded)

#### Common Queries

```bash
# Top search queries — what brings people to your site
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"startDate": "2024-01-01", "endDate": "2024-01-31", "dimensions": ["query"], "rowLimit": 50}' \
  https://api.rankspot.ai/v1/gsc/performance | jq '.data.rows | sort_by(-.clicks)[:10]'

# Top landing pages by clicks
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"startDate": "2024-01-01", "endDate": "2024-01-31", "dimensions": ["page"]}' \
  https://api.rankspot.ai/v1/gsc/performance | jq '.data.rows | sort_by(-.clicks)[:10]'

# Performance by date — track trends over time
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"startDate": "2024-01-01", "endDate": "2024-01-31", "dimensions": ["date"]}' \
  https://api.rankspot.ai/v1/gsc/performance | jq '.data.rows[] | {date: .keys[0], clicks, impressions, ctr, position}'

# Query + page combined — see which pages rank for which queries
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"startDate": "2024-01-01", "endDate": "2024-01-31", "dimensions": ["query", "page"], "rowLimit": 100}' \
  https://api.rankspot.ai/v1/gsc/performance | jq '.data.rows[] | {query: .keys[0], page: .keys[1], clicks, position}'

# Filter by country — UK traffic only
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "startDate": "2024-01-01",
    "endDate": "2024-01-31",
    "dimensions": ["query"],
    "dimensionFilterGroups": [
      { "filters": [{ "dimension": "country", "expression": "gbr" }] }
    ]
  }' \
  https://api.rankspot.ai/v1/gsc/performance | jq '.data.rows | sort_by(-.clicks)[:20]'

# Queries containing a keyword — brand vs non-brand
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "startDate": "2024-01-01",
    "endDate": "2024-01-31",
    "dimensions": ["query"],
    "dimensionFilterGroups": [
      { "filters": [{ "dimension": "query", "expression": "rankspot", "operator": "contains" }] }
    ]
  }' \
  https://api.rankspot.ai/v1/gsc/performance | jq '.data.rows'
```

**Filter `operator` values:** `equals` (default) · `notEquals` · `contains` · `notContains` · `includingRegex` · `excludingRegex`

**Country codes** use ISO 3166-1 alpha-3 (e.g. `usa`, `gbr`, `deu`, `fra`, `ind`). **Device values:** `DESKTOP`, `MOBILE`, `TABLET`.

---

## Workflow: Search Performance Analysis

Use this when the user asks to:
- "What are my top search queries?"
- "Which pages get the most clicks from Google?"
- "How has my search traffic changed over the last month?"
- "Show me my click-through rate and average position"
- "What queries is this page ranking for?"
- "Compare my mobile vs desktop search performance"

**Requires:** Google Search Console connected from **Integrations → Google Search Console** in the RankSpot dashboard.

```bash
# Step 1: Check overall performance for a date range
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"startDate": "2024-01-01", "endDate": "2024-01-31", "dimensions": ["date"]}' \
  https://api.rankspot.ai/v1/gsc/performance | jq '.data.rows[] | {date: .keys[0], clicks, impressions, ctr, position}'

# Step 2: Find top queries driving traffic
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"startDate": "2024-01-01", "endDate": "2024-01-31", "dimensions": ["query"], "rowLimit": 25}' \
  https://api.rankspot.ai/v1/gsc/performance | jq '.data.rows | sort_by(-.clicks)[:10]'

# Step 3: Find queries with high impressions but low CTR — quick-win opportunities
# (ranking well but not getting clicked — title/meta description may need improving)
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"startDate": "2024-01-01", "endDate": "2024-01-31", "dimensions": ["query"], "rowLimit": 500}' \
  https://api.rankspot.ai/v1/gsc/performance | \
  jq '[.data.rows[] | select(.impressions > 100 and .ctr < 3 and .position < 10)] | sort_by(-.impressions)[:10]'

# Step 4: Identify which pages have the best/worst average position
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"startDate": "2024-01-01", "endDate": "2024-01-31", "dimensions": ["page"], "rowLimit": 100}' \
  https://api.rankspot.ai/v1/gsc/performance | jq '.data.rows | sort_by(.position)[:10]'

# Step 5: Drill into a specific page to see what queries it ranks for
curl -s -X POST -H "Authorization: Bearer $RANKSPOT_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "startDate": "2024-01-01",
    "endDate": "2024-01-31",
    "dimensions": ["query"],
    "dimensionFilterGroups": [
      { "filters": [{ "dimension": "page", "expression": "https://example.com/blog/seo-guide" }] }
    ]
  }' \
  https://api.rankspot.ai/v1/gsc/performance | jq '.data.rows | sort_by(-.clicks)'
```

---

## Error Handling

All error responses return JSON.

| Status Code | Meaning                                                                                |
|-------------|----------------------------------------------------------------------------------------|
| 200         | Success                                                                                |
| 201         | Created                                                                                |
| 204         | No content (delete / unarchive)                                                        |
| 400         | Validation error or business rule violation (check the `message` field)               |
| 401         | Missing or invalid API key                                                             |
| 404         | Resource not found or belongs to another workspace                                     |
| 429         | Rate limited — use exponential backoff (wait 10s, retry; if still 429, wait 20s, 40s) |
| 500         | Server error — retry once after 5 seconds                                              |

**400 error types** for competitors:
- `competitor-limit-exceeded` — plan limit reached; inform the user and do not retry
- `competitor-already-exists` — domain already tracked; retrieve the existing record from `GET /competitors`

**Handling 429:**

```bash
# Wait 10s, retry. If still 429, wait 20s, then 40s.
sleep 10
```

Do not hammer the API in a loop. Space sequential requests by at least 1 second.

---

## Rate Limits

| Endpoint group                                    | Limit             |
|---------------------------------------------------|-------------------|
| All endpoints (global)                            | 5,000 req / 60s per API key |
| `POST /gsc/performance`                           | 100 req / 60s per API key |

---

## Tips

- **Start with `type=new` keywords.** These are the unplanned opportunities — keywords in the workspace with no content assigned yet. Always check here first before deciding what to write next.
- **Competitor sync runs every 1–2 weeks.** After `POST /competitors`, data will not appear immediately. If the user needs it right away, direct them to contact **dan@rankspot.ai** for a manual sync.
- **Sort by `compositeScore` first.** It blends opportunity score with AI relevance — the best single signal for which keywords to prioritise.
- **Competitor backlinks = your link prospecting list.** Sites linking to competitors have already decided this niche is link-worthy. `type=competitors` is the default for good reason — it only shows unprocessed prospects so the list stays actionable.
- **Mark backlinks as processed to track outreach.** After contacting a site or submitting a listing, `PATCH /backlinks/<id>/update` with `{"status": "processed"}` moves it out of your prospect list into `type=processed`. Once RankSpot verifies the link points to your domain, it automatically moves to `type=mine`.
- **Use `/cluster` before creating a topic.** Grouping semantically related keywords into one topic produces better, broader-ranking articles.
- **Forum threads are for GEO, not just links.** Helpful participation in Reddit/Quora threads that match your customer's problems puts your brand into the conversations AI models learn from.
- **PAA questions = free FAQ content.** Add them as structured FAQ sections in articles to capture featured snippet positions.
- **Forum opportunities age quickly.** Process `new` items regularly — threads go stale once the discussion moves on.
- **Archive, don't delete keywords.** Archiving preserves the data; a competitor's keywords cannot be recovered once the competitor is deleted.
- **`contentHtml` is only in single-article responses.** The list endpoint omits it to keep payloads small — always fetch by ID to get the full content.
- **Trial subscriptions are capped.** `offset=0` and `limit≤20` apply to non-active subscriptions. Upgrade at https://rankspot.ai if you hit these limits.
- **GSC data has a 2–3 day delay.** `endDate` of yesterday will often return no data — use a date at least 3 days in the past.
- **GSC `dimensions` defaults to `["query"]`.** Omit it to get query-level breakdown, or pass `["date"]` for trend analysis or `["page"]` for page-level performance.
- **High impressions + low CTR + position < 10 = quick wins.** These queries are visible but not getting clicked — improving the page title or meta description can yield immediate traffic gains without changing rankings.
- **Country codes are alpha-3.** `usa`, `gbr`, `deu`, `fra`, `ind` — not the more common alpha-2 (`us`, `gb`, etc.).
