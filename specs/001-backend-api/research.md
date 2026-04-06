# Research: AI Conversation API

This document outlines the research and decisions made to resolve the "NEEDS CLARIFICATION" items in the implementation plan.

## 1. Language and Framework

- **Decision**: NestJS (TypeScript).
- **Rationale**: NestJS is a powerful and progressive Node.js framework for building efficient, reliable, and scalable server-side applications. Its modular architecture, built on TypeScript, makes it highly suitable for this project. It has first-class support for WebSockets through its gateways feature, which is perfect for our real-time communication needs. The strong typing provided by TypeScript helps in maintaining a clean and robust codebase, and its dependency injection system simplifies managing services for STT, TTS, and LLM integrations.
- **Alternatives Considered**:
  - **FastAPI (Python)**: Initially considered, but the team has a preference for the TypeScript ecosystem. While Python has a strong AI/ML ecosystem, the core logic of this service is I/O-bound orchestration, which Node.js excels at.
  - **Express/Fastify (Node.js)**: While both are excellent and fast, NestJS provides a more structured, opinionated framework out-of-the-box, which helps in maintaining consistency and scalability as the project grows.

## 2. Unified Audio Solution: Gemini Native Audio

- **Decision**: Google Gemini Native Audio Model (All-in-One Solution)
- **Rationale**: After initial research, we discovered that Google's Gemini API now offers a **Native Audio Model** (`gemini-2.5-flash-native-audio-preview-09-2025`) that combines Speech-to-Text, Text-to-Speech, and Conversational AI in a single, integrated service. This represents a significant architectural simplification over using separate providers.

### Benefits of Gemini Native Audio:

1. **Single Provider Integration**: Only requires one API key and one SDK (`@google/genai`)
2. **Lower Latency**: No need to chain multiple services (STT → LLM → TTS), reducing round-trip time
3. **Better Context Understanding**: The model processes audio input and generates audio responses within the same context, leading to more natural conversations
4. **Cost Efficiency**: Consolidated API calls and simplified billing
5. **Architectural Simplicity**: One service implementation instead of three separate adapters
6. **Real-time Streaming**: Native WebSocket support for bidirectional audio streaming
7. **High Quality German Support**: Built-in German language support for both transcription and synthesis

### Technical Specifications:

**For Real-time Conversations:**
- Model: `gemini-2.5-flash-native-audio-preview-09-2025`
- Input: PCM audio at 16kHz, 16-bit, mono
- Output: Both text transcription and PCM audio at 24kHz, 16-bit, mono
- Connection: WebSocket for bidirectional streaming

**For High-Quality TTS:**
- Model: `gemini-2.5-flash-preview-tts`
- Voice: "Kore" (natural German voice)
- Format: PCM audio output, base64 encoded

This consolidation aligns perfectly with our constitutional principle of "Lean Maintainability" by minimizing dependencies while maintaining all required functionality.

## 3. Speech-to-Text Transcription via Gemini Live API

- **Decision**: Use Gemini Live API's built-in transcription feature
- **Rationale**: Gemini Live API provides native support for both input and output audio transcription through its `BidiGenerateContentSetup` configuration. This eliminates the need for a separate STT service and provides synchronous transcription alongside the audio stream.

### Transcription Configuration

The Gemini Live WebSocket API supports enabling transcription during session setup:

```json
{
  "setup": {
    "model": "models/gemini-2.5-flash-native-audio-preview-09-2025",
    "inputAudioTranscription": {},
    "outputAudioTranscription": {}
  }
}
```

### How It Works

1. **Input Audio Transcription (`inputAudioTranscription`)**: When enabled, Gemini Live transcribes the user's audio input as it's streamed. The backend receives `BidiGenerateContentTranscription` messages containing the transcribed text of what the user said.

2. **Output Audio Transcription (`outputAudioTranscription`)**: When enabled, Gemini Live provides transcriptions of the AI's audio responses. This allows the client to display what the AI is saying in text form alongside the audio playback.

### Server Message Structure

Transcriptions are delivered via `BidiGenerateContentServerMessage`:

```json
{
  "serverContent": {
    "inputTranscription": {
      "text": "Hallo, wie geht es dir?"
    }
  }
}
```

```json
{
  "serverContent": {
    "outputTranscription": {
      "text": "Mir geht es gut, danke! Wie kann ich dir heute helfen?"
    }
  }
}
```

### Benefits of Native Transcription

1. **Synchronous Delivery**: Transcriptions arrive in real-time alongside the audio stream
2. **No Additional API Calls**: Eliminates the need for a separate STT service like Google Cloud Speech-to-Text
3. **Language Alignment**: Transcription matches the configured audio language (German/English)
4. **Lower Latency**: No round-trip to an external service; transcription is part of the same WebSocket stream
5. **Cost Efficiency**: Bundled with the Gemini Live API usage, no separate billing

### Implementation Approach

1. Enable `inputAudioTranscription` and `outputAudioTranscription` in the session setup
2. Parse incoming `serverContent` messages for `inputTranscription` and `outputTranscription` fields
3. Emit `input_transcription` and `output_transcription` WebSocket events to connected clients
4. Persist transcriptions as `Message` entities in the `ConversationRepository`

### Transcription Storage

Transcriptions are saved to the conversation history as `Message` entities:
- **User transcriptions**: `sender: 'user'`, `textContent: <transcribed text>`
- **AI transcriptions**: `sender: 'ai'`, `textContent: <transcribed text>`

This allows clients to retrieve the full conversation transcript for review and ensures a complete audit trail of the conversation.

## 4. Cloud Provider Selection

This section evaluates which cloud provider is most cost-effective for this project while providing comparable quality and service breadth. The project requires: container hosting (with WebSocket support for real-time audio), managed PostgreSQL, a key-value store with TTL for auth tokens, a load balancer, and a container registry.

### 4.1 Requirements Specific to This Project

| Requirement | Why |
|---|---|
| **WebSocket support** | Real-time bidirectional audio streaming (Gemini Live). Connections last up to 5 minutes per session. Eliminates pure request-based serverless options unless they support long-lived connections. |
| **Managed PostgreSQL** | Relational data store for users, conversations, messages (decided in §4.3 below). |
| **Key-value store with TTL** | Ephemeral auth tokens (magic links 15 min, refresh tokens 7 days). Needs auto-expiry. |
| **Container hosting** | NestJS app in Docker. Single service, not microservices. |
| **Terraform support** | IaC is already decided. All major clouds supported. |
| **Gemini API** | Google's API, but accessed via API key over HTTPS — no cloud-specific integration needed. Works equally from any cloud. |

### 4.2 Provider Comparison

#### AWS (current plan)

| Service | AWS Product | Monthly Cost (US East, on-demand) |
|---|---|---|
| Container hosting | ECS Fargate (0.25 vCPU, 0.5 GB) | ~$9.00 |
| Load balancer | ALB | ~$16.20 (fixed) + LCU charges |
| Managed PostgreSQL | RDS db.t4g.micro + 20 GB gp3 | ~$14.40 |
| Auth token store | DynamoDB On-Demand | ~$0 (free tier) |
| Container registry | ECR | ~$0.50 |
| **Total** | | **~$40–42/month** |

**Strengths**: Broadest service catalog, DynamoDB's perpetual free tier + native TTL, ECS Fargate is simple and has no control plane cost, mature Terraform provider, enterprise-grade.

**Weaknesses**: ALB alone costs ~$16/month (fixed hourly charge) which dominates the bill for a low-traffic app. Free tier for RDS expires after 12 months.

#### GCP

| Service | GCP Product | Monthly Cost (us-central1, on-demand) |
|---|---|---|
| Container hosting | Cloud Run (1 always-on instance, 0.25 vCPU, 512 MiB) | ~$14.25 |
| Load balancer | Included with Cloud Run (no separate LB needed) | $0 |
| Managed PostgreSQL | Cloud SQL db-f1-micro (shared, 0.6 GiB) + 10 GB SSD | ~$9.40 |
| Auth token store | Firestore (Native mode) | ~$0 (free tier: 1 GiB storage, 50K reads/day) |
| Container registry | Artifact Registry | ~$0.50 |
| **Total** | | **~$24–26/month** |

**Cloud Run WebSocket note**: Cloud Run supports WebSockets with a max request timeout of 60 minutes. Since conversation sessions have a 5-minute inactivity timeout, this is sufficient. Cloud Run also handles autoscaling to zero when idle — no traffic = no compute cost.

**Strengths**: Cloud Run eliminates the need for a separate load balancer (built-in HTTPS + WebSocket termination), Cloud SQL shared-core is cheaper than RDS micro, Firestore has a generous perpetual free tier with TTL support, $300 free credits for new accounts, potential Gemini API latency advantage (same Google network).

**Weaknesses**: Cloud Run instance-based billing at ~$14/month is slightly more than ECS Fargate for always-on. Cloud SQL shared-core (db-f1-micro) is not SLA-backed. Smaller Terraform provider ecosystem than AWS.

#### Azure

| Service | Azure Product | Monthly Cost (East US, on-demand) |
|---|---|---|
| Container hosting | Container Apps (0.25 vCPU, 0.5 GiB, consumption plan) | ~$9.00 |
| Load balancer | Included with Container Apps | $0 |
| Managed PostgreSQL | Azure Database for PostgreSQL Flexible Server (Burstable B1ms, 1 vCPU, 2 GiB) | ~$12.40 |
| Auth token store | Cosmos DB (serverless, request units) | ~$0–1 (very low at this scale) |
| Container registry | ACR Basic | ~$5.00 |
| **Total** | | **~$27–30/month** |

**Strengths**: Container Apps includes load balancing and supports WebSockets natively, competitive managed PostgreSQL pricing, Cosmos DB serverless has TTL support.

**Weaknesses**: Smallest ecosystem for small dev projects, ACR Basic has a fixed $5/month floor, Cosmos DB's RU pricing model is harder to predict, least community momentum for indie/startup projects.

#### Fly.io

| Service | Fly.io Product | Monthly Cost |
|---|---|---|
| Container hosting | Shared CPU 1×, 256 MB | ~$1.94 |
| Load balancer | Included (Anycast + Fly Proxy) | $0 |
| Managed PostgreSQL | Fly Postgres (1 shared CPU, 256 MB, 1 GB disk) | ~$3.82 |
| Auth token store | None — use PostgreSQL with scheduled cleanup | $0 |
| Container registry | Built-in (fly deploy) | $0 |
| **Total** | | **~$6–8/month** |

**Strengths**: Dramatically cheapest option, excellent WebSocket support (edge-based Anycast routing), built-in deploy pipeline, no separate LB/registry costs, generous free allowance ($5/month of compute).

**Weaknesses**: Fly Postgres is **not a managed database** — it's a self-managed Postgres in a VM. No automatic failover, no managed backups beyond what you script. No DynamoDB equivalent (must use Postgres for tokens). Limited Terraform support (community provider, not official). Smaller company with less enterprise reliability history. No TTL-enabled key-value store.

#### Neon (PostgreSQL-only, serverless alternative)

Worth noting as a complement: **Neon** offers serverless PostgreSQL that scales to zero.

| Plan | Cost | Includes |
|---|---|---|
| Free | $0/month | 0.5 GB storage, 190 compute hours/month, autoscaling, scale-to-zero |
| Launch | $19/month | 10 GB storage, 300 compute hours/month |

If used instead of RDS/Cloud SQL, Neon's free tier could eliminate the Postgres cost entirely at early stage. However, it would need to be paired with a container hosting solution (Fly.io + Neon could be ~$2–6/month total). TypeORM works with Neon since it's standard PostgreSQL.

### 4.3 Cost Summary

| Provider | Monthly Cost (on-demand) | WebSocket Support | Managed DB Quality | Terraform Support | Free Tier |
|---|---|---|---|---|---|
| **Fly.io** | **~$6–8** | Excellent | Self-managed | Community | $5/month compute |
| **GCP** | **~$24–26** | Good (60 min limit) | Managed (shared, no SLA) | Official | $300 credits |
| **Azure** | **~$27–30** | Good | Managed | Official | $200 credits |
| **AWS** | **~$40–42** | Good | Managed (SLA-backed) | Official (best) | 12 months |

### 4.4 Recommendation: AWS

Despite being the most expensive option at ~$40/month, **AWS is recommended** for the following reasons:

1. **The cost difference is modest in absolute terms**: The gap between AWS ($42) and GCP ($25) is ~$17/month (~$204/year). For Fly.io the gap is ~$35/month. These are meaningful percentages but small absolute numbers for a production application.

2. **ALB cost can be eliminated**: The biggest AWS cost driver is the ALB at ~$16/month. This can be avoided by placing the NestJS container behind an **API Gateway** (HTTP API at $1/million requests) or by using **ECS with a public IP directly** behind CloudFront for simple setups. This would bring AWS to ~$24–26/month, on par with GCP.

3. **DynamoDB is unmatched for the token use case**: No other provider offers a key-value store with native TTL, automatic item deletion, 3-AZ durability, and a perpetual free tier that covers this use case at $0. On GCP, Firestore comes close but has more complex pricing. On Fly.io, you'd need to DIY token expiry in Postgres.

4. **RDS is the gold standard for managed PostgreSQL**: Unlike Cloud SQL's shared-core (no SLA) or Fly.io's self-managed Postgres (no automatic failover/backups), RDS provides automated backups, point-in-time recovery, and optional Multi-AZ — critical for user data and auth credentials.

5. **Terraform provider maturity**: The AWS Terraform provider is the most mature, best-documented, and most widely used. This matters for a solo/small-team project where IaC debugging time is costly.

6. **Ecosystem breadth**: As the project grows (email delivery via SES, CDN via CloudFront, monitoring via CloudWatch, secrets via Secrets Manager), AWS provides everything in one account. Fly.io and GCP would require stitching together services from multiple providers.

**Cost optimization path**: Start with ECS Fargate + API Gateway (instead of ALB) to bring the monthly cost to ~$24/month, competitive with GCP. If traffic grows, the 12-month RDS free tier covers the initial period, and Reserved Instances can reduce ongoing costs by 30–55%.

## 5. Data Storage Strategy: Token Store & User Data

This section evaluates cost-effective storage options for two distinct data categories:
1. **Ephemeral auth tokens** (magic links, refresh tokens) — short-lived, TTL-driven, simple key-value lookups.
2. **User-related data** (User, LearningProfile, UsageStats, Preferences, Passkeys) — relational, long-lived, queried together.

### 5.1 Redis (ElastiCache) vs DynamoDB for Magic Links & Refresh Tokens

#### Access Patterns

| Pattern | Magic Link Tokens | Refresh Tokens |
|---|---|---|
| Write | Create on request (hash → metadata) | Create on login/refresh |
| Read | Lookup by token hash | Lookup by token hash |
| Delete | Mark consumed / auto-expire | Revoke / auto-expire |
| TTL | 15 minutes | 7 days |
| Item size | ~200 bytes | ~200 bytes |
| Volume (early stage) | ~10–50/day | ~10–50/day |

#### Option A: ElastiCache Serverless for Valkey

| Dimension | Detail |
|---|---|
| Data stored | $0.0837/GB-hour. Minimum 100 MB (Valkey) = **~$6.10/month minimum** |
| Compute (ECPUs) | $0.00227/million ECPUs. At low volume (~1K requests/day), effectively **< $0.01/month** |
| **Total estimate** | **~$6.10/month** (dominated by 100 MB minimum storage charge) |
| Native TTL | Yes — `SET key value EX seconds`, automatic expiry |
| Latency | Sub-millisecond |
| Durability | Snapshots + AOF replication; not a primary data store |
| Ops overhead | None (serverless) |

#### Option B: ElastiCache On-Demand Node (cache.t4g.micro)

| Dimension | Detail |
|---|---|
| Node cost | ~$0.016/hr = **~$11.52/month** |
| Native TTL | Yes |
| Latency | Sub-millisecond |
| Free tier | 750 hrs/month of cache.t3.micro for 12 months |
| Ops overhead | Must choose node size; no auto-scaling |

#### Option C: DynamoDB On-Demand

| Dimension | Detail |
|---|---|
| Writes | $1.25/million WRUs (US East). At ~1K writes/day → **< $0.04/month** |
| Reads | $0.25/million RRUs. At ~1K reads/day → **< $0.01/month** |
| Storage | $0.25/GB/month. Token data < 1 MB → **< $0.01/month** |
| **Total estimate** | **< $0.10/month** (well within free tier: 25 GB storage included) |
| Native TTL | Yes — `TimeToLiveSpecification` auto-deletes expired items at no cost |
| Latency | Single-digit milliseconds |
| Durability | Fully durable, replicated across 3 AZs |
| Ops overhead | None (fully serverless, true scale-to-zero) |

#### Comparison Summary

| Factor | ElastiCache Serverless (Valkey) | ElastiCache Node | DynamoDB On-Demand |
|---|---|---|---|
| Monthly cost (low traffic) | ~$6.10 (100 MB min) | ~$11.52 (t4g.micro) | **< $0.10** |
| Scale-to-zero | No (100 MB floor) | No (always-on node) | **Yes** |
| Native TTL | Yes | Yes | Yes |
| Latency | Sub-ms | Sub-ms | Single-digit ms |
| Durability | Multi-AZ replication | Depends on config | **3-AZ by default** |
| Free tier | Credits-based | 750 hrs t3.micro/12 mo | **25 GB + request allowance** |
| Additional infra | Yes (new service) | Yes (new service) | Yes (new service) |

#### Recommendation: DynamoDB On-Demand

DynamoDB is the clear winner for this use case:
- **Cost**: At early-stage volumes, DynamoDB is effectively free (well within the perpetual free tier of 25 GB storage). ElastiCache has a minimum ~$6/month floor.
- **TTL**: DynamoDB's TTL feature automatically deletes expired items with no additional cost or code — ideal for magic link tokens (15 min) and refresh tokens (7 days).
- **Durability**: Tokens are critical auth data. DynamoDB provides automatic 3-AZ replication, whereas Redis is primarily an in-memory cache and risks data loss on node failure unless carefully configured.
- **Latency tradeoff**: Single-digit ms vs sub-ms is negligible for auth token lookups that happen once per login/refresh.
- **Operational simplicity**: No node sizing, no cluster management, no minimum data charges that dwarf actual usage.

### 5.2 AWS RDS PostgreSQL vs AWS DynamoDB for All Application Data

Nothing is deployed to production yet, so both options are equally viable as the primary data store. This comparison evaluates replacing PostgreSQL entirely with DynamoDB for all data (users, conversations, messages, passkeys), not just adding user tables to an existing RDS instance.

#### Data Characteristics

| Aspect | Detail |
|---|---|
| Entities | User, UserLearningProfile, UserUsageStats, UserPreferences, UserPasskey, Conversation, Message |
| Relationships | User 1:1 LearningProfile, 1:1 UsageStats, 1:1 Preferences, 1:N Passkeys, 1:N Conversations; Conversation 1:N Messages |
| Access patterns | Full profile fetch (user + 3 sub-resources), conversation history listing, message retrieval by conversation |
| Query complexity | Relational JOINs for profile, potential future queries by email, learning level, date ranges, aggregations |
| Volume (early stage) | Hundreds to low thousands of users; thousands of conversations/messages |
| Item size | ~1–2 KB per user profile; ~0.5 KB per message |

#### Option A: DynamoDB On-Demand (replacing PostgreSQL entirely)

**Cost (US East, on-demand):**

| Dimension | Detail |
|---|---|
| Writes | $1.25/million WRUs. At early-stage volumes → **< $0.10/month** |
| Reads | $0.25/million RRUs. At early-stage volumes → **< $0.05/month** |
| Storage | $0.25/GB/month. All app data < 1 GB for months → **< $0.25/month** |
| **Total estimate** | **< $0.50/month** (well within perpetual free tier: 25 GB storage, 25 WRU/sec, 25 RRU/sec) |
| Free tier | **Perpetual** — 25 GB storage, 25 RRU/sec, 25 WRU/sec (not time-limited) |

**Capabilities:**

| Dimension | Detail |
|---|---|
| Data modeling | Single-table design (recommended) or multiple tables; must denormalize relationships |
| JOINs | **Not supported** — must fetch related items with multiple queries or denormalize into composite items |
| Transactions | TransactWriteItems / TransactGetItems (max 100 items, 4 MB per transaction) |
| Schema enforcement | Application-level only — no DB-level CHECK, FK, or UNIQUE constraints |
| Schema evolution | No migrations needed; schema-less items |
| Query flexibility | Primary key + sort key + GSIs only; no ad-hoc SQL, no aggregations |
| Latency | Single-digit milliseconds |
| Durability | **3-AZ replication by default** |
| ORM integration | No TypeORM support — requires `@aws-sdk/client-dynamodb` or `@aws-sdk/lib-dynamodb` |
| Local development | DynamoDB Local (Docker) or LocalStack |

**DynamoDB Single-Table Design for User Data:**
```
PK                  SK                      Attributes
USER#<userId>       PROFILE                 displayName, email, createdAt, ...
USER#<userId>       LEARNING_PROFILE        learningLevel, nativeLanguage, ...
USER#<userId>       USAGE_STATS             totalConversationCount, ...
USER#<userId>       PREFERENCES             timezone, uiLanguage
USER#<userId>       PASSKEY#<passkeyId>     credentialId, publicKey, counter, ...
USER#<userId>       CONV#<convId>           startTime, endTime
CONV#<convId>       MSG#<msgId>             sender, textContent, timestamp
GSI1: email → USER  (for lookup by email)
```

#### Option B: AWS RDS PostgreSQL

**Cost (US East, on-demand, Single-AZ):**

| Dimension | Detail |
|---|---|
| Instance | db.t4g.micro: **~$12.10/month** ($0.016/hr × 730 hrs) |
| Storage | gp3, 20 GB: **~$2.30/month** ($0.115/GB/month) |
| Backup | Included up to 1× instance storage |
| **Total estimate** | **~$14.40/month** |
| Free tier | **12 months only** — 750 hrs db.t3.micro + 20 GB storage + 20 GB backup |
| After free tier | Full on-demand pricing applies (~$14+/month minimum) |

**Capabilities:**

| Dimension | Detail |
|---|---|
| Data modeling | Natural relational schema with foreign keys, constraints, indexes |
| JOINs | **Native** — fetch full user profile in a single query |
| Transactions | **Full ACID** — no item/size limits |
| Schema enforcement | **DB-level** CHECK constraints (CEFR levels, enum values), NOT NULL, UNIQUE (email), FK cascades |
| Schema evolution | Liquibase migrations (structured, version-controlled) |
| Query flexibility | **Full SQL** — ad-hoc queries, aggregations, window functions, CTEs |
| Latency | Low single-digit milliseconds (within same VPC) |
| Durability | Single-AZ (default); Multi-AZ available at 2× cost |
| ORM integration | **TypeORM** — entities, repositories, relations, eager loading |
| Local development | Docker `postgres:17` (already configured in docker-compose.yml) |

#### Comparison Summary

| Factor | DynamoDB On-Demand | RDS PostgreSQL |
|---|---|---|
| Monthly cost (early stage) | **< $0.50** (perpetual free tier) | ~$14.40 (free tier expires after 12 mo) |
| Scale-to-zero | **Yes** (pay per request) | No (always-on instance) |
| Data model fit | Poor (relational data forced into NoSQL) | **Excellent** (natural relational schema) |
| Full profile fetch | Multiple queries or denormalized blob | **Single JOIN query** |
| Schema enforcement | Application-level only | **DB-level constraints, CHECK, FK** |
| Query flexibility | Key/index only | **Full SQL** |
| ORM / TypeORM | Not supported (custom SDK layer) | **Native support** |
| Local dev | DynamoDB Local or LocalStack | **Docker postgres (already set up)** |
| Migrations | None needed (schema-less) | Liquibase (already configured) |
| Infrastructure | Fully managed, no sizing | Must choose instance size |
| Durability | **3-AZ by default** | Single-AZ default; Multi-AZ at 2× cost |
| Free tier duration | **Perpetual** | 12 months only |
| Operational overhead | **None** | Patches, backups, sizing, scaling |

#### Cost Projection

| Timeframe | DynamoDB | RDS PostgreSQL (db.t4g.micro) |
|---|---|---|
| Month 1–12 (free tier) | ~$0 | ~$0 (free tier covers t3.micro) |
| Month 13–24 | ~$0 (still in perpetual free tier) | **~$14.40/month** = ~$173/year |
| Year 2+ ongoing | ~$0–$1/month at low volumes | **~$173/year** minimum |
| At 10K users, moderate traffic | ~$5–15/month | ~$14–30/month (may need db.t4g.small) |
| At 100K users, high traffic | ~$50–150/month | ~$50–200/month (db.r6g series) |

**Key cost insight**: DynamoDB is dramatically cheaper at low-to-moderate volumes due to true pay-per-request pricing and the perpetual free tier. RDS has a fixed floor of ~$14/month regardless of traffic. At high scale, costs converge.

#### Tradeoff Analysis

**Arguments for DynamoDB (all-in):**
- **~$170/year savings** in the post-free-tier steady state vs RDS minimum
- True scale-to-zero: zero cost when the app has no traffic
- Zero operational overhead: no patching, no instance sizing, no storage scaling
- Auth tokens (magic links, refresh tokens) are already going to DynamoDB — single data platform
- 3-AZ durability by default without paying 2× for Multi-AZ

**Arguments for RDS PostgreSQL:**
- **Data model fit**: User data is inherently relational (1:1, 1:N relationships with constraints). DynamoDB requires denormalization or multi-query patterns that increase application complexity
- **Schema safety**: DB-level CHECK constraints (CEFR levels), UNIQUE (email), FK cascades catch bugs that application-level validation misses
- **Query flexibility**: Full SQL enables future features like admin dashboards, analytics, searching users by criteria, aggregation reports — all impossible in DynamoDB without exporting to another service
- **TypeORM ecosystem**: Repository pattern, eager/lazy loading, relation decorators, query builders — all lost with DynamoDB. Must write a custom data access layer
- **Local dev simplicity**: Docker Postgres is already configured. DynamoDB Local works but adds Docker complexity and has behavioral differences from production
- **Conversation/Message data**: Messages with conversation history, ordered retrieval, and relational integrity (FK to conversation, FK to user) are a natural relational fit. Single-table design for this is complex
- **Team familiarity**: SQL is universally understood; DynamoDB single-table design has a steep learning curve

#### Recommendation: Hybrid — PostgreSQL (RDS) for relational data, DynamoDB for ephemeral tokens

Despite DynamoDB's significant cost advantage at low volumes, **PostgreSQL is recommended for all relational application data** (users, profiles, conversations, messages, passkeys) for the following reasons:

1. **Application complexity cost exceeds infrastructure cost**: The ~$14/month RDS cost is far less expensive than the engineering time to build and maintain a custom DynamoDB data access layer, handle denormalization, and work around missing JOINs/constraints. This is a solo/small-team project where developer time is the scarcest resource.

2. **Data integrity**: User email uniqueness, CEFR level validation, FK cascades on user deletion — these are critical for an auth-gated app. Relying solely on application-level validation for these invites subtle bugs.

3. **Future flexibility**: Full SQL means adding a "find users by learning level" admin query, a "top streaks" leaderboard, or conversation analytics is trivial. With DynamoDB, each new access pattern requires a new GSI or data export pipeline.

4. **The cost gap narrows with scale**: At the point where DynamoDB's cost advantage matters (~$170/year savings at low volume), the app likely has revenue or can optimize RDS costs with Reserved Instances (up to 55% savings = ~$77/year, making the gap only ~$93/year).

5. **DynamoDB still wins for tokens**: Magic link tokens and refresh tokens are purely ephemeral, key-value lookups with TTL. DynamoDB's native TTL auto-cleanup and scale-to-zero pricing make it ideal for this specific use case without the overhead of managing a full RDS instance just for tokens.

### 5.3 Overall Architecture Decision

| Data Category | Storage | Rationale |
|---|---|---|
| User, LearningProfile, UsageStats, Preferences | **PostgreSQL (RDS)** | Relational data with constraints, JOINs, TypeORM |
| UserPasskey | **PostgreSQL (RDS)** | FK to User, counter updates, relational integrity |
| Conversation, Message | **PostgreSQL (RDS)** | FK relationships, ordered queries, history retrieval |
| Magic Link Tokens | **DynamoDB** | Ephemeral key-value, native TTL auto-cleanup, scale-to-zero |
| Refresh Tokens | **DynamoDB** | Ephemeral key-value, native TTL, same table as magic links |

This hybrid approach uses each engine where it excels: PostgreSQL for relational data with integrity constraints, DynamoDB for ephemeral tokens with automatic expiry. The ~$14/month RDS cost is justified by the significant reduction in application complexity and the preservation of full SQL flexibility for future features.

## 6. Secrets Management: JWT Signing Key Storage

The backend issues JWT access tokens signed with a private key (or symmetric secret). This key must be stored securely, never committed to source control, and be accessible to every running container instance at startup. This section evaluates how to store and retrieve it.

### 6.1 Requirements

| Requirement | Detail |
|---|---|
| **Security** | The JWT signing key is the single most sensitive credential in the auth system. A leaked key lets an attacker forge access tokens for any user. Must be encrypted at rest and in transit. |
| **Access pattern** | Read once at application startup (or on first JWT operation), then cached in memory for the process lifetime. Extremely low API call volume (~1 call per container start). |
| **Rotation support** | Must support updating the key without redeploying code. Ideally supports versioning so old tokens signed with a previous key can still be verified during a grace period. |
| **Multi-environment** | Separate keys per environment (dev, staging, prod). |
| **Terraform-manageable** | Must be provisionable via Terraform/Terragrunt alongside the rest of the infrastructure. |

### 6.2 Options Considered

#### Option A: Environment variable via ECS Task Definition

| Dimension | Detail |
|---|---|
| How it works | Store the key as a plaintext env var in the ECS task definition or in a `.env` file baked into the container. |
| Cost | $0 |
| Security | **Poor**. The key appears in plaintext in the ECS console, CloudFormation templates, Terraform state, and container inspection. Anyone with ECS `DescribeTaskDefinition` permission can read it. |
| Rotation | Requires a new task definition revision and service redeployment. |
| Verdict | **Rejected**. Unacceptable for a credential that can forge user identities. |

#### Option B: AWS Systems Manager Parameter Store (SecureString)

| Dimension | Detail |
|---|---|
| How it works | Store the key as a `SecureString` parameter encrypted with KMS. Read it at startup via the SSM SDK or ECS's native `valueFrom` secret injection. |
| Cost | Standard tier: free for up to 10,000 parameters. Advanced tier: $0.05/parameter/month. API calls: free up to 10,000/month (standard), then $0.05/10,000. |
| Security | Good — encrypted at rest with KMS, IAM-controlled access, audit trail via CloudTrail. |
| Rotation | Manual — no built-in rotation. Must update the parameter value and restart containers (or re-read on a schedule). |
| SDK | `@aws-sdk/client-ssm` — `GetParameter` with `WithDecryption: true`. |
| Terraform | `aws_ssm_parameter` resource with `type = "SecureString"`. |
| Verdict | Viable but lacks automatic rotation and versioned secret retrieval. |

#### Option C: AWS Secrets Manager

| Dimension | Detail |
|---|---|
| How it works | Store the key as a secret in Secrets Manager. Read it at startup via the SDK. Supports automatic rotation via Lambda, secret versioning (`AWSCURRENT`, `AWSPREVIOUS`), and cross-account access. |
| Cost | **$0.40/secret/month** + **$0.05/10,000 API calls**. For 1 secret retrieved ~1–10 times/month (container starts): **~$0.40/month**. |
| Security | Excellent — encrypted at rest with KMS (AWS-managed or CMK), fine-grained IAM policies, CloudTrail audit, automatic rotation support. |
| Rotation | Built-in rotation with Lambda functions. For a symmetric JWT secret, rotation creates a new version; the app can verify tokens against both `AWSCURRENT` and `AWSPREVIOUS` during the transition window. |
| SDK | `@aws-sdk/client-secrets-manager` — `GetSecretValue`. Lightweight, single API call. |
| Terraform | `aws_secretsmanager_secret` + `aws_secretsmanager_secret_version` resources. |
| ECS integration | ECS natively supports `valueFrom` pointing to a Secrets Manager ARN, automatically injecting the secret as an environment variable at container start — no application code changes needed if using env-var-based config. |
| Verdict | **Best fit**. Purpose-built for this exact use case. |

### 6.3 Comparison Summary

| Factor | Env Var (plaintext) | SSM Parameter Store | Secrets Manager |
|---|---|---|---|
| Monthly cost | $0 | $0 | ~$0.40 |
| Encryption at rest | No | Yes (KMS) | Yes (KMS) |
| Automatic rotation | No | No | **Yes (Lambda)** |
| Secret versioning | No | Version history | **AWSCURRENT/AWSPREVIOUS** |
| ECS native injection | Yes | Yes (`valueFrom`) | **Yes (`valueFrom`)** |
| Terraform support | N/A | `aws_ssm_parameter` | `aws_secretsmanager_secret` |
| Audit trail | No | CloudTrail | CloudTrail |

### 6.4 Recommendation: AWS Secrets Manager

**AWS Secrets Manager is recommended** for storing the JWT signing key (and other sensitive credentials like the WebAuthn RP configuration and database password):

1. **Security-first**: The JWT signing key is the highest-value secret in the system. Secrets Manager provides KMS encryption, IAM access control, and CloudTrail auditing out of the box.
2. **Cost is negligible**: At $0.40/month for 1 secret (or ~$1.20/month if JWT key, DB password, and WebAuthn config are stored as 3 secrets), this is trivial relative to the ~$40/month infrastructure bill.
3. **Rotation without redeployment**: When it's time to rotate the JWT signing key, Secrets Manager's versioning (`AWSCURRENT` / `AWSPREVIOUS`) allows the app to verify tokens signed with either version during the transition. No need for a coordinated restart.
4. **ECS native integration**: The ECS task definition can reference the Secrets Manager ARN directly via `valueFrom`, injecting the secret as an environment variable at container start. This means `ConfigService` reads it from `process.env.JWT_SECRET` as it does today — zero application code changes beyond adding the env var to the Zod schema.
5. **Already in the AWS ecosystem**: The project already uses DynamoDB, RDS, ECS, and ECR. Secrets Manager is a natural addition with no new provider or billing relationship.

### 6.5 Implementation Approach

1. **Terraform**: Add a `secrets-manager` module under `infra/aws/modules/` that creates `aws_secretsmanager_secret` and `aws_secretsmanager_secret_version` resources. The initial secret value is set via Terraform variable (marked `sensitive = true`) or populated manually after provisioning.
2. **ECS task definition**: Reference the secret ARN in the container definition's `secrets` block (not `environment`). ECS injects it as an env var at container start.
3. **IAM**: Grant the ECS Task Execution Role `secretsmanager:GetSecretValue` on the specific secret ARN.
4. **Application**: No code change — `ConfigService` reads `JWT_SECRET` from `process.env` as before. The secret is already decrypted and injected by ECS.
5. **Local development**: Continue using `.env` file with a local-only development key. The Secrets Manager integration is transparent — the app only sees an environment variable regardless of where it came from.

## 7. DynamoDB Adapter Test Strategy: Reusing Production Terraform via tflocal + LocalStack

### 7.1 Problem Statement

The project uses DynamoDB for ephemeral auth token storage (magic links, refresh tokens). Adapter tests for the DynamoDB repositories must exercise real DynamoDB behavior (TTL semantics, key structure, conditional writes) rather than mocks. Two approaches exist:

1. **Separate IaC for tests**: Write a second set of Terraform resources (or raw SDK `createTable` calls) just for the test environment. Duplicates the production schema definition and risks drift.
2. **Reuse the production Terraform module**: Apply the same `infra/aws/modules/dynamodb/` module against a local DynamoDB-compatible service during tests. Schema changes are automatically reflected in tests.

The goal is option 2 — no separate IaC, production module is the single source of truth.

### 7.2 Solution: `tflocal` + LocalStack via Testcontainers

#### How it works

LocalStack provides [`tflocal`](https://github.com/localstack/terraform-local) — a thin wrapper around the `terraform` CLI. It uses Terraform's [Override mechanism](https://developer.hashicorp.com/terraform/language/files/override) to generate a temporary `localstack_providers_override.tf` file that redirects every AWS provider endpoint to `http://localhost:4566`. Production `.tf` files are **not modified**.

```
tflocal apply  →  terraform apply (with auto-generated endpoint overrides)
```

The generated provider override looks like:

```hcl
provider "aws" {
  access_key                  = "mock_access_key"
  secret_key                  = "mock_secret_key"
  region                      = "us-east-1"
  skip_credentials_validation = true
  skip_metadata_api_check     = true
  skip_requesting_account_id  = true

  endpoints {
    dynamodb = "http://localhost:4566"
    # ... all services redirected
  }
}
```

#### Integration with Testcontainers

The test lifecycle:

1. **Start LocalStack via Testcontainers** — the `@testcontainers/localstack` module starts a `localstack/localstack` Docker container with a dynamically assigned host port.
2. **Resolve the endpoint URL** — `container.getConnectionUri()` returns `http://localhost:<mapped-port>`.
3. **Set `AWS_ENDPOINT_URL`** — export this env var before invoking `tflocal`. The `tflocal` wrapper reads it and injects it into all provider endpoint overrides.
4. **Run `tflocal init && tflocal apply -auto-approve`** — applies `infra/aws/modules/dynamodb/` against LocalStack, creating the exact same table(s) and TTL configuration as production.
5. **Run adapter tests** — the DynamoDB SDK client in the test suite is configured with the same `AWS_ENDPOINT_URL`, pointing at the now-provisioned LocalStack container.
6. **Tear down** — Testcontainers stops and removes the container after the test suite.

```typescript
// test/adapters/secondary/dynamodb/setup.ts (illustrative)
import { LocalStackContainer } from '@testcontainers/localstack';
import { execSync } from 'child_process';

let container: StartedLocalStackContainer;

beforeAll(async () => {
  container = await new LocalStackContainer('localstack/localstack:latest').start();
  const endpoint = container.getConnectionUri();  // e.g. http://localhost:32771

  execSync('tflocal init -input=false', {
    cwd: 'infra/aws/modules/dynamodb',
    env: { ...process.env, AWS_ENDPOINT_URL: endpoint },
  });
  execSync('tflocal apply -auto-approve -input=false', {
    cwd: 'infra/aws/modules/dynamodb',
    env: { ...process.env, AWS_ENDPOINT_URL: endpoint },
  });

  process.env.AWS_ENDPOINT_URL = endpoint;
});

afterAll(async () => {
  await container.stop();
  delete process.env.AWS_ENDPOINT_URL;
});
```

### 7.3 Caveats and Constraints

| Concern | Detail |
|---|---|
| **LocalStack service parity** | DynamoDB is fully supported on LocalStack's free tier. TTL, conditional writes, GSIs, and `TransactWriteItems` are all covered. No Pro subscription needed for this use case. |
| **`tflocal` installation** | `tflocal` is a Python CLI tool (`pip install terraform-local`). Must be available in the CI environment. Add to CI setup steps alongside `terraform`. |
| **Local Terraform modules** | If `infra/aws/modules/dynamodb/` references other local modules, set `ADDITIONAL_TF_OVERRIDE_LOCATIONS` to those module paths so `tflocal` drops override files there too. |
| **Terraform state** | Each test run uses a fresh LocalStack container and a fresh `tflocal apply`. No state backend is needed — use a local state file (default) isolated to the test run. |
| **Parallelism** | If multiple test workers run concurrently, each must have its own LocalStack container and mapped port to avoid table name collisions. |
| **Table name parameterization** | The production DynamoDB module should accept table names as Terraform input variables (already the pattern). Tests pass the same variable values, ensuring schema parity. |

### 7.4 Decision

**Use `tflocal` + LocalStack via Testcontainers for DynamoDB adapter tests.** 

- The production `infra/aws/modules/dynamodb/` module is the single source of truth for the DynamoDB table schema and TTL configuration.
- DynamoDB adapter tests apply this module against LocalStack at test startup, exercising the same infrastructure definition that ships to production.
- No separate `createTable` SDK calls or test-only IaC. Schema changes in the Terraform module are automatically reflected in adapter tests on the next run.
- `tflocal` is installed as a CI dependency alongside `terraform`. Locally, developers must have both `terraform` and `tflocal` installed (via `pip install terraform-local`).
