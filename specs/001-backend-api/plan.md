# Implementation Plan: AI Conversation API

**Branch**: `001-backend-api` | **Date**: 2025-11-09 | **Updated**: 2026-04-01 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/home/ksp/projects/ichbinkalt-specs-001-backend-api/specs/001-backend-api/spec.md`
**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.
- Better context understanding (audio-to-audio processing)

---

## Summary

This plan outlines the implementation of a backend service for a German learning application. The core feature is an API that allows a client application to have a real-time, interactive voice conversation with an AI. The technical approach involves using a NestJS (Node.js/TypeScript) server with WebSockets for real-time communication, and integrating Google's Gemini Native Audio Model for unified speech-to-text, text-to-speech, and conversational AI capabilities.

## Technical Context

**Language/Version**: TypeScript (using Node.js >= 18)
**Primary Dependencies**:
- Web Framework: NestJS
- **Unified Audio Service**: Google Gemini Native Audio (`@google/genai`)
  - Speech-to-Text (STT): Built-in (model: `gemini-2.5-flash-native-audio-preview-09-2025`)
  - Text-to-Speech (TTS): Built-in (model: `gemini-2.5-flash-preview-tts`)
  - Conversational AI (LLM): Built-in (same unified model)
- **Authentication**: `@simplewebauthn/server` (WebAuthn passkeys), `jsonwebtoken` (JWT)
**Storage** (hybrid — see [research.md §5](./research.md)):
- **PostgreSQL (RDS)** via TypeORM: All relational data — users, profiles, passkeys, conversations, messages.
- **DynamoDB** via `@aws-sdk/lib-dynamodb`: Ephemeral auth tokens — magic link tokens (15 min TTL) and refresh tokens (7 day TTL). Native TTL auto-cleanup, scale-to-zero, effectively free at low volumes.
- **AWS Secrets Manager**: Stores the JWT signing key (and other sensitive credentials). ECS injects secrets as environment variables at container start via `valueFrom`. ~$0.40/secret/month. See [research.md §6](./research.md).
**Testing**: Jest with Testcontainers for secondary adapter tests. PostgreSQL adapter tests use the official `postgres` container image. DynamoDB adapter tests use LocalStack via Testcontainers and apply the production `infra/aws/modules/dynamodb/` Terraform module via `tflocal` at test startup — no separate IaC for tests. See [research.md §7](./research.md) for details.
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
| VI. Environment Configuration | PASS | The project will be structured to handle `local`, `test`, and `prod` environments. |
| VII. Continuous Integration | PASS | CI checks for linting, testing, and builds will be set up. |

All constitutional gates pass.

## Project Structure

### Infrastructure

```text
infra/
└── aws/
    ├── modules/                        # Plain Terraform modules (no Terragrunt awareness)
    │   ├── rds/                        # RDS PostgreSQL instance, subnet group, security group
    │   ├── dynamodb/                   # DynamoDB tables with TTL config; outputs table ARN
    │   ├── ecs/                        # ECS cluster, task definition, service + IAM roles
    │   ├── ecr/                        # ECR repository for container images
    │   ├── secrets-manager/            # Secrets Manager secrets (JWT signing key, DB password)
    │   ├── vpc/                        # VPC, subnets, route tables, IGW
    │   └── alb/                        # Application Load Balancer, listener, target group
    ├── terragrunt.hcl                  # Root config: DRY remote_state backend pattern
    └── accounts/
        ├── dev/
        │   ├── account.hcl             # dev AWS account ID + common locals
        │   └── us-east-1/
        │       ├── region.hcl          # Region locals (aws_region = "us-east-1")
        │       ├── vpc/
        │       │   └── terragrunt.hcl  # inputs: CIDR, AZs
        │       ├── ecr/
        │       │   └── terragrunt.hcl
        │       ├── rds/
        │       │   └── terragrunt.hcl  # dependency: vpc | inputs: db.t4g.micro, single-AZ
        │       ├── dynamodb/
        │       │   └── terragrunt.hcl  # inputs: table name, TTL attribute
        │       ├── secrets-manager/
        │       │   └── terragrunt.hcl  # inputs: secret names (JWT key, DB password)
        │       └── ecs/
        │           └── terragrunt.hcl  # dependency: vpc, ecr, rds, dynamodb, secrets-manager
        ├── staging/
        │   ├── account.hcl
        │   └── us-east-1/
        │       ├── region.hcl
        │       ├── vpc/
        │       │   └── terragrunt.hcl
        │       ├── ecr/
        │       │   └── terragrunt.hcl
        │       ├── rds/
        │       │   └── terragrunt.hcl  # inputs: db.t4g.micro, single-AZ
        │       ├── dynamodb/
        │       │   └── terragrunt.hcl
        │       ├── secrets-manager/
        │       │   └── terragrunt.hcl
        │       └── ecs/
        │           └── terragrunt.hcl
            ├── account.hcl
            └── us-east-1/
                ├── region.hcl
                ├── vpc/
                │   └── terragrunt.hcl
                ├── ecr/
                │   └── terragrunt.hcl
                ├── rds/
                │   └── terragrunt.hcl  # inputs: db.t4g.small, Multi-AZ enabled
                ├── dynamodb/
                │   └── terragrunt.hcl
                ├── secrets-manager/
                │   └── terragrunt.hcl
                └── ecs/
                    └── terragrunt.hcl
```

**Infrastructure Decision**: Each environment maps to a dedicated AWS account (AWS Organizations best practice) for hard blast-radius isolation. Terragrunt is used to eliminate boilerplate across accounts and regions:

- **Root `terragrunt.hcl`**: Defines the remote state backend pattern once — `s3://ichbinkalt-tfstate-${account_name}/${region}/${path}/terraform.tfstate` with a DynamoDB lock table. Each module root inherits this automatically via `find_in_parent_folders()`, so no backend block is ever copy-pasted.
- **`account.hcl`**: Plain HCL file (not a Terragrunt config) declaring the AWS account ID and account name as locals. Read by child `terragrunt.hcl` files via `read_terragrunt_config(find_in_parent_folders("account.hcl"))`.
- **`region.hcl`**: Declares the region local. Combined with `account.hcl`, provides all context needed to construct unique resource names and state paths without repetition.
- **Per-module `terragrunt.hcl`**: Each leaf file declares `terraform { source = "../../../../modules/rds" }`, a `dependency` block for cross-module outputs (e.g., `ecs` reads the DynamoDB table ARN from the `dynamodb` dependency), and `inputs` with environment-specific values. No `main.tf`, `variables.tf`, or `terraform.tfvars` files needed in the accounts tree.
- **`terragrunt run-all apply`**: Running this from `accounts/prod/us-east-1/` applies all modules in dependency order (vpc → ecr/rds/dynamodb → ecs) in a single command.
- **Adding a region**: Create `accounts/prod/us-west-2/` with the same module subdirectories and updated `region.hcl` — no module code changes.

**IAM**: No separate IAM module. The `ecs/` module defines the ECS Task Execution Role and Task Role internally, accepting a `task_role_extra_policy_arns` variable. The `ecs/terragrunt.hcl` reads the DynamoDB table ARN from its `dependency.dynamodb.outputs.table_arn` and passes a constructed policy ARN into that variable. The ECS Task Execution Role also receives `secretsmanager:GetSecretValue` permission on the Secrets Manager secret ARN (from `dependency.secrets-manager.outputs.secret_arn`) so that ECS can inject secrets at container start. Policies stay close to the resources they govern.

### CI/CD

```text
ci.sh                      # CI entrypoint
cd.sh                      # CD build, migration, and deployment entrypoint
Dockerfile                 # Runtime container image
Dockerfile.builder         # Builder image
Dockerfile.dbmigrator      # Database migration image
docker-compose.yml         # Local orchestration
```

### Database

```text
db/
├── changelog/
└── run-postgres-migrations.sh
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
│       ├── dynamodb/      # DynamoDB client for auth tokens (magic links, refresh tokens)
│       ├── email/         # Email delivery adapter (IEmailService port)
│       ├── gemini-live/   # Gemini Live integration
│       └── typeorm/       # TypeORM repositories and persistence entities
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
│       ├── dynamodb/      # Adapter tests run against LocalStack via Testcontainers
│       └── typeorm/       # Adapter tests run against the official postgres image via Testcontainers
└── config/
```

### Development Environment

```text
.vscode/
```

**Structure Decision**: The structure follows the backend repository as implemented and continues to follow hexagonal architecture principles. The `core` layer contains entities, DTOs, ports, services, and validation rules. The `adapters` layer is split into `primary` adapters for HTTP and WebSocket entrypoints and `secondary` adapters for Gemini Live and TypeORM persistence. Root-level modules (`AppModule.ts`, `ConversationModule.ts`, `ConversationEventDispatcherModule.ts`, and `Main.ts`) compose those pieces into the NestJS application.

**Note on Unified Service**: The `gemini-live` adapter implements the live-session integration behind the secondary ports, while the `typeorm` adapter owns relational database persistence and the `dynamodb` adapter owns ephemeral token storage. This preserves hexagonal boundaries while matching the backend's concrete module layout.

**Note on Hybrid Storage**: Auth token repositories (`IMagicLinkTokenRepository`, `IRefreshTokenRepository`) are defined as secondary ports in `core/ports/secondary/` and implemented by the `dynamodb` adapter. All relational data repositories (`IUserRepository`, `IConversationRepository`, etc.) are implemented by the `typeorm` adapter. This keeps the storage decision invisible to the service layer.

**Note on Adapter Testing**: Secondary adapter tests should exercise real backing services through Testcontainers rather than mocks for persistence behavior. The `typeorm` adapter tests use the native official `postgres` image, while the `dynamodb` adapter tests use LocalStack to provide a DynamoDB-compatible environment.

This separation keeps the core logic independent of external concerns while aligning the documented structure with the backend repository used for implementation. The `test` directory mirrors the exercised adapter and config areas that exist in the backend today.
