# Tasks: AI Conversation API

**Input**: Design documents from `/home/ksp/projects/ichbinkalt-specs-001-backend-api/specs/001-backend-api/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/openapi.yaml

**Tests**: Tests are NOT explicitly requested in the specification, so test tasks are OMITTED.

**Organization**: Tasks are grouped by shared prerequisites and user story to enable independent implementation and testing.

## Format: `[ID] [P?] [Scope] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Scope]**: Which shared phase or user story this task belongs to (e.g., Shared, US1)
- Include exact file paths in descriptions

## Path Conventions

- Single project at repository root
- Application code: `src/`
- Tests: `test/`
- Database migrations: `db/`
- Delivery scripts: `ci.sh`, `cd.sh`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [ ] T001 Create NestJS project structure at repository root with src/, test/, db/, ci.sh, and cd.sh
- [ ] T002 Initialize Node.js project with package.json and install NestJS dependencies (@nestjs/core, @nestjs/common, @nestjs/platform-express, @nestjs/platform-socket.io, @nestjs/websockets)
- [ ] T003 [P] Configure TypeScript with tsconfig.json for Node.js >= 18 target
- [ ] T004 [P] Configure ESLint and Prettier for code quality
- [ ] T005 [P] Setup Jest testing framework in test/ directory
- [ ] T006 [P] Setup Docker configuration with Dockerfile, Dockerfile.builder, Dockerfile.dbmigrator, and docker-compose.yml at repository root
- [ ] T007 [P] Create .gitignore file for Node.js project

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

 - [ ] T008 Create backend-aligned folder structure: src/core/{dtos,entities,ports/{primary,secondary},services,validation}, src/adapters/{primary/{rest,websocket},secondary/{gemini-live,typeorm}}, src/config, and db/changelog
 - [ ] T009 [P] Setup configuration module in src/config/ConfigModule.ts for environment variables
 - [ ] T010 [P] Create environment configuration service in src/config/ConfigService.ts
 - [ ] T011 [P] Implement structured logging with correlation IDs in src/config/LoggerService.ts
 - [ ] T012 [P] Create health check endpoint in src/adapters/primary/rest/HealthController.ts
 - [ ] T013 [P] Setup OpenAPI/Swagger documentation configuration in src/Main.ts
 - [ ] T014 [P] Create global exception filter in src/config/GlobalExceptionFilter.ts
 - [ ] T015 [P] Setup graceful shutdown handler in src/Main.ts
 - [ ] T016 Create base application module structure in src/AppModule.ts, src/ConversationModule.ts, and src/ConversationEventDispatcherModule.ts

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: Core User Domain (Shared Prerequisites)

**Purpose**: Establish the shared user model and services required by authentication and user-facing APIs.

**⚠️ CRITICAL**: Authentication and profile work depend on this phase. Conversation work also relies on the user model created here.

### Shared User Domain Entities

 - [ ] T017 [P] [Shared] Create User entity in src/core/entities/UserEntity.ts with userId, displayName, email, createdAt, updatedAt, lastActiveAt
 - [ ] T067 [P] [Shared] Create UserLearningProfile entity in src/core/entities/UserLearningProfileEntity.ts with userId, learningLevel (CEFR A1–C2), nativeLanguage, learningGoals, correctionStyle, topicsOfInterest
 - [ ] T068 [P] [Shared] Create UserUsageStats entity in src/core/entities/UserUsageStatsEntity.ts with userId, totalConversationCount, totalPracticeMinutes, currentStreak, longestStreak
 - [ ] T069 [P] [Shared] Create UserPreferences entity in src/core/entities/UserPreferencesEntity.ts with userId, timezone, uiLanguage

### Shared User Ports and Services

 - [ ] T073 [P] [Shared] Create IUserRepository port interface in src/core/ports/secondary/IUserRepository.ts (create, findById, update, findByEmail)
 - [ ] T074 [P] [Shared] Create IUserService port interface in src/core/ports/primary/IUserService.ts
 - [ ] T075 [Shared] Implement UserRepository in src/adapters/secondary/typeorm/UserRepository.ts with mapping between domain and ORM entities, including eager loading of learningProfile, usageStats, preferences (depends on T017, T067-T069, T073)
 - [ ] T076 [Shared] Implement UserService in src/core/services/UserService.ts: create user with default sub-records (used by AuthService), get full user profile, update displayName, update learning profile, update preferences (depends on T073, T074)

**Checkpoint**: Shared user domain is ready. Authentication and profile implementation can proceed.

---

## Phase 4: User Story 3 - Authentication: Magic Link & Passkey (Priority: P1)

**Goal**: Enable passwordless authentication via magic links and WebAuthn passkeys before conversation features are built on top.

**Independent Test**: A new user signs up via magic link, registers a passkey, logs out, and logs back in with either passkey or magic link fallback.

### Domain Entities for User Story 3

 - [ ] T084 [P] [US3] Create MagicLinkTokenEntity in src/core/entities/MagicLinkTokenEntity.ts (tokenId, email, tokenHash, expiresAt, usedAt, createdAt)
 - [ ] T085 [P] [US3] Create UserPasskeyEntity in src/core/entities/UserPasskeyEntity.ts (passkeyId, userId, credentialId, publicKey, counter, deviceName, createdAt, lastUsedAt)
 - [ ] T086 [P] [US3] Create RefreshTokenEntity in src/core/entities/RefreshTokenEntity.ts (tokenId, userId, tokenHash, expiresAt, revokedAt, createdAt)

### Domain DTOs for User Story 3

 - [ ] T087 [P] [US3] Create MagicLinkRequestDto with Zod schema in src/core/dtos/MagicLinkRequestDto.ts (email)
 - [ ] T088 [P] [US3] Create VerifyMagicLinkDto with Zod schema in src/core/dtos/VerifyMagicLinkDto.ts (token)
 - [ ] T089 [P] [US3] Create AuthResponseDto in src/core/dtos/AuthResponseDto.ts (accessToken, refreshToken, userId, passkeyRegistrationRequired)
 - [ ] T090 [P] [US3] Create RefreshTokenRequestDto with Zod schema in src/core/dtos/RefreshTokenRequestDto.ts (refreshToken)

### Port Interfaces for User Story 3

 - [ ] T091 [P] [US3] Create IMagicLinkTokenRepository port in src/core/ports/secondary/IMagicLinkTokenRepository.ts (create, findByTokenHash, markUsed)
 - [ ] T092 [P] [US3] Create IUserPasskeyRepository port in src/core/ports/secondary/IUserPasskeyRepository.ts (create, findByUserId, findByCredentialId, updateCounter)
 - [ ] T093 [P] [US3] Create IRefreshTokenRepository port in src/core/ports/secondary/IRefreshTokenRepository.ts (create, findByTokenHash, revoke, revokeAllForUser)
 - [ ] T094 [P] [US3] Create IEmailService port in src/core/ports/secondary/IEmailService.ts (sendMagicLinkEmail)
 - [ ] T095 [P] [US3] Create IAuthService port in src/core/ports/primary/IAuthService.ts

### ORM Entities for User Story 3

 - [ ] T096 [P] [US3] Create MagicLinkTokenOrmEntity in src/adapters/secondary/typeorm/entities/MagicLinkTokenOrmEntity.ts
 - [ ] T097 [P] [US3] Create UserPasskeyOrmEntity in src/adapters/secondary/typeorm/entities/UserPasskeyOrmEntity.ts
 - [ ] T098 [P] [US3] Create RefreshTokenOrmEntity in src/adapters/secondary/typeorm/entities/RefreshTokenOrmEntity.ts

### Database Migrations for User Story 3

 - [ ] T099 [P] [US3] Create Liquibase migration for magic_link_tokens table in db/changelog/
 - [ ] T100 [P] [US3] Create Liquibase migration for user_passkeys table in db/changelog/
 - [ ] T101 [P] [US3] Create Liquibase migration for refresh_tokens table in db/changelog/

### Persistence Adapters for User Story 3

 - [ ] T102 [US3] Implement MagicLinkTokenRepository in src/adapters/secondary/typeorm/MagicLinkTokenRepository.ts (depends on T091, T096)
 - [ ] T103 [US3] Implement UserPasskeyRepository in src/adapters/secondary/typeorm/UserPasskeyRepository.ts (depends on T092, T097)
 - [ ] T104 [US3] Implement RefreshTokenRepository in src/adapters/secondary/typeorm/RefreshTokenRepository.ts (depends on T093, T098)

### Email Adapter for User Story 3

 - [ ] T105 [US3] Create stub EmailService implementation in src/adapters/secondary/email/StubEmailService.ts behind IEmailService port (depends on T094). Logs magic link to console for local development.

### Services for User Story 3

 - [ ] T106 [US3] Implement AuthService in src/core/services/AuthService.ts: magic link request/verify (calls UserService.create for new users), JWT issuance (access+refresh), token refresh, logout/revocation, rate limiting (5/email/hour) (depends on T091-T095, T073, T076)
 - [ ] T107 [US3] Implement PasskeyService in src/core/services/PasskeyService.ts: registration options, register credential, authentication options, authenticate. Use @simplewebauthn/server (depends on T092, T095)

### Auth Guard for User Story 3

 - [ ] T108 [US3] Implement JwtAuthGuard in src/config/JwtAuthGuard.ts to validate Bearer tokens on protected routes (depends on T106)

### Primary Adapters (REST Controllers) for User Story 3

 - [ ] T109 [US3] Create AuthController in src/adapters/primary/rest/AuthController.ts with endpoints: POST /api/v1/auth/magic-link, POST /api/v1/auth/verify-magic-link, POST /api/v1/auth/refresh, POST /api/v1/auth/logout (depends on T106)
 - [ ] T110 [US3] Create PasskeyController in src/adapters/primary/rest/PasskeyController.ts with endpoints: POST registration-options, POST register, POST authentication-options, POST authenticate (depends on T107)

### Integration & Validation for User Story 3

 - [ ] T111 [US3] Create AuthModule in src/AuthModule.ts and wire AuthService, PasskeyService, repositories, EmailService, controllers (depends on T102-T110)
 - [ ] T112 [US3] Register AuthModule in root AppModule.ts (depends on T111)
 - [ ] T114 [US3] Add OpenAPI documentation and security scheme decorators to auth endpoints (depends on T109, T110)
 - [ ] T115 [US3] Validate magic link verify creates new User with defaults when email is not found (depends on T109)
 - [ ] T116 [US3] Validate 429 rate limiting on magic link requests (depends on T109)
 - [ ] T117 [US3] Validate 401 on expired/used magic link token (depends on T109)
 - [ ] T118 [US3] Add JWT_SECRET and WEBAUTHN_RP_ID, WEBAUTHN_RP_NAME, WEBAUTHN_ORIGIN to ConfigService (JWT_SECRET injected via AWS Secrets Manager in deployed environments, .env locally) (depends on T106, T107)

**Checkpoint**: User Story 3 complete. Users can sign up via magic link, register passkeys, and obtain JWTs for protected APIs.

---

## Phase 5: User Story 2 - User Profile Management API (Priority: P1)

**Goal**: Enable authenticated users to retrieve their full profile and update learning preferences and app preferences via REST APIs. User creation is handled by magic link verification (User Story 3).

**Independent Test**: An authenticated client can retrieve full user profile via GET /api/v1/users/:userId, update display name, learning profile, and preferences via PUT.

### Domain DTOs for User Story 2

 - [ ] T070 [P] [US2] Create UpdateUserDto with Zod schema in src/core/dtos/UpdateUserDto.ts (displayName)
 - [ ] T071 [P] [US2] Create UpdateLearningProfileDto with Zod schema in src/core/dtos/UpdateLearningProfileDto.ts (learningLevel, nativeLanguage, learningGoals, correctionStyle, topicsOfInterest)
 - [ ] T072 [P] [US2] Create UpdatePreferencesDto with Zod schema in src/core/dtos/UpdatePreferencesDto.ts (timezone, uiLanguage)

### Primary Adapters (REST Controllers) for User Story 2

 - [ ] T077 [US2] Create UserController in src/adapters/primary/rest/UserController.ts with endpoints: GET /api/v1/users/:userId (full profile with nested learningProfile, usageStats, preferences), PUT /api/v1/users/:userId, PUT /api/v1/users/:userId/learning-profile, PUT /api/v1/users/:userId/preferences (depends on T076)

### Integration & Validation for User Story 2

 - [ ] T078 [US2] Create UserModule in src/UserModule.ts and wire UserService, UserRepository, UserController (depends on T075-T077)
 - [ ] T079 [US2] Register UserModule in root AppModule.ts (depends on T078)
 - [ ] T080 [US2] Add OpenAPI documentation decorators to UserController endpoints (depends on T077)
 - [ ] T081 [US2] Validate all GET/PUT endpoints return 404 for non-existent userId (depends on T077)
 - [ ] T113 [US2] Apply JwtAuthGuard to UserController (depends on T077, T108)

**Checkpoint**: User Story 2 complete. Users can view their full profile, configure learning preferences, and set app preferences through authenticated endpoints.

---

## Phase 6: User Story 1 - Interactive Conversation API (Priority: P1) 🎯 MVP

**Goal**: Enable client applications to initiate, conduct, and terminate interactive AI conversation sessions via WebSocket with real-time transcription and AI responses.

**Independent Test**: A client application can successfully create a conversation via POST /api/v1/conversations, connect to the WebSocket, stream German audio, receive real-time transcription updates, get AI text and audio responses, and gracefully terminate the session.

### Core Domain Entities for User Story 1

 - [ ] T018 [P] [US1] Create Message entity class in src/core/entities/MessageEntity.ts
 - [ ] T019 [P] [US1] Create Conversation entity class in src/core/entities/ConversationEntity.ts

### Domain DTOs for User Story 1

 - [ ] T020 [P] [US1] Create CreateConversationDto in src/core/dtos/CreateConversationDto.ts
 - [ ] T021 [P] [US1] Create ConversationResponseDto in src/core/dtos/ConversationResponseDto.ts
 - [ ] T022 [P] [US1] Create TranscriptionUpdateDto in src/core/dtos/TranscriptionUpdateDto.ts
 - [ ] T023 [P] [US1] Create AIResponseDto in src/core/dtos/AIResponseDto.ts

### Port Interfaces for User Story 1

 - [ ] T025 [P] [US1] Create IConversationRepository port interface in src/core/ports/secondary/IConversationRepository.ts
 - [ ] T028 [P] [US1] Create ILiveSession port interface in src/core/ports/secondary/ILiveSession.ts

### Persistence Adapter for User Story 1

 - [ ] T029 [US1] Implement ConversationRepository in src/adapters/secondary/typeorm/ConversationRepository.ts (depends on T019, T025)

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
 - [ ] T063 [US1] Emit input_transcription WebSocket events to clients in ConversationGateway (depends on T061)
 - [ ] T064 [US1] Emit output_transcription WebSocket events to clients in ConversationGateway (depends on T062)
 - [ ] T065 [US1] Persist user transcriptions to ConversationRepository as Message entities (depends on T063)
 - [ ] T066 [US1] Persist AI transcriptions to ConversationRepository as Message entities (depends on T064)

### Services for User Story 1

 - [ ] T032 [US1] Implement ConversationService in src/core/services/ConversationService.ts for conversation lifecycle management (depends on T019, T025, T028-T031)
 - [ ] T033 [US1] Implement WebSocket audio streaming flow in src/adapters/primary/websocket/ConversationGateway.ts, including `audio_chunk` handling and `audio_stream_end` signaling (depends on T029, T031, T032)
 - [ ] T034 [US1] Implement Gemini Live response handling in src/adapters/secondary/gemini-live/GeminiLiveService.ts (depends on T028, T030)
 - [ ] T035 [US1] Implement conversation event dispatch integration in src/adapters/primary/websocket/ConversationEventDispatcher.ts (depends on T033, T034)

### Primary Adapters (REST Controllers) for User Story 1

- [ ] T036 [US1] Create ConversationController for REST API in src/adapters/primary/rest/ConversationController.ts (depends on T032)

### Session Management & Edge Cases for User Story 1

 - [ ] T037 [US1] Implement 5-minute session timeout mechanism in src/adapters/primary/websocket/ConversationGateway.ts (depends on T033)

 - [ ] T039 [US1] Configure AI to enforce German/English only conversation in src/adapters/secondary/gemini-live/GeminiLiveService.ts (depends on T031)

### Integration & Validation for User Story 1

 - [ ] T040 [US1] Wire all services and adapters in ConversationModule in src/ConversationModule.ts (depends on T032-T036)
 - [ ] T041 [US1] Register ConversationModule in root AppModule.ts (depends on T040)
 - [ ] T121 [US1] Apply JwtAuthGuard to ConversationController (depends on T036, T108)
 - [ ] T042 [US1] Add OpenAPI documentation for POST /api/v1/conversations endpoint (depends on T036)
 - [ ] T043 [US1] Validate conversation creation returns conversation_id and websocket_url per contract (depends on T036)
 - [ ] T044 [US1] Add error handling for missing or invalid API keys (depends on T032, T034, T031)
 - [ ] T045 [US1] Add request/response logging with correlation IDs for debugging (depends on T036)

**Checkpoint**: At this point, User Story 1 should be fully functional and testable independently. The backend can handle full conversation lifecycle: creation, real-time audio streaming, transcription, AI responses, and session termination.

---

## Phase 7: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories and production readiness

 - [ ] T046 [P] Create comprehensive README.md at repository root with setup instructions
 - [ ] T047 [P] Document API usage examples in docs/api-examples.md
 - [ ] T048 [P] Add performance monitoring for latency metrics (transcription <500ms, AI response <3s) in src/config/MetricsService.ts
 - [ ] T049 [P] Implement rate limiting for concurrent user protection (100+ concurrent conversations) in src/config/RateLimiterGuard.ts
 - [ ] T050 [P] Setup CI/CD script structure in ci.sh and cd.sh per plan.md
 - [ ] T051 [P] Create database migration scaffolding in db/run-postgres-migrations.sh and db/changelog/ per plan.md
 - [ ] T119 [P] Create Terraform secrets-manager module in infra/aws/modules/secrets-manager/ (aws_secretsmanager_secret + aws_secretsmanager_secret_version) and add per-environment terragrunt.hcl entries
 - [ ] T120 [P] Update ECS Terraform module to reference Secrets Manager ARN via container definition secrets block (valueFrom) and grant Task Execution Role secretsmanager:GetSecretValue
 - [ ] T122 [P] Install tflocal as a CI and local dev dependency: add `pip install terraform-local` to ci.sh setup steps and document in README.md (requires Python + terraform already present)
 - [ ] T123 [P] Create DynamoDB adapter test setup helper in test/adapters/secondary/dynamodb/setup.ts that starts a LocalStack container via @testcontainers/localstack, sets AWS_ENDPOINT_URL, then runs `tflocal init` and `tflocal apply -auto-approve` against infra/aws/modules/dynamodb/ to provision tables from the production module
 - [ ] T124 [US3] Implement MagicLinkTokenRepository adapter test in test/adapters/secondary/dynamodb/MagicLinkTokenRepository.test.ts: verify create, findByTokenHash, and markUsed against the LocalStack-provisioned table (depends on T102, T123)
 - [ ] T125 [US3] Implement RefreshTokenRepository adapter test in test/adapters/secondary/dynamodb/RefreshTokenRepository.test.ts: verify create, findByTokenHash, and revoke against the LocalStack-provisioned table (depends on T104, T123)
 - [ ] T052 [P] Security audit: validate HTTPS configuration and token-based auth preparation
 - [ ] T053 Performance testing: verify 95% transcription accuracy (WER <10%)
 - [ ] T054 Performance testing: verify p95 transcription latency <500ms
 - [ ] T055 Performance testing: verify p95 AI response latency <3s
 - [ ] T056 Load testing: verify support for 100 concurrent conversations

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all downstream work
- **Core User Domain (Phase 3)**: Depends on Foundational phase completion
- **User Story 3 / Authentication (Phase 4)**: Depends on Foundational and Core User Domain completion
- **User Story 2 / User Profile (Phase 5)**: Depends on Core User Domain and Authentication completion
- **User Story 1 / Conversation (Phase 6)**: Depends on Foundational and Authentication completion. It also consumes the shared user model from Phase 3.
- **Polish (Phase 7)**: Depends on User Stories 1, 2, and 3 completion

### Core User Domain Internal Dependencies

**Critical Path**:
 1. Shared user entities (T017, T067-T069) → Foundation for repository and service work
 2. Shared ports (T073-T074) → Required by implementations
 3. Shared persistence adapter (T075) → Required by user-facing modules
 4. Shared service layer (T076) → Enables auth-driven user creation and profile management

**Within Phase 3**:
- Shared user entities can be created in parallel
- Shared user ports can be created in parallel
- UserRepository depends on shared entities and IUserRepository
- UserService depends on IUserRepository and IUserService

### User Story 3 Internal Dependencies

**Critical Path**:
 1. Auth entities, DTOs, and ports (T084-T095) → Foundation for auth flows
 2. ORM entities and migrations (T096-T101) → Required by repository implementations
 3. Auth repositories and email adapter (T102-T105) → Required by services
 4. Auth services and guard (T106-T108) → Required by controllers and protected routes
 5. Auth controllers and module wiring (T109-T118) → Final delivery and validation

### User Story 2 Internal Dependencies

**Critical Path**:
 1. Profile DTOs (T070-T072) → Define request contracts for profile updates
 2. Shared user service (T076) and auth guard (T108) → Required by the controller layer
 3. User controller and module wiring (T077-T080, T113) → Deliver authenticated profile endpoints
 4. Validation (T081) → Confirms profile API behavior for missing users

### User Story 1 Internal Dependencies

**Critical Path**:
 1. Conversation entities and DTOs (T018-T023) → Foundation for all other conversation components
 2. Conversation port interfaces (T025, T028) → Required by services and implementations
 3. Persistence and external service adapters (T029-T031) → Required by services
 4. Services and WebSocket flow (T032-T035) → Required by controllers and realtime messaging
 5. Controllers and session handling (T036, T037, T039) → Primary entry points and runtime safeguards
 6. Integration and validation (T040-T045, T057-T066, T121) → Final wiring, security, transcription handling, and validation

### Parallel Opportunities

**Phase 1 (Setup)**: Tasks T003, T004, T005, T006, T007 can run in parallel after T001-T002

 **Phase 2 (Foundational)**: Tasks T009-T015 can run in parallel after T008

**Phase 3 (Core User Domain)**:
- Shared entities: T017, T067, T068, T069 can run in parallel
- Shared ports: T073, T074 can run in parallel
- T075 (UserRepository) depends on shared entities and IUserRepository
- T076 (UserService) depends on T073-T074 and is wired with T075 at module level

**Phase 4 (User Story 3)**:
- Entities: T084, T085, T086 can run in parallel
- DTOs: T087-T090 can run in parallel
- Port Interfaces: T091-T095 can run in parallel
- ORM Entities: T096-T098 can run in parallel
- Migrations: T099-T101 can run in parallel
- Repositories: T102-T104 can run in parallel (after ORM entities + ports)
- T106 (AuthService) and T107 (PasskeyService) can run in parallel
- T109 (AuthController) and T110 (PasskeyController) can run in parallel
- T111-T118 (Integration) depend on services + controllers

**Phase 5 (User Story 2)**:
- DTOs: T070-T072 can run in parallel
- T077 (UserController) depends on T076
- T078-T080 depend on T075-T077
- T081 depends on T077
- T113 depends on T077 and JwtAuthGuard from Phase 4

**Phase 6 (User Story 1)**:
- Entities: T018, T019 can run in parallel
- DTOs: T020-T023 can run in parallel
- Port Interfaces: T025, T028 can run in parallel
- Service modules: T030 can run in parallel
- After modules: T032 and T034 can run in parallel (each depends on its respective adapter path)
- Services and WebSocket flow: T032-T035 can run in parallel after dependencies are met
- Controllers: T036 can run after dependencies are met
- Transcription tasks T057-T066 layer on after Gemini Live and gateway plumbing exists

**Phase 7 (Polish)**: Tasks T046-T052 and T119-T120 can run in parallel, then T053-T056 follow

---

## Parallel Example: Core User/Auth Kickoff

```bash
# After Foundational (Phase 2) completes, launch these in parallel:

# Batch 1: Shared User Domain
 Task T017: "Create User entity in src/core/entities/UserEntity.ts"
 Task T067: "Create UserLearningProfile entity in src/core/entities/UserLearningProfileEntity.ts"
 Task T068: "Create UserUsageStats entity in src/core/entities/UserUsageStatsEntity.ts"
 Task T069: "Create UserPreferences entity in src/core/entities/UserPreferencesEntity.ts"

# Batch 2: Shared User Contracts
 Task T073: "Create IUserRepository port interface in src/core/ports/secondary/IUserRepository.ts"
 Task T074: "Create IUserService port interface in src/core/ports/primary/IUserService.ts"

# Batch 3: Authentication Contracts
 Task T087: "Create MagicLinkRequestDto in src/core/dtos/MagicLinkRequestDto.ts"
 Task T088: "Create VerifyMagicLinkDto in src/core/dtos/VerifyMagicLinkDto.ts"
 Task T091: "Create IMagicLinkTokenRepository port in src/core/ports/secondary/IMagicLinkTokenRepository.ts"
 Task T094: "Create IEmailService port in src/core/ports/secondary/IEmailService.ts"

# Batch 4: Shared/User Auth Implementations (after Batch 1-3)
 Task T075: "Implement UserRepository in src/adapters/secondary/typeorm/UserRepository.ts"
 Task T076: "Implement UserService in src/core/services/UserService.ts"
 Task T105: "Create StubEmailService in src/adapters/secondary/email/StubEmailService.ts"
```

---

## Implementation Strategy

### Recommended Delivery Order

 1. **Complete Phase 1**: Setup → project structure ready
2. **Complete Phase 2**: Foundational → backend-aligned infrastructure ready
3. **Complete Phase 3**: Shared user domain → user model and shared services ready
4. **Complete Phase 4**: Authentication → secure sign-up/login flow ready
5. **Complete Phase 5**: User profile management → authenticated profile APIs ready
6. **Complete Phase 6**: Conversation API → realtime conversation features built on authenticated user context
7. **STOP and VALIDATE**:
  - Verify magic link and passkey flows issue valid JWTs
  - Verify authenticated profile endpoints work end-to-end
  - Verify conversation creation returns conversation_id and websocket_url per contract

### Production Readiness

After MVP validation:
1. **Complete Phase 7**: Polish & Cross-Cutting
2. Validate performance metrics (latency, accuracy, concurrency)
3. Security audit and monitoring
4. Deploy to production environment

### Single Developer Strategy

Follow sequential phases:
1. Setup (1-2 days)
2. Foundational (1-2 days)
3. Shared user domain (1-2 days)
4. Authentication (2-4 days)
5. User profile APIs (1-2 days)
6. Conversation API (3-5 days)
7. Cross-cutting concerns (1-2 days)

**Total Estimate**: ~10-18 days for full feature set

### Multiple Developer Strategy

With 2-3 developers:
1. **Team**: Complete Setup + Foundational together (1-2 days)
2. **Split the secured foundation first**:
  - **Developer A**: Shared user domain → T017, T067-T069, T073-T076
  - **Developer B**: Authentication tokens, repositories, and auth controllers → T084-T118
  - **Developer C**: User profile controllers/module and conversation DTOs/ports groundwork → T070-T081, T018-T028
3. **Converge on conversation delivery**: Gemini Live adapters, realtime flow, and gateway integration → T029-T045, T057-T066, T121
4. **Team**: Polish and validation → Phase 7

**Total Estimate**: ~7-11 days for full feature set with team

---

## Notes

- **[P] marking**: Tasks with different file outputs and no sequential dependencies
- **[Shared] label**: Indicates cross-story prerequisites that should be completed before auth/profile/conversation work
- **No tests included**: Testing is not explicitly required in the specification
- **Constitution compliance**: All tasks align with constitutional principles (security, observability, resilience)
- **Hexagonal architecture**: Strict separation between core domain and adapters
- **Real-time focus**: Critical performance requirements (<500ms transcription, <3s AI response)
- **Edge cases covered**: Session timeouts and auth abuse controls
- **Commit strategy**: Commit after each task or logical batch (e.g., all entities, all DTOs)
- **Validation gates**: Test at checkpoints before proceeding to next phase

---

## Summary

- **Total Tasks**: 114 tasks
- **Task Distribution**:
  - Phase 1 (Setup): 7 tasks
  - Phase 2 (Foundational): 9 tasks
  - Phase 3 (Core User Domain): 8 tasks
  - Phase 4 (User Story 3 / Authentication): 35 tasks
  - Phase 5 (User Story 2 / User Profile): 8 tasks
  - Phase 6 (User Story 1 / Conversation): 35 tasks
  - Phase 7 (Polish): 13 tasks
- **User Stories**: 3 plus 1 shared prerequisite phase
- **MVP Scope**: Complete Phases 1-6 for the secured end-to-end platform
- **Parallel Opportunities**:
  - Phase 1: 6 tasks can run in parallel
  - Phase 2: 7 tasks can run in parallel
  - Phase 3: 6 tasks can run in parallel
  - Phase 4: Multiple auth batches can run in parallel
  - Phase 5: DTOs and integration prep can run in parallel
  - Phase 6: Multiple conversation batches can run in parallel
  - Phase 7: 9 tasks can run in parallel
- **Independent Test Criteria**:
  - US3: User can complete magic link and passkey authentication and receive JWTs
  - US2: Authenticated client can manage user profile endpoints
  - US1: Authenticated client can create conversation via REST API and receive conversation_id and websocket_url
- **Critical Path**: Setup → Foundational → Core User Domain → Authentication → User Profile / Conversation → Polish
- **Performance Requirements**:
  - 95% transcription accuracy (WER <10%)
  - <500ms transcription latency (p95)
  - <3s AI response latency (p95)
  - Support 100+ concurrent conversations
