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

**Azure services:** Azure OpenAI GPT-4o, Azure Functions  
**Model:** `gpt-4o` with structured JSON output mode

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

**Azure services:** Azure OpenAI GPT-4o-mini, Azure Functions  
**Model:** `gpt-4o-mini` (sufficient quality, ~70% cheaper than GPT-4o)

---

### Node 5 — SEO Caption Generator

**Purpose:** Generate a platform-optimized caption with SEO keywords, a call to action, and hashtags.

**Inputs:**

| Field | Type | Required | Description |
|---|---|---|---|
| `script` | `string` | ✅ | Video script or topic description |
| `platform` | `string` | ✅ | `"instagram"` / `"youtube"` / `"tiktok"` |
| `cta_type` | `string` | ✅ | `"follow"` / `"link_in_bio"` / `"comment"` / `"save"` |
| `language` | `string` | ✅ | ISO 639-1 code |
| `niche` | `string` | ✅ | e.g. `"fitness"`, `"personal_finance"`, `"tech"` |

**Platform character limits enforced:**

| Platform | Caption limit | Hashtag placement |
|---|---|---|
| Instagram | 2,200 chars | End of caption or first comment |
| YouTube | 5,000 chars | Description body + timestamp links |
| TikTok | 2,200 chars | Inline within caption |

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

**Azure services:** Azure OpenAI GPT-4o, Azure AI Search (keyword index), Azure Functions

---

### Node 6 — Hot Topics

**Purpose:** Surface real-time trending topics in a niche with content angle suggestions. Fully clickable — select a topic to get full context and subtopics.

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

**Azure services:** Bing Search API v7, Azure AI Search, Azure OpenAI GPT-4o, Redis (TTL: 1h)

---

### Node 7 — Trending Hashtags

**Purpose:** Generate a stratified set of hashtags (mega / mid / micro) for maximum reach and discoverability.

**Inputs:**

| Field | Type | Required | Description |
|---|---|---|---|
| `niche` | `string` | ✅ | e.g. `"personal_finance"` |
| `platform` | `string` | ✅ | `"instagram"` / `"tiktok"` / `"youtube"` |
| `topic` | `string` | ✅ | Specific video topic |
| `region` | `string` | ❌ | For region-specific hashtags |

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

**Azure services:** Bing Search API, Apify hashtag actor, Azure OpenAI GPT-4o-mini, Redis (TTL: 24h)

---

### Node 8 — Competitor Analysis

**Purpose:** Analyze 2–5 competitor profiles side-by-side and generate strategic recommendations.

**Inputs:**

| Field | Type | Required | Description |
|---|---|---|---|
| `competitor_handles` | `string[]` | ✅ | 2–5 handles (Instagram or YouTube) |
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

**Azure services:** Apify scrapers (parallel fan-out), Azure Durable Functions, Azure OpenAI GPT-4o, Cosmos DB  
**Cost:** Apify credits × number of competitors × depth

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
         OpenAI Whisper API (whisper-1)
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

### Compute

| Service | Usage |
|---|---|
| **Azure Container Apps** | Hosts each node as an independent microservice |
| **Azure Durable Functions** | Orchestration, fan-out/fan-in, retry logic, state |
| **Azure Container Instances** | Ephemeral yt-dlp and ffmpeg jobs (billed per second) |
| **Azure API Management** | Single gateway — auth, rate limiting, routing, versioning |

### AI / LLM

| Service | Model | Used for |
|---|---|---|
| **Azure OpenAI** | GPT-4o | Script gen, competitor analysis, topic angles, captions |
| **Azure OpenAI** | GPT-4o-mini | Translation, hashtags, summary text (70% cheaper) |
| **Azure OpenAI** | Whisper-1 | Audio transcription |
| **Azure AI Search** | — | Hot topics vector index, keyword search |
| **Bing Search API v7** | — | Live trending data, topic research |

> Your existing OpenAI API keys can be onboarded into Azure OpenAI Service. Traffic stays within your Azure tenant — no data leaves your VNet.

### Storage & Data

| Service | Usage | Retention |
|---|---|---|
| **Azure Blob Storage (hot)** | .mp4 and .mp3 temp files | 6–24h then auto-deleted |
| **Azure Cosmos DB (NoSQL)** | Lookup results, profile snapshots | 30 days |
| **Azure Cache for Redis** | Apify results (24h), transcripts (7d) | Per-key TTL |
| **Azure CDN** | Media previews served to frontend | CDN edge cache |

### Messaging & Ops

| Service | Usage |
|---|---|
| **Azure Service Bus** | Async job queue for long-running media jobs |
| **Azure Monitor + App Insights** | Telemetry, latency tracking, error alerts |
| **Azure Key Vault** | Apify API key, OpenAI key, Bing key — never in code |
| **Azure Active Directory B2C** | ProPeers user authentication |

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

| Lookups/month | Apify cost | Whisper + GPT-4o | Apify subscription | Azure infra | **Total** |
|---|---|---|---|---|---|
| 100 | $3.40 | $0.90 | $29 | ~$10 | **~$43** |
| 500 | $17.00 | $4.50 | $29 | ~$15 | **~$66** |
| 1,000 | $34.00 | $9.00 | $29 | ~$20 | **~$92** |
| 5,000 | $170.00 | $45.00 | $199 | ~$40 | **~$454** |
| 10,000 | $340.00 | $90.00 | $199 | ~$60 | **~$689** |

> **Note:** Apify Starter ($29/mo) includes $35 in credits — covers ~1,000 full lookups before paying overage.

### Cost reduction strategies

| Strategy | Saves |
|---|---|
| Redis cache Apify results (24h per handle) | $0.034/repeat lookup |
| Redis cache transcripts (7d per video ID) | $0.006/repeat video |
| Lazy transcription (on-demand, not upfront) | Up to 90% of Whisper cost |
| GPT-4o-mini for translation, hashtags, captions | ~70% LLM cost reduction |
| Blob TTL auto-delete .mp4/.mp3 after 6h | Significant storage savings |
| Azure Reserved Instances for Container Apps | Up to 40% compute savings |

---

## API Contracts

All nodes expose REST endpoints behind Azure API Management.

### Base URL
```
https://api.propeers.in/v1/agent
```

### Authentication
```
Authorization: Bearer <propeeers_user_jwt>
X-Api-Version: 2025-05
```

### Endpoint map

| Node | Method | Endpoint |
|---|---|---|
| Profile Analyzer | `POST` | `/profile/analyze` |
| Video Transcription | `POST` | `/video/transcribe` |
| Script Generator | `POST` | `/content/script` |
| Translation | `POST` | `/content/translate` |
| SEO Caption | `POST` | `/content/caption` |
| Hot Topics | `GET` | `/discover/topics` |
| Trending Hashtags | `POST` | `/discover/hashtags` |
| Competitor Analysis | `POST` | `/competitor/analyze` |

### Example: profile analyze request

```bash
curl -X POST https://api.propeers.in/v1/agent/profile/analyze \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "platform": "instagram",
    "handle": "@propeersindia",
    "date_range_days": 30
  }'
```

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

---

## Environment Variables

```env
# Azure OpenAI
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com
AZURE_OPENAI_API_KEY=your_key_here
AZURE_OPENAI_DEPLOYMENT_GPT4O=gpt-4o
AZURE_OPENAI_DEPLOYMENT_MINI=gpt-4o-mini
AZURE_OPENAI_DEPLOYMENT_WHISPER=whisper

# Apify
APIFY_API_TOKEN=your_apify_token

# Bing Search
BING_SEARCH_API_KEY=your_bing_key
BING_SEARCH_ENDPOINT=https://api.bing.microsoft.com/v7.0

# Azure Storage
AZURE_BLOB_CONNECTION_STRING=DefaultEndpointsProtocol=https;...
AZURE_BLOB_CONTAINER_MEDIA=media-temp

# Azure Cosmos DB
COSMOS_ENDPOINT=https://your-account.documents.azure.com:443
COSMOS_KEY=your_cosmos_key
COSMOS_DATABASE=propeers
COSMOS_CONTAINER_LOOKUPS=lookups

# Azure Cache for Redis
REDIS_CONNECTION_STRING=your-cache.redis.cache.windows.net:6380,password=...

# Azure AI Search
AZURE_SEARCH_ENDPOINT=https://your-search.search.windows.net
AZURE_SEARCH_API_KEY=your_search_key
AZURE_SEARCH_INDEX=topics-index

# Durable Functions Storage
DURABLE_TASK_HUB_NAME=PropeersPipeline
AzureWebJobsStorage=DefaultEndpointsProtocol=https;...
```

> **Never commit secrets to git.** All values above should be stored in **Azure Key Vault** and referenced via managed identity in production.

---

## Deployment

### Prerequisites

- Azure CLI installed and logged in
- Node.js 20+ (for Azure Functions)
- Docker (for Container Apps + ACI jobs)
- Apify account with Starter plan ($29/mo minimum)
- Azure subscription with OpenAI access approved

### Quick start

```bash
# 1. Clone the repo
git clone https://github.com/your-org/social-media-ai-agent.git
cd social-media-ai-agent

# 2. Create Azure resources
az group create --name propeers-rg --location eastus
az deployment group create \
  --resource-group propeers-rg \
  --template-file infra/main.bicep \
  --parameters @infra/parameters.prod.json

# 3. Deploy Durable Functions orchestrator
cd src/orchestrator
func azure functionapp publish propeers-orchestrator

# 4. Build and push Container Apps
az acr build --registry propeerscr --image nodes/profile-analyzer:latest ./nodes/profile-analyzer
az containerapp update --name profile-analyzer --resource-group propeers-rg \
  --image propeerscr.azurecr.io/nodes/profile-analyzer:latest

# 5. Set secrets in Key Vault (never in env files)
az keyvault secret set --vault-name propeers-kv --name ApifyToken --value "your_token"
az keyvault secret set --vault-name propeers-kv --name AzureOpenAiKey --value "your_key"
```

### Recommended node deployment order

1. Profile Analyzer (core — everything depends on it)
2. Video Transcription (independent)
3. Hot Topics (independent, uses Bing)
4. Trending Hashtags (light dependency on topics)
5. Script Generator (depends on transcription output)
6. SEO Caption Generator (depends on script)
7. Translation (wraps all other nodes)
8. Competitor Analysis (depends on Profile Analyzer × N)

---

## Node Dependency Graph

```
                    ┌──────────────────┐
                    │  Profile Analyzer │
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
    │Script Generator│  │Trending    │
    │             │  │Hashtags    │
    └──────┬──────┘  └────────────┘
           │
    ┌──────┴────────┐
    ▼               ▼
┌────────┐   ┌──────────────┐
│Translate│   │SEO Caption   │
│ Node   │   │Generator     │
└────────┘   └──────────────┘
```

---

## Limitations & Known Constraints

- **Shares and saves** are never available from any platform or scraping service
- **YouTube direct .mp4 URL** is not available — yt-dlp required server-side
- **Instagram private profiles** cannot be scraped — public only
- **Apify rate limits** apply — concurrent actor runs limited by plan
- **Whisper accuracy** drops below ~80% for heavy regional accents or music-heavy audio
- **Hot Topics** data has ~15-minute latency from Bing API

---

## License

MIT — see [LICENSE](./LICENSE)

---

*Built for ProPeers · Architecture version 1.0 · May 2025*
