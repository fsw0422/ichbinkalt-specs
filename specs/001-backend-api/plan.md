# Implementation Plan: AI Conversation API

**Branch**: `001-ai-conversation-api` | **Date**: 2025-11-09 | **Updated**: 2025-11-14 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/home/fsw0422/projects/ichbinkalt-backend/specs/001-ai-conversation-api/spec.md`
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
**Storage**: In-memory (volatile session storage).
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
specs/001-ai-conversation-api/
├── plan.md                # This file (/speckit.plan command output)
├── research.md            # Phase 0 output (/speckit.plan command)
├── data-model.md          # Phase 1 output (/speckit.plan command)
├── contracts/             # Phase 1 output (/speckit.plan command)
└── tasks.md               # Phase 2 output (/speckit.tasks command - NOT created by /speckit.plan)
```

### CI/CD

```text
cicd/
├── builder/
│   ├── Dockerfile         # Builder container
│   └── run.sh             # Build script
├── ci/
│   ├── Dockerfile         # CI container
│   └── run.sh             # CI script
├── cd/
│   ├── Dockerfile         # CD container
│   └── run.sh             # CD script
Dockerfile                 # Docker containerization
```

### Infrastructure

```text
iac/                       # Infrastructure as Code (Terraform for provisioning cloud resources)
```

### Source Code
```text
src/
├── core/
│   ├── entities/          # Domain entities (User, Conversation, Message)
│   ├── ports/             # Domain interfaces (repositories, external services)
│   ├── services/          # Application services (business logic)
│   └── dtos/              # Data transfer objects
├── adapters/
│   ├── primary/
│   │   ├── rest/          # REST controllers (entry points)
│   │   └── websocket/     # WebSocket gateways
│   └── secondary/         # External services integrations
│       ├── repositories/  # In-memory storage implementation
│       └── gemini-native-audio/  # Unified Gemini Native Audio service
├── config/                # Configuration, environment setup
└── main.ts                # Application bootstrap
test/
├── core/
│   ├── entities/
│   ├── ports/
│   ├── services/
│   └── dtos/
├── adapters/
│   ├── primary/
│   │   ├── rest/
│   │   └── websocket/
│   └── secondary/
│       ├── repositories/
│       └── gemini-native-audio/
└── config/
```

### Development Environment

```text
.vscode/
├── launch.json
└── settings.json
.devcontainer/
├── devcontainer.json      # Using cicd/builder/Dockerfile as base image
└── post-create.sh
```

**Structure Decision**: The structure explicitly follows hexagonal architecture principles. The `core` layer contains domain entities, ports (interfaces), services (business logic), and DTOs. The `adapters` layer is divided into `primary` (entry points like REST controllers and WebSocket gateways) and `secondary` (external dependencies).

**Note on Unified Service**: The `gemini-native-audio` adapter implements all three service ports (`ISpeechToTextService`, `ITextToSpeechService`, and `ILiveLesson`) in a single service class. This is a deliberate architectural decision that takes advantage of Gemini's native audio capabilities while maintaining the hexagonal architecture principles through the port interfaces. The core domain logic remains independent and can be tested with mock implementations of these ports.

This separation ensures the core logic remains independent of external concerns, improving testability and maintainability. The `test` directory mirrors the `src` structure to facilitate organized unit and integration testing.
