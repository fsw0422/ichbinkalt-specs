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

The earlier provider comparison assumed a managed PostgreSQL dependency. That is no longer the current architecture. The current scope uses:

- container hosting with WebSocket support for Gemini Live sessions,
- a single DynamoDB table for user/auth state,
- Secrets Manager for JWT signing material,
- a container registry,
- Terraform/Terragrunt-managed AWS infrastructure.

### 4.1 Current Requirements Specific to This Project

| Requirement | Why |
|---|---|
| **WebSocket support** | Real-time bidirectional audio streaming for Gemini Live sessions. |
| **Single-table key-value/document storage with TTL** | `UserCore`, user-owned singleton items, passkeys, magic links, and refresh tokens all live in DynamoDB. Tokens need automatic expiry. |
| **Container hosting** | The NestJS backend runs in Docker and is deployed as a single service. |
| **Secrets management** | JWT signing material and other sensitive configuration must be injected securely at runtime. |
| **Terraform support** | IaC is already a hard requirement for repeatable provisioning. |
| **Gemini API access** | Gemini is accessed over HTTPS/WebSockets via API key, so no cloud-specific AI runtime is required. |

### 4.2 Provider Comparison Against the Current Scope

| Provider | Fit for Current Scope | Main Drawbacks | Verdict |
|---|---|---|---|
| **AWS** | Strongest fit. ECS Fargate, DynamoDB, Secrets Manager, and ECR align directly with the current implementation and infrastructure modules. | ALB introduces a fixed monthly cost floor in the current infra shape. | **Selected** |
| **GCP** | Cloud Run and Secret Manager are viable, and Firestore could cover some TTL-driven token flows. | Firestore diverges materially from the current DynamoDB single-table design and its access patterns. | Not selected |
| **Azure** | Container Apps, Key Vault, and Cosmos DB can cover the broad categories. | Cosmos DB pricing and data-model ergonomics are a poorer fit for the current DynamoDB-centric design. | Not selected |
| **Fly.io** | Cheapest hosting path and good WebSocket support. | No managed DynamoDB-equivalent, weaker Terraform story, and more operational tradeoffs around surrounding infrastructure. | Not selected |

### 4.3 Recommendation: AWS

AWS remains the best fit for the current architecture for these reasons:

1. **Implementation alignment**: The backend and infra already target ECS, DynamoDB, Secrets Manager, and ECR.
2. **Storage fit**: DynamoDB supports the current single-table user/auth model, including TTL for tokens and transactional uniqueness enforcement for normalized email addresses.
3. **Operational maturity**: The AWS Terraform provider and surrounding tooling are the most mature option for the current stack.
4. **Future flexibility**: The same account and provider can later absorb email delivery, CDN, monitoring, and additional persistence without re-platforming.

The main current cost tradeoff is the ALB fixed floor. If needed later, the ingress layer can be revisited independently of the rest of the stack.

## 5. Data Storage Strategy: Single-Table DynamoDB for User and Auth Data

The current storage decision is simpler than the earlier hybrid exploration: user and auth data live together in a single DynamoDB table, while conversation state remains in memory for the active session.

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

### 5.2 Current DynamoDB Single-Table Decision

The current implementation and spec direction use DynamoDB not just for short-lived tokens, but for the entire user/auth slice:

- `UserCore`
- `UserLearningProfile`
- `UserUsageStats`
- `UserPreferences`
- `UserPasskey`
- `MagicLinkToken`
- `RefreshToken`

Conversation state remains in memory for the active session and is not yet part of the persistent storage decision.

#### Current User/Auth Item Layout

| Item | Partition Key | Sort Key | Notes |
|---|---|---|---|
| `UserCore` | `A#{userId}` | `A` | Minimal identity row: `userId`, `email`, `displayName`, `createdAt`, `updatedAt`, `lastActiveAt`, `emailConfirmed` |
| `UserPasskey` | `A#{userId}` | `B#{credentialId}` | One-to-many child items for passkeys |
| `RefreshToken` | `A#{userId}` | `C#{tokenId}` | Child items with TTL |
| `UserLearningProfile` | `A#{userId}` | `D` | Singleton child item |
| `UserUsageStats` | `A#{userId}` | `E` | Singleton child item |
| `UserPreferences` | `A#{userId}` | `F` | Singleton child item |
| `MagicLinkToken` | `B#{tokenId}` | `A` | Standalone token item with TTL |
| Email lookup | `C#{normalizedEmail}` | `A` | Enforces exactly one durable user core per normalized email |

#### Current Index and Access Decisions

| Concern | Current Decision |
|---|---|
| Passkey ceremony user identifier | Use the stable ULID `userId` directly as the WebAuthn `userID` |
| User-handle lookup GSI | **Removed**. There is no `user-handle-index` in the current model |
| Token verification lookup | `tokenHash-index` on the compact token-hash attribute |
| Recent email-link rate limit lookup | `email-createdAt-index` on normalized email plus created-at |
| Token expiry cleanup | DynamoDB TTL on the configured TTL attribute |

#### Why the Current Model Works

1. **The minimal core row stays stable**: core identity fields remain compact and easy to reason about.
2. **User-owned singleton items avoid nested-document bloat**: profile, usage, and preferences evolve independently without turning the core row into a catch-all blob.
3. **Email uniqueness is enforced transactionally**: the normalized-email lookup item prevents duplicate account creation.
4. **The access patterns are key-driven**: lookups are by `userId`, normalized email, credential ID within a user partition, or token hash.
5. **TTL remains native and automatic**: magic-link tokens and refresh tokens expire without custom cleanup jobs.

### 5.3 Overall Architecture Decision

| Data Category | Storage | Rationale |
|---|---|---|
| `UserCore` | **DynamoDB** | Minimal durable account identity |
| `UserLearningProfile` | **DynamoDB** | Dedicated user-owned singleton item |
| `UserUsageStats` | **DynamoDB** | Dedicated user-owned singleton item |
| `UserPreferences` | **DynamoDB** | Dedicated user-owned singleton item |
| `UserPasskey` | **DynamoDB** | Passkey lookup and update patterns are partition-local |
| `MagicLinkToken` | **DynamoDB** | Ephemeral token data with native TTL |
| `RefreshToken` | **DynamoDB** | Ephemeral token data with native TTL |
| `Conversation`, `Message` | **In-memory (current scope)** | Active-session state only; persistent conversation storage is still out of scope |

This is the current architecture reflected by the spec set in this session. The earlier hybrid `RDS + DynamoDB` exploration is no longer the active direction.

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

**AWS Secrets Manager is recommended** for storing the JWT signing key (and other sensitive credentials like the WebAuthn RP configuration and email-delivery provider credentials):

1. **Security-first**: The JWT signing key is the highest-value secret in the system. Secrets Manager provides KMS encryption, IAM access control, and CloudTrail auditing out of the box.
2. **Cost is negligible**: At $0.40/month for 1 secret (or ~$1.20/month if JWT key, DB password, and WebAuthn config are stored as 3 secrets), this is trivial relative to the ~$40/month infrastructure bill.
3. **Rotation without redeployment**: When it's time to rotate the JWT signing key, Secrets Manager's versioning (`AWSCURRENT` / `AWSPREVIOUS`) allows the app to verify tokens signed with either version during the transition. No need for a coordinated restart.
4. **ECS native integration**: The ECS task definition can reference the Secrets Manager ARN directly via `valueFrom`, injecting the secret as an environment variable at container start. This means `ConfigService` reads it from `process.env.JWT_SECRET` as it does today — zero application code changes beyond adding the env var to the Zod schema.
5. **Already in the AWS ecosystem**: The project already uses DynamoDB, ECS, ECR, and AWS-managed infrastructure modules. Secrets Manager is a natural addition with no new provider or billing relationship.

### 6.5 Implementation Approach

1. **Terraform**: Add a `secrets-manager` module under `infra/aws/modules/` that creates `aws_secretsmanager_secret` and `aws_secretsmanager_secret_version` resources. The initial secret value is set via Terraform variable (marked `sensitive = true`) or populated manually after provisioning.
2. **ECS task definition**: Reference the secret ARN in the container definition's `secrets` block (not `environment`). ECS injects it as an env var at container start.
3. **IAM**: Grant the ECS Task Execution Role `secretsmanager:GetSecretValue` on the specific secret ARN.
4. **Application**: No code change — `ConfigService` reads `JWT_SECRET` from `process.env` as before. The secret is already decrypted and injected by ECS.
5. **Local development**: Continue using `.env` file with a local-only development key. The Secrets Manager integration is transparent — the app only sees an environment variable regardless of where it came from.

## 7. DynamoDB Adapter Test Strategy: Reusing Production Terraform via Host Terraform + LocalStack

### 7.1 Problem Statement

The project uses DynamoDB for the user/auth single-table model. Adapter tests for the DynamoDB repositories must exercise real DynamoDB behavior (key structure, GSIs, conditional writes, transactions, TTL configuration) rather than mocks. Two approaches exist:

1. **Separate IaC for tests**: Write a second set of Terraform resources (or raw SDK `createTable` calls) just for the test environment. Duplicates the production schema definition and risks drift.
2. **Reuse the production Terraform module**: Apply the same `infra/aws/modules/dynamodb/` module against a local DynamoDB-compatible service during tests. Schema changes are automatically reflected in tests.

The goal is option 2 — no separate IaC, production module is the single source of truth.

### 7.2 Solution: Stage the Terraform Module and Apply It with Host `terraform`

#### How it works

The current helper does not use `tflocal`. Instead it stages a temporary Terraform workspace under `.terraform-test-tmp/`, copies in:

- the production DynamoDB module from `infra/aws/modules/dynamodb`, and
- the LocalStack-specific test root from `infra/aws/localstack/test/dynamodb`.

It then writes a generated `terraform.auto.tfvars.json` file containing the LocalStack endpoint URL, region, and environment, and runs the host `terraform` binary directly.

#### Integration with Testcontainers

The test lifecycle:

1. **Start LocalStack via Testcontainers** — the `@testcontainers/localstack` module starts a `localstack/localstack` Docker container with a dynamically assigned host port.
2. **Resolve the endpoint URL** — `container.getConnectionUri()` returns `http://localhost:<mapped-port>`.
3. **Stage a temporary Terraform workspace** — copy the test harness and the production module into a temp directory.
4. **Write `terraform.auto.tfvars.json`** — include the LocalStack endpoint URL and standard test values.
5. **Run `terraform init && terraform apply -auto-approve`** — apply the staged module against LocalStack, creating the same table shape, GSIs, and TTL configuration expected by the repositories.
5. **Run adapter tests** — the DynamoDB SDK client in the test suite is configured with the same `AWS_ENDPOINT_URL`, pointing at the now-provisioned LocalStack container.
6. **Tear down** — Testcontainers stops and removes the container after the test suite.

### 7.3 Caveats and Constraints

| Concern | Detail |
|---|---|
| **LocalStack service parity** | DynamoDB is fully supported on LocalStack's free tier. TTL, conditional writes, GSIs, and `TransactWriteItems` are all covered. No Pro subscription needed for this use case. |
| **Host `terraform` binary** | The test helper shells out to `terraform` directly. If `terraform` is missing or not executable, adapter tests fail with `spawn terraform ENOENT` or `spawn terraform EACCES`. |
| **Terraform state** | Each test run uses a fresh LocalStack container and a fresh staged workspace. No remote state backend is needed. |
| **Parallelism** | If multiple test workers run concurrently, each must have its own LocalStack container and mapped port to avoid table name collisions. |
| **Table name parameterization** | The production DynamoDB module accepts test values via the generated `terraform.auto.tfvars.json`, ensuring schema parity. |

### 7.4 Decision

**Use LocalStack via Testcontainers and apply the staged production Terraform module with host `terraform` for DynamoDB adapter tests.**

- The production `infra/aws/modules/dynamodb/` module is the single source of truth for the DynamoDB table schema and TTL configuration.
- DynamoDB adapter tests apply this module against LocalStack at test startup, exercising the same infrastructure definition that ships to production.
- No separate `createTable` SDK calls or test-only IaC. Schema changes in the Terraform module are automatically reflected in adapter tests on the next run.
- CI and local development both require an executable host `terraform` binary on `PATH`.
