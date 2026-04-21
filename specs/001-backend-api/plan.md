# Implementation Plan: AI Conversation API

**Branch**: `001-backend-api` | **Date**: 2025-11-09 | **Updated**: 2026-04-01 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/home/ksp/projects/ichbinkalt-specs-001-backend-api/specs/001-backend-api/spec.md`
**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.
- Better context understanding (audio-to-audio processing)

---

## Summary

Authentication is now passkey-first at sign-up time. A user enters an email, the backend creates or reuses the single user bound to that normalized email, and the client is immediately prompted to register a passkey. Email verification becomes a separate later flow used to confirm inbox ownership and unlock subscription-based services and upgrades. Recovery remains email-based: a verified recovery link auto-confirms the email, revokes stale credentials when reclaim is needed, and returns a scoped bootstrap token that drives direct passkey re-registration.

This plan outlines the implementation of a backend service for a German learning application. The core feature is an API that allows a client application to have a real-time, interactive voice conversation with an AI. The technical approach involves using a NestJS (Node.js/TypeScript) server with WebSockets for real-time communication, and integrating Google's Gemini Native Audio Model for unified speech-to-text, text-to-speech, and conversational AI capabilities. Authentication now begins with a unique-email user record created or resumed on first passkey-registration request, followed immediately by WebAuthn registration. Later inbox verification is handled by a dedicated email-verification flow, while passkey loss or account reclaim is handled by an emailed passkey-recovery link that drops the user directly into replacement passkey registration. Returning login uses passkey authentication. Authorization is modeled separately via `accountStatus`, derived `onboardingComplete`, and `subscriptionStatus` rather than a single `accountState` field.

## Technical Context

**Language/Version**: TypeScript (using Node.js >= 18)
**Primary Dependencies**:
- Web Framework: NestJS
- **Unified Audio Service**: Google Gemini Native Audio (`@google/genai`)
  - Speech-to-Text (STT): Built-in (model: `gemini-2.5-flash-native-audio-preview-09-2025`)
  - Text-to-Speech (TTS): Built-in (model: `gemini-2.5-flash-preview-tts`)
  - Conversational AI (LLM): Built-in (same unified model)
- **Authentication**: `@simplewebauthn/server` (WebAuthn passkeys), `jsonwebtoken` (JWT)
**Storage** (see [research.md §5](./research.md)):
- **DynamoDB** via `@aws-sdk/lib-dynamodb`: All auth-related data lives in a single table (`ichbinkalt`) using compact `pk`/`sk` string values. The minimal `UserCore` item is stored as `pk = A#{userId}`, `sk = A`, where `userId` is a ULID and also serves as the WebAuthn user ID. `UserLearningProfile`, `UserUsageStats`, and `UserPreferences` are separate singleton child items in the same user partition under `sk = D`, `sk = E`, and `sk = F`. Passkeys remain child items under `sk = B#{credentialId}`, refresh tokens remain child items under `sk = C#{tokenId}`, emailed verification and recovery tokens remain standalone items under `pk = B#{tokenId}`, `sk = A`, and unique email addresses are enforced through a transactional `pk = C#{normalizedEmail}`, `sk = A` lookup item. GSIs still support token-hash lookups and recent email-link lookups. See [data-model.md](./data-model.md) for the full mapping.
- **AWS Secrets Manager**: Stores the JWT signing key (and other sensitive credentials). ECS injects secrets as environment variables at container start via `valueFrom`. ~$0.40/secret/month. See [research.md §6](./research.md).
**Testing**: Jest with Testcontainers for secondary adapter tests. DynamoDB adapter tests use LocalStack via Testcontainers and apply the staged production Terraform module through the host `terraform` binary at test startup — no separate test-only schema definition. See [research.md §7](./research.md) for details.
**Target Platform**: Linux server (containerized).
**Project Type**: Backend service for a web/mobile application.
**Performance Goals**: As per spec: <500ms transcription latency, <3s AI response latency, support for 100 concurrent users.
**Constraints**: Real-time processing, high-accuracy German transcription.
**Scale/Scope**: Initial version for 100 concurrent users, with volatile storage.
**Containerization**: Docker
**Infrastructure as Code**: Terraform

**Audio Specifications**:
- Input: PCM audio at 16kHz, 16-bit, mono (for STT)
- Output: PCM audio at 24kHz, 16-bit, mono (for TTS)
- Voice: "Kore" (German-optimized natural voice)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|---|---|---|
| I. Basic Security First | PASS | Plan includes HTTPS and token-based auth for non-public routes. |
| II. Health and Resilience | PASS | A `/health` endpoint and graceful shutdown will be included. |
| III. Measurable Observability | PASS | Structured logging with correlation IDs will be implemented. |
| IV. Lean Maintainability | PASS | Dependencies will be kept minimal, and public endpoints will be documented via OpenAPI. |
| V. Hexagonal Architecture | PASS | The proposed structure with a `core` layer and `adapters` aligns with hexagonal principles. |
| VI. Environment Configuration | PASS | The project will be structured to handle `local`, `test`, and `prod` environments. Only `prod` is active infrastructure for now. |
| VII. Continuous Integration | PASS | CI checks for linting, testing, and builds will be set up. |

All constitutional gates pass.

## Project Structure

### Infrastructure

```text
infra/
└── aws/
    ├── modules/                        # Plain Terraform modules (no Terragrunt awareness)
    │   ├── dynamodb/                   # Single DynamoDB table (ichbinkalt) with auth item patterns, lookup items, GSIs, and TTL config; outputs table ARN
    │   ├── ecs/                        # ECS cluster, task definition, service + IAM roles
    │   ├── ecr/                        # ECR repository for container images
    │   ├── secrets-manager/            # Secrets Manager secrets (JWT signing key)
    │   ├── vpc/                        # VPC, subnets, route tables, IGW
    │   └── alb/                        # Application Load Balancer, listener, target group
    ├── terragrunt.hcl                  # Root config: DRY remote_state backend pattern
    └── accounts/
        ├── dev/                            # Directory kept; not deployed — prod only for now
        │   ├── account.hcl
        │   └── eu-west-1/
        │       ├── region.hcl
        │       ├── vpc/
        │       │   └── terragrunt.hcl  # not in use
        │       ├── ecr/
        │       │   └── terragrunt.hcl  # not in use
        │       ├── dynamodb/
        │       │   └── terragrunt.hcl  # not in use
        │       ├── secrets-manager/
        │       │   └── terragrunt.hcl  # not in use
        │       └── ecs/
        │           └── terragrunt.hcl  # not in use
        ├── staging/                        # Directory kept; not deployed — prod only for now
        │   ├── account.hcl
        │   └── eu-west-1/
        │       ├── region.hcl
        │       ├── vpc/
        │       │   └── terragrunt.hcl  # not in use
        │       ├── ecr/
        │       │   └── terragrunt.hcl  # not in use
        │       ├── dynamodb/
        │       │   └── terragrunt.hcl  # not in use
        │       ├── secrets-manager/
        │       │   └── terragrunt.hcl  # not in use
        │       └── ecs/
        │           └── terragrunt.hcl  # not in use
        └── prod/
            ├── account.hcl
            └── eu-west-1/
                ├── region.hcl
                ├── vpc/
                │   └── terragrunt.hcl
                ├── ecr/
                │   └── terragrunt.hcl
                ├── dynamodb/
                │   └── terragrunt.hcl  # inputs: table name, GSIs on `N`/`B`+`D`, TTL attr `R`
                ├── secrets-manager/
                │   └── terragrunt.hcl
                └── ecs/
                    └── terragrunt.hcl
```

**Infrastructure Decision**: Each environment maps to a dedicated AWS account (AWS Organizations best practice) for hard blast-radius isolation. Terragrunt is used to eliminate boilerplate across accounts and regions:

- **Root `terragrunt.hcl`**: Defines the remote state backend pattern once — `s3://ichbinkalt-tfstate-${account_name}/${region}/${path}/terraform.tfstate` with a DynamoDB lock table. Each module root inherits this automatically via `find_in_parent_folders()`, so no backend block is ever copy-pasted.
- **`account.hcl`**: Plain HCL file (not a Terragrunt config) declaring the AWS account ID and account name as locals. Read by child `terragrunt.hcl` files via `read_terragrunt_config(find_in_parent_folders("account.hcl"))`.
- **`region.hcl`**: Declares the region local. Combined with `account.hcl`, provides all context needed to construct unique resource names and state paths without repetition.
- **Per-module `terragrunt.hcl`**: Each leaf file declares `terraform { source = "../../../../modules/<module-name>" }`, a `dependency` block for cross-module outputs (e.g., `ecs` reads the DynamoDB table ARN from the `dynamodb` dependency), and `inputs` with environment-specific values. No `main.tf`, `variables.tf`, or `terraform.tfvars` files needed in the accounts tree.
- **`terragrunt run-all apply`**: Running this from `accounts/prod/eu-west-1/` applies all modules in dependency order (vpc → ecr/dynamodb → ecs) in a single command.
- **Adding a region**: Create `accounts/prod/us-east-1/` with the same module subdirectories and updated `region.hcl` — no module code changes.

**IAM**: No separate IAM module. The `ecs/` module defines the ECS Task Execution Role and Task Role internally, accepting a `task_role_extra_policy_arns` variable. The `ecs/terragrunt.hcl` reads the DynamoDB table ARN from its `dependency.dynamodb.outputs.table_arn` and passes a constructed policy ARN into that variable. The ECS Task Execution Role also receives `secretsmanager:GetSecretValue` permission on the Secrets Manager secret ARN (from `dependency.secrets-manager.outputs.secret_arn`) so that ECS can inject secrets at container start. Policies stay close to the resources they govern.

### CI/CD

```text
ci.sh                      # CI entrypoint
cd.sh                      # CD build and deployment entrypoint
Dockerfile                 # Runtime container image
Dockerfile.builder         # Builder image
docker-compose.yml         # Local orchestration
```

### Source Code
```text
src/
├── AppModule.ts
├── Main.ts                # Application bootstrap
├── adapters/
│   ├── primary/
│   │   ├── rest/          # REST controllers
│   │   └── websocket/     # WebSocket gateways and sockets
│   └── secondary/
│       ├── dynamodb/      # DynamoDB client for user core, user-owned singleton items, passkeys, magic links, refresh tokens, and lookup items
│       ├── email/         # Email delivery adapter (IEmailService port)
│       └── gemini-live/   # Gemini Live integration
├── config/                # Configuration, logging, validation, exception handling
├── core/
│   ├── dtos/              # Data transfer objects
│   ├── entities/          # Domain entities
│   ├── ports/
│   │   ├── primary/       # Use-case interfaces exposed to adapters
│   │   └── secondary/     # Repository and infrastructure ports
│   ├── services/          # Business logic
│   └── validation/        # Shared validation schemas
test/
├── adapters/
│   ├── primary/
│   │   ├── rest/
│   │   └── websocket/
│   └── secondary/
│       └── dynamodb/      # Adapter tests run against LocalStack via Testcontainers
└── config/
```

### Development Environment

```text
.vscode/
```

**Structure Decision**: The structure follows the backend repository as implemented and continues to follow hexagonal architecture principles. The `core` layer contains entities, DTOs, ports, services, and validation rules. The `adapters` layer is split into `primary` adapters for HTTP and WebSocket entrypoints and `secondary` adapters for Gemini Live, DynamoDB, and in-memory persistence. Root-level modules (`AppModule.ts`, `ConversationModule.ts`, `ConversationEventDispatcherModule.ts`, and `Main.ts`) compose those pieces into the NestJS application.

**Note on Unified Service**: The `gemini-live` adapter implements the live-session integration behind the secondary ports, while the `dynamodb` adapter owns all auth-related data (`UserCore`, user-owned singleton profile items, passkeys, emailed auth tokens, refresh tokens) in a single-table design. This preserves hexagonal boundaries while matching the backend's concrete module layout.

**Note on Storage**: The auth model is centered on an `A#{userId}` partition. The singleton `A` row is a minimal `UserCore` item holding only `userId`, `email`, `displayName`, `createdAt`, `updatedAt`, `lastActiveAt`, and `emailConfirmed`. `UserLearningProfile`, `UserUsageStats`, and `UserPreferences` are separate singleton child items in that same partition under `sk = D`, `sk = E`, and `sk = F`, so they are not embedded as nested documents on the core row. Onboarding completion is derived from whether passkey child items currently exist under that partition. Future lifecycle or billing state should likewise avoid bloating the user core row unless a concrete access pattern demands it. Passkeys and refresh tokens remain child items under `B#{credentialId}` and `C#{tokenId}` sort keys. Emailed verification and recovery tokens remain standalone items under `B#{tokenId}` with `sk = A`. Email uniqueness is enforced through a transactional `C#{normalizedEmail}` lookup item with `sk = A`. Conversation state (`IConversationRepository`) is held in memory for the duration of the active session and discarded on session end. This keeps the storage decision invisible to the service layer.

**Note on Verified Ownership**: `emailConfirmed = true` is no longer a prerequisite for first sign-in. It is a later inbox-ownership proof used to unlock subscription-based services and upgrades. Recovery verification also sets `emailConfirmed = true` and may reclaim the account by revoking stale credentials before passkey re-registration.

**Note on Adapter Testing**: Secondary adapter tests should exercise real backing services through Testcontainers rather than mocks for persistence behavior. The `dynamodb` adapter tests use LocalStack to provide a DynamoDB-compatible environment.

This separation keeps the core logic independent of external concerns while aligning the documented structure with the backend repository used for implementation. The `test` directory mirrors the exercised adapter and config areas that exist in the backend today.
