# Script Writer AI Agent

> A multi-node AI agent system for social media intelligence, content creation, and competitive strategy. Input an Instagram or YouTube profile and get full analytics and competitor insights. [LLM powered]


## 📋 Table of Contents

- [System Overview](#system-overview)
- [Architecture Diagram](#architecture-diagram)
- [Agent Nodes](#agent-nodes)
  - [Node 1 — Profile Analyzer](#node-1--profile-analyzer)
  - [Node 2 — Video Transcription](#node-2--video-transcription)
  - [Node 3 — Script Generator](#node-3--script-generator)
  - [Node 4 — Translation](#node-4--translation)
  - [Node 5 — SEO Caption Generator](#node-5--seo-caption-generator)
  - [Node 6 — Hot Topics](#node-6--hot-topics)
  - [Node 7 — Trending Hashtags](#node-7--trending-hashtags)
  - [Node 8 — Competitor Analysis](#node-8--competitor-analysis)
- [ProPeers Full Lookup Pipeline](#propeers-full-lookup-pipeline)
- [Azure Infrastructure](#azure-infrastructure)
- [Cost Model](#cost-model)
- [API Contracts](#api-contracts)
- [Environment Variables](#environment-variables)
- [Deployment](#deployment)

---

## System Overview

```
User Input (handle / URL)
        │
        ▼
┌─────────────────────┐
│ API Management Node │  ← Auth, rate limiting, routing
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│    Orchestrator     │  ← Fan-out / fan-in across nodes
└──────────┬──────────┘
           │
    ┌──────┴───────┐
    │              │
    ▼              ▼
[Apify]   [instagram-reel-scraper]
[yt-dlp]       [AI Search]
[yt]            [OpenAI]
    │              │
    └──────┬───────┘
           ▼
    ┌─────────────┐
    │  Mongo DB   │  ← Results stored +  Trending Results
    └─────────────┘
           │
           ▼
      Response → Frontend
```

**Tech stack at a glance:**

| Layer | Technology |
|---|---|
| Orchestration | Azure Durable Functions (fan-out / fan-in) |
| LLM | Azure OpenAI — GPT-4o + GPT-4o-mini + Whisper |
| Scraping | Apify (`instagram-reel-scraper`, `youtube-scraper`) |
| Media processing | yt-dlp |
| Storage | Azure Blob Storage (media)|
| Discovery | Azure AI Search |

---

## Architecture Diagram

```
                         ┌──────────────────────────────────────────────
                         │              AZURE BOUNDARY                   
                         │                                               
  ┌──────────┐           │  ┌─────────┐     ┌────────────────────────┐  
  │ Frontend │──────────►│  │  APIM   │────►│  Durable Orchestrator  │  
  └──────────┘           │  └─────────┘     └───────────┬────────────┘  
                         │                              │                
                         │              ┌───────────────┼────────────┐  
                         │              │               │            │  
                         │              ▼               ▼            ▼  
                         │     ┌──────────────┐  ┌──────────┐  ┌──────
                         │     │Profile       │  │  Apify   │  │ Azure-AI 
                         │     │Analyzer Node │  │ Scraper  │  │  Search
                         │     └──────┬───────┘  └────┬─────┘  └──┬───
                         │            │               │           │    
                         │            ▼               ▼           │    
                         │     ┌──────────────┐  ┌──────────┐     │    
                         │     │   yt-dlp     │  │  Blob &  │     │    
                         │     │              │  │ Storage  │     │    
                         │     └──────┬───────┘  └────┬─────┘     │    
                         │            │               │           │   
                         │            ▼               ▼           ▼    
                         │     ┌──────────────────────────────────────┐ 
                         │     │         Azure LLM Service            │ 
                         │     │   GPT-4o · GPT-4o-mini · Whisper     │ 
                         │     └──────────────────┬───────────────────┘ 
                         └──────────────────────────────────────────────
```

---

## Agent Nodes

### Node 1 — Profile Analyzer

**Purpose:** Retrieve the last 30 days of activity from an Instagram or YouTube account.

**Inputs:**

| Field | Type | Required | Description |
|---|---|---|---|
| `platform` | `string` | ✅ | `"instagram"` or `"youtube"` |
| `handle` | `string` | ✅ | Username or full profile URL |
| `date_range_days` | `number` | ❌ | Default `30` |

**Outputs:**

```json
{
  "profile": {
    "handle": "@propeersindia",
    "platform": "instagram",
    "followers": 48200,
    "following": 312,
    "bio": "...",
    "verified": false
  },
  "activity_summary": {
    "posts_last_30d": 14,
    "avg_views": 32400,
    "avg_likes": 1820,
    "avg_comments": 94,
    "engagement_rate": 3.77,
    "best_posting_days": ["Tuesday", "Thursday"],
    "best_posting_hours": [18, 20]
  },
  "content": [
    {
      "id": "reel_001",
      "type": "reel",
      "caption": "...",
      "views": 84000,
      "likes": 3100,
      "comments": 142,
      "music": "Original audio - creator",
      "mp4_url": "https://...",
      "posted_at": "2025-05-10T18:32:00Z"
    }
  ]
}
```

**Azure services:** Apify actors (parallel) → Azure Durable Functions
**LLM:** GPT-4o for activity summary narrative only  

---

### Node 2 — Video Transcription

**Purpose:** Download a video from a reel or YouTube URL, extract audio, and transcribe it.

**Inputs:**

| Field | Type | Required | Description |
|---|---|---|---|
| `video_url` | `string` | ✅ | Instagram .mp4 URL or YouTube video URL |
| `language_hint` | `string` | ❌ | ISO 639-1 code e.g. `"hi"`, `"en"` |

**Outputs:**

```json
{
  "transcript": "So today I want to talk about the one thing that changed my content strategy...",
  "language_detected": "en",
  "duration_seconds": 47,
  "segments": [
    { "start": 0.0, "end": 3.2, "text": "So today I want to talk about" },
    { "start": 3.2, "end": 8.5, "text": "the one thing that changed my content strategy" }
  ],
  "cached": false,
  "video_id": "sha256:abc123"
}
```

**Processing pipeline:**

```
Instagram URL → direct .mp4 download → Azure Blob Storage
YouTube URL   → yt-dlp (Azure Container Instance) → Azure Blob Storage
                      ↓
             OpenAI Whisper API (whisper-1)
                  .mp3 → transcript JSON
                      
```

> **Note:** Instagram reels return a direct `.mp4` URL via Apify — no yt-dlp needed.  
> YouTube does **not** return a direct `.mp4` — yt-dlp is required server-side.  
> Shares and saves are **never** available on any platform — no service can provide them.

---

### Node 3 — Script Generator

**Purpose:** Generate a ready-to-record video script from a topic, hook style, tone, and language.

**Inputs:**

| Field | Type | Required | Description |
|---|---|---|---|
| `topic` | `string` | ✅ | e.g. `"3 morning habits that doubled my productivity"` |
| `hook_style` | `string` | ✅ | `"question"` / `"shock"` / `"story"` / `"bold_claim"` |
| `tone` | `string` | ✅ | `"educational"` / `"funny"` / `"motivational"` / `"conversational"` |
| `language` | `string` | ✅ | ISO 639-1 code |
| `duration_seconds` | `number` | ❌ | Target video length. Default `60` |
| `source_transcript` | `string` | ❌ | Existing transcript for style-matching |

**Outputs:**

```json
{
  "hook": "You've been wasting the first 30 minutes of your day — here's what to do instead.",
  "body": [
    { "section": "Problem", "script": "Most people wake up and immediately check their phone...", "duration_est": 12 },
    { "section": "Solution 1", "script": "Habit one: no screens for the first 20 minutes...", "duration_est": 15 },
    { "section": "Solution 2", "script": "Habit two: write down three things you will finish today...", "duration_est": 15 }
  ],
  "cta": "Follow for one productivity tip every Tuesday.",
  "total_duration_est": 52,
  "word_count": 210
}
```

**System prompt pattern:**

```
You are an expert short-form video scriptwriter for {platform}.
Write a {duration}s script on: "{topic}"
Hook style: {hook_style}
Tone: {tone}
Language: {language}

Structure: Hook (3–5s) → Body (value delivery) → CTA (5s)
Output valid JSON only. No markdown fences.
```

---

### Node 4 — Translation

**Purpose:** Translate scripts, captions, hashtags, or transcripts between languages while preserving tone and platform idioms.

**Inputs:**

| Field | Type | Required | Description |
|---|---|---|---|
| `text` | `string` or `string[]` | ✅ | Content to translate |
| `source_language` | `string` | ❌ | Auto-detected if omitted |
| `target_language` | `string` | ✅ | ISO 639-1 code |
| `content_type` | `string` | ✅ | `"script"` / `"caption"` / `"hashtags"` / `"transcript"` |

**Outputs:**

```json
{
  "translated": "आज मैं आपको वो एक चीज़ बताऊंगा जिसने मेरी कंटेंट स्ट्रेटेजी बदल दी...",
  "source_language_detected": "en",
  "target_language": "hi",
  "tone_preserved": true,
  "notes": "Adapted 2 English idioms to Hindi equivalents"
}
```

---

### Node 5 — SEO Caption Generator

**Purpose:** Generate a platform-optimized caption with SEO keywords, a call to action, and hashtags.

**Inputs:**

| Field | Type | Required | Description |
|---|---|---|---|
| `script` | `string` | ✅ | Video script or topic description |
| `platform` | `string` | ✅ | `"instagram"` / `"youtube"` |
| `cta_type` | `string` | ✅ | `"follow"` / `"link_in_bio"` / `"comment"` / `"save"` |
| `language` | `string` | ✅ | ISO 639-1 code |
| `niche` | `string` | ✅ | e.g. `"fitness"`, `"personal_finance"`, `"tech"` |

**Platform character limits enforced:**

| Platform | Caption limit | Hashtag placement |
|---|---|---|
| Instagram | 2,200 chars | End of caption or first comment |
| YouTube | 5,000 chars | Description body + timestamp links |

**Outputs:**

```json
{
  "caption": "Your morning routine is costing you 2 hours of peak focus every day. Here's the fix 👇\n\n...",
  "seo_keywords": ["morning routine", "productivity habits", "focus tips"],
  "cta": "Save this for tomorrow morning ↓",
  "character_count": 847,
  "hashtag_block": "#morningroutine #productivitytips #morninghabits ...",
  "platform_compliant": true
}
```


---

### Node 6 — Hot Topics

**Purpose:** Surface real-time trending topics in a niche with content angle suggestions. 

**Inputs:**

| Field | Type | Required | Description |
|---|---|---|---|
| `niche` | `string` | ✅ | e.g. `"fitness"`, `"crypto"`, `"ai tools"` |
| `platform` | `string` | ❌ | Filters relevance by platform audience |
| `region` | `string` | ❌ | ISO 3166-1 alpha-2, e.g. `"IN"`, `"US"` |
| `time_window` | `string` | ❌ | `"24h"` / `"7d"` / `"30d"` |

**Outputs:**

```json
{
  "topics": [
    {
      "topic": "AI tools for content creators",
      "trend_score": 94,
      "search_volume_change": "+340%",
      "subtopics": ["AI video editing", "auto-captioning", "script AI"],
      "content_angles": [
        "5 AI tools that replaced my entire editing workflow",
        "I used AI for 30 days — here's what actually worked"
      ],
      "competitor_coverage": "low",
      "source_urls": ["https://..."]
    }
  ],
  "generated_at": "2025-05-21T10:00:00Z"
}
```

---

### Node 7 — Trending Hashtags in niche

**Purpose:** Generate a stratified set of hashtags (mega / mid / micro) for maximum reach and discoverability.

**Inputs:**

| Field | Type | Required | Description |
|---|---|---|---|
| `niche` | `string` | ✅ | e.g. `"personal_finance"` |
| `platform` | `string` | ✅ | `"instagram"` / `"youtube"` |


**Hashtag strategy (30-tag output):**

| Tier | Followers on tag | Count | Purpose |
|---|---|---|---|
| Mega | 10M+ | 5 | Broad visibility |
| Mid | 100k–1M | 15 | Targeted reach |
| Micro | < 100k | 10 | High engagement, low competition |

**Outputs:**

```json
{
  "hashtags": {
    "mega": ["#fitness", "#workout", "#health"],
    "mid": ["#fitnessmotivation", "#gymmotivation", "#fitfam"],
    "micro": ["#mumbaifit", "#homeworkoutindia", "#fitnesstipstelugu"]
  },
  "flat_list": "#fitness #workout #health ...",
  "banned_tags_removed": ["#follow4follow"],
  "estimated_combined_reach": "47M posts",
  "refreshed_at": "2025-05-21T09:00:00Z"
}
```

---

### Node 8 — Competitor Analysis

**Purpose:** Analyze competitor profiles side-by-side and generate strategic recommendations.

**Inputs:**

| Field | Type | Required | Description |
|---|---|---|---|
| `competitor_handles` | `string[]` | ✅ | handles (Instagram or YouTube) |
| `your_handle` | `string` | ❌ | Include yourself for relative comparison |
| `platform` | `string` | ✅ | `"instagram"` or `"youtube"` |
| `depth` | `string` | ❌ | `"quick"` (last 10 posts) / `"deep"` (last 30 days) |

**Outputs:**

```json
{
  "comparison": [
    {
      "handle": "@competitor_a",
      "followers": 120000,
      "avg_views": 45000,
      "engagement_rate": 4.2,
      "posting_frequency": "1.2/day",
      "top_content_pillars": ["tutorials", "product reviews", "behind the scenes"]
    }
  ],
  "content_gaps": [
    "None of the competitors cover beginner-level Hindi content",
    "No competitor posts on Saturdays despite high audience activity"
  ],
  "recommendations": [
    "Post educational carousels on Saturdays to capture untapped engagement window",
    "Create Hindi-subtitled reels — zero competitor coverage in this niche"
  ],
  "your_rank": {
    "by_engagement": 2,
    "by_avg_views": 3,
    "by_posting_frequency": 1
  }
}
```


---

## ProPeers Full Lookup Pipeline

One full lookup = Instagram (10 reels) + YouTube (10 videos) + transcription + script.

```
Step 1 → Profile input
         User pastes handle/URL into ProPeers UI
         APIM validates → routes to Durable Orchestrator

Step 2 → Parallel scraping (fan-out)
         ├─ Apify instagram-reel-scraper → last 10 reels
         │    Returns: caption, views, likes, music, direct .mp4 URL
         └─ Apify youtube-scraper → last 10 videos/shorts
              Returns: title, views, likes, comments, thumbnail, date

Step 3 → Video download (conditional)
         ├─ Instagram: .mp4 URL available → download directly to Blob Storage
         └─ YouTube: no direct URL → yt-dlp on Azure Container Instance

Step 4 → Audio extraction
         ffmpeg on Azure Container Instance: .mp4 → .mp3
         Output written to Blob Storage (hot tier)
         .mp4 retained 24h then purged

Step 5 → Transcription
         OpenAI Whisper API (whisper)
         .mp3 → timestamped transcript JSON
         Cached in Redis by video ID hash (TTL: 7 days)

Step 6 → Script generation
         Azure OpenAI GPT-4o
         Transcript → structured script (hook + body + CTA)
         Simultaneously triggers SEO caption + hashtag nodes

Step 7 → Fan-in + delivery
         Durable Orchestrator aggregates all results
         Written to Cosmos DB (retained 30 days)
         CDN-cached JSON response → ProPeers frontend
```

### What is and isn't available

| Data point | Available | Notes |
|---|---|---|
| Views | ✅ | Both platforms |
| Likes | ✅ | Both platforms |
| Comments | ✅ | Both platforms |
| Thumbnails | ✅ | Both platforms |
| Music / audio info | ✅ | Instagram only |
| Direct .mp4 URL | ✅ | Instagram only |
| Publish date | ✅ | Both platforms |
| Shares | ❌ | Never available — no service can provide this |
| Saves | ❌ | Never available — no service can provide this |
| Direct YouTube .mp4 URL | ❌ | Requires yt-dlp server-side |

---

## Azure Infrastructure
---

## Cost Model

### Cost per full lookup

| Step | Service | Cost |
|---|---|---|
| Instagram 10 reels | Apify | $0.010 |
| YouTube 10 videos | Apify | $0.024 |
| Whisper transcription (1 video) | OpenAI Whisper | $0.006 |
| GPT-4o script generation | Azure OpenAI | $0.003 |
| Azure compute (Functions + ACI) | Azure | ~$0.002 |
| **Total per lookup** | | **~$0.045** |

### Monthly scale

| Lookups/month | Apify cost | Whisper + GPT-4o | Apify subscription | **Total** |
|---|---|---|---|---|---|
| 100 | $3.40  | $29 | **~$29** |
| 500 | $17.00 | $29 | **~$29** |
| 1,000 | $34.00 | $29 | **~$29** |

> **Note:** Apify Starter ($29/mo) includes $35 in credits — covers ~1,000 full lookups before paying overage.





### Example: full pipeline trigger

```bash
curl -X POST https://api.propeers.in/v1/agent/pipeline/full \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "instagram_handle": "@example",
    "youtube_handle": "UCxxxxxxx",
    "transcribe_count": 1,
    "generate_script": true
  }'
```

## Node Dependency Graph

```
                    ┌──────────────────┐
                    │ Profile Analyzer │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
    ┌──────────────┐  ┌──────────┐  ┌────────────────┐
    │Video Transcr.│  │Hot Topics│  │Competitor Anal.│
    └──────┬───────┘  └────┬─────┘  └────────────────┘
           │               │
           ▼               ▼
    ┌─────────────┐  ┌────────────┐
    │   Script    │  │ Trending   │
    │             │  │ Hashtags   │
    └──────┬──────┘  └────────────┘
           │
    ┌──────┴────────┐
    ▼               ▼
┌────────┐   ┌──────────────┐
│Translate│  │ SEO Caption  │
│ Node   │   │ Generator    │
└────────┘   └──────────────┘
```

---

## Limitations & Known Constraints

- **Shares and saves** are never available from any platform or scraping service
- **YouTube direct .mp4 URL** is not available — yt-dlp required server-side
- **Instagram private profiles** cannot be scraped — public only
- **Apify rate limits** apply — concurrent actor runs limited by plan
- **Whisper accuracy** drops below ~80% for heavy regional accents or music-heavy audio

