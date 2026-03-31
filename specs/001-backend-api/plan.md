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
**Storage**: PostgreSQL via TypeORM for conversations, lessons, and messages.
**Testing**: Jest
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

### Documentation (this feature)

```text
specs/001-backend-api/
├── plan.md                # This file (/speckit.plan command output)
├── research.md            # Phase 0 output (/speckit.plan command)
├── data-model.md          # Phase 1 output (/speckit.plan command)
├── contracts/             # Phase 1 output (/speckit.plan command)
└── tasks.md               # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

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
│   ├── 001-create-conversations-table.sql
│   ├── 002-create-messages-table.sql
│   ├── 003-create-lessons-table.sql
│   └── db.changelog-master.xml
└── run-postgres-migrations.sh
```

### Source Code
```text
src/
├── AppModule.ts
├── ConversationModule.ts
├── ConversationEventDispatcherModule.ts
├── Main.ts                # Application bootstrap
├── adapters/
│   ├── primary/
│   │   ├── rest/          # REST controllers
│   │   └── websocket/     # WebSocket gateways and sockets
│   └── secondary/
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
│       └── typeorm/
└── config/
```

### Development Environment

```text
.devcontainer/
└── devcontainer.json
.vscode/
```

**Structure Decision**: The structure follows the backend repository as implemented and continues to follow hexagonal architecture principles. The `core` layer contains entities, DTOs, ports, services, and validation rules. The `adapters` layer is split into `primary` adapters for HTTP and WebSocket entrypoints and `secondary` adapters for Gemini Live and TypeORM persistence. Root-level modules (`AppModule.ts`, `ConversationModule.ts`, `ConversationEventDispatcherModule.ts`, and `Main.ts`) compose those pieces into the NestJS application.

**Note on Unified Service**: The `gemini-live` adapter implements the live-session integration behind the secondary ports, while the `typeorm` adapter owns database persistence. This preserves hexagonal boundaries while matching the backend's concrete module layout.

This separation keeps the core logic independent of external concerns while aligning the documented structure with the backend repository used for implementation. The `test` directory mirrors the exercised adapter and config areas that exist in the backend today.
