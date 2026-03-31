````markdown
# Tasks: AI Conversation API

**Input**: Design documents from `/home/fsw0422/ichbinkalt-backend/specs/001-ai-conversation-api/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/openapi.yaml

**Tests**: Tests are NOT explicitly requested in the specification, so test tasks are OMITTED.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1)
- Include exact file paths in descriptions

## Path Conventions

- Single project at repository root
- Source code: `src/`
- Tests: `test/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Create NestJS project structure at repository root with src/ and test/ directories
- [ ] T002 Initialize Node.js project with package.json and install NestJS dependencies (@nestjs/core, @nestjs/common, @nestjs/platform-express, @nestjs/platform-socket.io, @nestjs/websockets)
- [ ] T003 [P] Configure TypeScript with tsconfig.json for Node.js >= 18 target
- [ ] T004 [P] Configure ESLint and Prettier for code quality
- [ ] T005 [P] Setup Jest testing framework in test/ directory
- [ ] T006 [P] Setup Docker configuration with Dockerfile at repository root
- [ ] T007 [P] Create .gitignore file for Node.js project

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

 - [ ] T008 Create hexagonal architecture folder structure: src/core/{entities,ports,services,dtos}, src/adapters/{primary,secondary}
 - [ ] T009 [P] Setup configuration module in src/config/ConfigModule.ts for environment variables
 - [ ] T010 [P] Create environment configuration service in src/config/ConfigService.ts
 - [ ] T011 [P] Implement structured logging with correlation IDs in src/config/LoggerService.ts
 - [ ] T012 [P] Create health check endpoint in src/adapters/primary/rest/HealthController.ts
 - [ ] T013 [P] Setup OpenAPI/Swagger documentation configuration in src/Main.ts
 - [ ] T014 [P] Create global exception filter in src/config/ExceptionFilter.ts
 - [ ] T015 [P] Setup graceful shutdown handler in src/Main.ts
 - [ ] T016 Create base application module structure in src/AppModule.ts

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: User Story 1 - Interactive Conversation API (Priority: P1) 🎯 MVP

**Goal**: Enable client applications to initiate, conduct, and terminate interactive AI conversation sessions via WebSocket with real-time transcription and AI responses.

**Independent Test**: A client application can successfully create a conversation via POST /api/v1/conversations, connect to the WebSocket, stream German audio, receive real-time transcription updates, get AI text and audio responses, and gracefully terminate the session.

### Core Domain Entities for User Story 1

 - [ ] T017 [P] [US1] Create User entity interface in src/core/entities/UserEntity.ts
 - [ ] T018 [P] [US1] Create Message entity class in src/core/entities/MessageEntity.ts
 - [ ] T019 [P] [US1] Create Conversation entity class in src/core/entities/ConversationEntity.ts

### Domain DTOs for User Story 1

 - [ ] T020 [P] [US1] Create CreateConversationDto in src/core/dtos/CreateConversationDto.ts
 - [ ] T021 [P] [US1] Create ConversationResponseDto in src/core/dtos/ConversationResponseDto.ts
 - [ ] T022 [P] [US1] Create TranscriptionUpdateDto in src/core/dtos/TranscriptionUpdateDto.ts
 - [ ] T023 [P] [US1] Create AIResponseDto in src/core/dtos/AIResponseDto.ts

### Port Interfaces for User Story 1

 - [ ] T025 [P] [US1] Create IConversationRepository port interface in src/core/ports/IConversationRepository.ts
 - [ ] T028 [P] [US1] Create ILiveLesson port interface in src/core/ports/ILiveLesson.ts

### In-Memory Repository for User Story 1

 - [ ] T029 [US1] Implement InMemoryConversationRepository in src/adapters/secondary/repositories/InMemoryConversationRepository.ts (depends on T019, T025)

### External Service Adapters for User Story 1

 - [ ] T030 [P] [US1] Create Gemini Live LLM adapter module in src/adapters/secondary/gemini-live/GeminiLiveModule.ts
 - [ ] T031 [US1] Implement GeminiLiveService with German persona configuration in src/adapters/secondary/gemini-live/GeminiLiveService.ts (depends on T028, T030)

### Transcription Implementation for User Story 1

 - [ ] T057 [US1] Enable inputAudioTranscription and outputAudioTranscription in GeminiLiveService session setup (depends on T031)
 - [ ] T058 [US1] Parse inputTranscription messages from Gemini Live server responses in GeminiLiveService (depends on T057)
 - [ ] T059 [US1] Parse outputTranscription messages from Gemini Live server responses in GeminiLiveService (depends on T057)
 - [ ] T060 [US1] Add transcription callback types (InputTranscriptionCallback, OutputTranscriptionCallback) to AISessionEntity (depends on T058, T059)
 - [ ] T061 [US1] Implement addInputTranscriptionCallback/removeInputTranscriptionCallback in IAISessionRepository (depends on T060)
 - [ ] T062 [US1] Implement addOutputTranscriptionCallback/removeOutputTranscriptionCallback in IAISessionRepository (depends on T060)
 - [ ] T063 [US1] Emit input_transcription WebSocket events to clients in LessonGateway (depends on T061)
 - [ ] T064 [US1] Emit output_transcription WebSocket events to clients in LessonGateway (depends on T062)
 - [ ] T065 [US1] Persist user transcriptions to ConversationRepository as Message entities (depends on T063)
 - [ ] T066 [US1] Persist AI transcriptions to ConversationRepository as Message entities (depends on T064)

### Services for User Story 1

 - [ ] T032 [US1] Implement CreateConversationService in src/core/services/CreateConversationService.ts (depends on T019, T029)
 - [ ] T033 [US1] Implement ProcessAudioStreamService in src/core/services/ProcessAudioStreamService.ts (depends on T029, T031)
 - [ ] T034 [US1] Implement GenerateAIResponseService in src/core/services/GenerateAIResponseService.ts (depends on T029, T031)
 - [ ] T035 [US1] Implement TerminateConversationService in src/core/services/TerminateConversationService.ts (depends on T029)

### Primary Adapters (REST Controllers) for User Story 1

- [ ] T036 [US1] Create ConversationsController for REST API in src/adapters/primary/rest/ConversationsController.ts (depends on T032)

### Session Management & Edge Cases for User Story 1

 - [ ] T037 [US1] Implement 5-minute session timeout mechanism in src/core/services/ProcessAudioStreamService.ts (depends on T033)

 - [ ] T039 [US1] Configure AI to enforce German/English only conversation in src/adapters/secondary/gemini-live/GeminiLiveService.ts (depends on T031)

### Integration & Validation for User Story 1

 - [ ] T040 [US1] Wire all services and adapters in ConversationModule in src/adapters/ConversationModule.ts (depends on T032-T036)
 - [ ] T036 [US1] Register ConversationModule in root AppModule.ts (depends on T040)
 - [ ] T042 [US1] Add OpenAPI documentation for POST /api/v1/conversations endpoint (depends on T036)
 - [ ] T043 [US1] Validate conversation creation returns conversation_id and websocket_url per contract (depends on T036)
 - [ ] T044 [US1] Add error handling for missing or invalid API keys (depends on T032, T034, T031)
 - [ ] T045 [US1] Add request/response logging with correlation IDs for debugging (depends on T036)

**Checkpoint**: At this point, User Story 1 should be fully functional and testable independently. The backend can handle full conversation lifecycle: creation, real-time audio streaming, transcription, AI responses, and session termination.

---

## Phase 4: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories and production readiness

 - [ ] T046 [P] Create comprehensive README.md at repository root with setup instructions
 - [ ] T047 [P] Document API usage examples in docs/api-examples.md
 - [ ] T048 [P] Add performance monitoring for latency metrics (transcription <500ms, AI response <3s) in src/config/MetricsService.ts
 - [ ] T049 [P] Implement rate limiting for concurrent user protection (100+ concurrent conversations) in src/config/RateLimiterGuard.ts
 - [ ] T050 [P] Setup CI/CD pipeline structure in cicd/ directory per plan.md
 - [ ] T051 [P] Create infrastructure as code with Terraform scaffolding in infra/ directory
 - [ ] T052 [P] Security audit: validate HTTPS configuration and token-based auth preparation
 - [ ] T053 Performance testing: verify 95% transcription accuracy (WER <10%)
 - [ ] T054 Performance testing: verify p95 transcription latency <500ms
 - [ ] T055 Performance testing: verify p95 AI response latency <3s
 - [ ] T056 Load testing: verify support for 100 concurrent conversations

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Story 1 (Phase 3)**: Depends on Foundational phase completion
- **Polish (Phase 4)**: Depends on User Story 1 completion

### User Story 1 Internal Dependencies

**Critical Path**:
 1. Entities (T017-T019) → Foundation for all other components
 2. DTOs (T020-T024) → Required by services and adapters
 3. Port Interfaces (T025-T028) → Required by services and implementations
 4. Repository (T029) → Required by services
 5. External Services (T030-T031) → Required by services
 6. Services (T032-T035) → Required by controllers
 7. Controllers (T036) → Primary entry points
 8. Session Management (T037, T038-T039) → Edge case handling
 9. Integration (T040-T042, T043, T044-T045) → Final wiring and validation

**Within User Story 1**:
- Entities and DTOs can be created in parallel
- Port interfaces can be created in parallel
- External service modules can be created in parallel
- Service implementations depend on their respective modules
- Services depend on entities, repository, and service interfaces
- Controllers depend on services
- Session management enhances existing services
- Integration tasks finalize the story

### Parallel Opportunities

**Phase 1 (Setup)**: Tasks T003, T004, T005, T006, T007 can run in parallel after T001-T002

 **Phase 2 (Foundational)**: Tasks T009-T015 can run in parallel after T008

**Phase 3 (User Story 1)**:
- Entities: T017, T018, T019 can run in parallel
- DTOs: T020-T024 can run in parallel
- Port Interfaces: T025-T028 can run in parallel
 - Service Modules: T030 can run in parallel
 - - After modules: T032, T034 can run in parallel (each depends only on their respective module)
- Services: T032-T035 can run in parallel after dependencies are met
- Controllers: T036 can run after dependencies are met

 **Phase 4 (Polish)**: Tasks T046-T052 can run in parallel, then T053-T057 follow

---

## Parallel Example: User Story 1 Kickoff

```bash
# After Foundation (Phase 2) completes, launch these in parallel:

# Batch 1: Core Domain Models
 Task T017: "Create User entity interface in src/core/entities/UserEntity.ts"
 Task T018: "Create Message entity class in src/core/entities/MessageEntity.ts"
 Task T019: "Create Conversation entity class in src/core/entities/ConversationEntity.ts"

# Batch 2: DTOs (after or during Batch 1)
 Task T020: "Create CreateConversationDto in src/core/dtos/CreateConversationDto.ts"
 Task T021: "Create ConversationResponseDto in src/core/dtos/ConversationResponseDto.ts"
 Task T022: "Create TranscriptionUpdateDto in src/core/dtos/TranscriptionUpdateDto.ts"
 Task T023: "Create AIResponseDto in src/core/dtos/AIResponseDto.ts"


# Batch 3: Port Interfaces (after or during Batch 1-2)
 Task T025: "Create IConversationRepository port interface"
 Task T026: "Create ISpeechToTextService port interface"
 Task T027: "Create ITextToSpeechService port interface"
 Task T028: "Create ILiveLesson port interface"

# Batch 4: External Service Modules (after Batch 3)
 Task T030: "Create Gemini Live LLM adapter module"
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

This feature specification contains only ONE user story (US1), making the entire Phase 3 the MVP.

 1. **Complete Phase 1**: Setup → ~7 tasks → Project structure ready
 2. **Complete Phase 2**: Foundational → ~9 tasks → Core infrastructure ready
 3. **Complete Phase 3**: User Story 1 → ~29 tasks → Full conversation API functional
4. **STOP and VALIDATE**:
   - Test conversation creation via REST API
   - Verify conversation creation returns conversation_id and websocket_url per contract
5. **Deploy MVP**: Backend ready for integration with client applications

### Production Readiness

After MVP validation:
1. **Complete Phase 4**: Polish & Cross-Cutting → ~11 tasks
2. Validate performance metrics (latency, accuracy, concurrency)
3. Security audit and monitoring
4. Deploy to production environment

### Single Developer Strategy

Follow sequential phases:
1. Setup (1-2 days)
2. Foundational (1-2 days)
3. User Story 1 Core (3-5 days)
   - Start with entities, DTOs, ports (parallelizable if desired)
   - Then repositories and external services
   - Then services
   - Finally controllers
4. User Story 1 Polish (1-2 days)
5. Cross-cutting concerns (1-2 days)

**Total Estimate**: ~8-13 days for MVP

### Multiple Developer Strategy

With 2-3 developers:
1. **Team**: Complete Setup + Foundational together (1-2 days)
2. **Split User Story 1 by layer**:
  - **Developer A**: Core domain (entities, DTOs, ports, repository) → T017-T029
  - **Developer B**: External services (Speech-to-Text, Text-to-Speech, Conversational AI adapters) → T030-T031
  - **Developer C**: Services and integration → T032-T035, T040-T045
3. **Converge**: Primary adapters (controllers) → T036, T037, T038-T039
4. **Team**: Polish and validation → Phase 4

**Total Estimate**: ~5-8 days for MVP with team

---

## Notes

- **[P] marking**: Tasks with different file outputs and no sequential dependencies
- **[US1] label**: All implementation tasks belong to the single user story
- **No tests included**: Testing is not explicitly required in the specification
- **Constitution compliance**: All tasks align with constitutional principles (security, observability, resilience)
- **Hexagonal architecture**: Strict separation between core domain and adapters
- **Real-time focus**: Critical performance requirements (<500ms transcription, <3s AI response)
- **Edge cases covered**: Session timeouts
- **Commit strategy**: Commit after each task or logical batch (e.g., all entities, all DTOs)
- **Validation gates**: Test at checkpoints before proceeding to next phase

---

## Summary

- - **Total Tasks**: 56 tasks
- - **Task Distribution**:
-   - Phase 1 (Setup): 7 tasks
-   - Phase 2 (Foundational): 9 tasks
-   - Phase 3 (User Story 1): 29 tasks
-   - Phase 4 (Polish): 11 tasks
- - **User Stories**: 1 (US1 - Interactive Conversation API)
- - **MVP Scope**: Complete Phase 1-3 (45 tasks)
- **Parallel Opportunities**:
  - Phase 1: 6 tasks can run in parallel
  - Phase 2: 7 tasks can run in parallel
  - Phase 3: Multiple batches of 3-5 tasks can run in parallel
  - Phase 4: 7 tasks can run in parallel
- **Independent Test Criteria**:
  - US1: Client can create conversation via REST API and receive conversation_id and websocket_url
- **Critical Path**: Setup → Foundational (BLOCKS US1) → User Story 1 → Polish
- **Performance Requirements**:
  - 95% transcription accuracy (WER <10%)
  - <500ms transcription latency (p95)
  - <3s AI response latency (p95)
  - Support 100+ concurrent conversations

````
