# Tasks: AI Conversation API

**Input**: Design documents from `/home/ksp/projects/ichbinkalt-specs-001-backend-api/specs/001-backend-api/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/openapi.yaml

**Organization**: Tasks are grouped by shared prerequisites and user story to enable independent implementation and testing.

## Format: `[ID] [P?] [Scope] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Scope]**: Which shared phase or user story this task belongs to (e.g., Shared, US1)
- Include exact file paths in descriptions

## Path Conventions

- Single project at repository root
- Application code: `src/`
- Tests: `test/`
- Delivery scripts: `ci.sh`, `cd.sh`

---

## Phase 0: Infrastructure (Terraform/Terragrunt)

**Purpose**: Provision AWS infrastructure using Terraform modules and Terragrunt for DRY, multi-environment management. See [plan.md §Infrastructure](./plan.md) for directory structure and [research.md §4–6](./research.md) for service decisions.

### Root Configuration

 - [ ] T200 Create infra/aws/ directory structure and root terragrunt.hcl with S3 remote state backend pattern (s3://ichbinkalt-tfstate-${account_name}/${region}/${path}/terraform.tfstate) and DynamoDB lock table

### Terraform Modules

 - [x] T201 [P] Create infra/aws/modules/vpc/ module (VPC, public/private subnets across AZs, route tables, IGW, NAT gateway) with variables.tf, main.tf, outputs.tf
 - [x] T202 [P] Create infra/aws/modules/ecr/ module (ECR repository with lifecycle policy) with variables.tf, main.tf, outputs.tf
 - [x] T204 [P] Create infra/aws/modules/dynamodb/ module (single DynamoDB table `ichbinkalt` with pk/sk string keys, PK markers `A#`, `B#`, `C#`, SK markers `A`, `B#`, `C#`, sequential alphabet attribute names `A`, `B`, ... `Z`, `AA`, ..., GSIs on `AA` and `B`, TTL on `AB`, and support for user/email lookup-item patterns; outputs table ARN) with variables.tf, main.tf, outputs.tf
 - [x] T205 [P] Create infra/aws/modules/secrets-manager/ module (Secrets Manager secrets for JWT signing key; outputs secret ARNs) with variables.tf, main.tf, outputs.tf
 - [x] T206 [P] Create infra/aws/modules/alb/ module (Application Load Balancer, HTTPS listener, target group, security group) with variables.tf, main.tf, outputs.tf
 - [x] T207 Create infra/aws/modules/ecs/ module (ECS Fargate cluster, task definition, service, Task Execution Role with secretsmanager:GetSecretValue, Task Role with DynamoDB access via task_role_extra_policy_arns variable) with variables.tf, main.tf, outputs.tf (depends on T201-T206 for input/output variable alignment)

### Prod Environment Terragrunt Configuration

 - [ ] T208 [P] Create infra/aws/accounts/prod/account.hcl with prod AWS account ID and account name locals
 - [x] T209 [P] Create infra/aws/accounts/prod/eu-west-1/region.hcl with region locals (aws_region = "eu-west-1")
 - [x] T210 [P] Create infra/aws/accounts/prod/eu-west-1/vpc/terragrunt.hcl with CIDR and AZ inputs, source = modules/vpc
 - [x] T211 [P] Create infra/aws/accounts/prod/eu-west-1/ecr/terragrunt.hcl with source = modules/ecr
 - [x] T213 [P] Create infra/aws/accounts/prod/eu-west-1/dynamodb/terragrunt.hcl with table name, GSI, and TTL attr `AB` inputs
 - [x] T214 [P] Create infra/aws/accounts/prod/eu-west-1/secrets-manager/terragrunt.hcl with secret name inputs (JWT key)
 - [ ] T215 Create infra/aws/accounts/prod/eu-west-1/ecs/terragrunt.hcl with dependencies on vpc, ecr, dynamodb, secrets-manager, and environment-specific inputs (depends on T210-T211, T213-T214)

**Checkpoint**: Prod environment infrastructure definition complete. `terragrunt run-all apply` from accounts/prod/eu-west-1/ provisions the full stack in dependency order (vpc → ecr/dynamodb/secrets-manager → ecs). Dev and staging directory structures are kept but not deployed for now.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

 - [ ] T001 Create NestJS project structure at repository root with src/, test/, ci.sh, and cd.sh
- [ ] T002 Initialize Node.js project with package.json and install NestJS dependencies (@nestjs/core, @nestjs/common, @nestjs/platform-express, @nestjs/platform-socket.io, @nestjs/websockets)
- [ ] T003 [P] Configure TypeScript with tsconfig.json for Node.js >= 18 target
- [ ] T004 [P] Configure ESLint and Prettier for code quality
- [ ] T005 [P] Setup Jest testing framework in test/ directory
 - [ ] T006 [P] Setup Docker configuration with Dockerfile, Dockerfile.builder, and docker-compose.yml at repository root
- [ ] T007 [P] Create .gitignore file for Node.js project

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

 - [ ] T008 Create src/config/ directory and src/AppModule.ts, src/Main.ts entry point files
 - [ ] T009 [P] Setup configuration module in src/config/ConfigModule.ts for environment variables
 - [ ] T010 [P] Create environment configuration service in src/config/ConfigService.ts
 - [ ] T011 [P] Implement structured logging with correlation IDs in src/config/LoggerService.ts
 - [ ] T012 [P] Create src/adapters/primary/rest/ directory and HealthController.ts health check endpoint
 - [ ] T013 [P] Setup OpenAPI/Swagger documentation configuration in src/Main.ts
 - [ ] T014 [P] Create global exception filter in src/config/GlobalExceptionFilter.ts
 - [ ] T015 [P] Setup graceful shutdown handler in src/Main.ts
 - [ ] T016 Wire ConfigModule, LoggerService, GlobalExceptionFilter, and HealthController into src/AppModule.ts

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: Core User Domain (Shared Prerequisites)

**Purpose**: Establish the shared user model and services required by authentication and user-facing APIs.

### Shared User Domain Entities

 - [ ] T017 [P] [Shared] Create src/core/entities/ directory and UserEntity.ts with userId, userHandle, accountState, displayName, email, emailVerifiedAt, createdAt, updatedAt, lastActiveAt, promotedAt
 - [ ] T067 [P] [Shared] Create UserLearningProfileEntity.ts in src/core/entities/ with userId, learningLevel (CEFR A1–C2), nativeLanguage, learningGoals, correctionStyle, topicsOfInterest
 - [ ] T068 [P] [Shared] Create UserUsageStatsEntity.ts in src/core/entities/ with userId, totalConversationCount, totalPracticeMinutes, currentStreak, longestStreak
 - [ ] T069 [P] [Shared] Create UserPreferencesEntity.ts in src/core/entities/ with userId, timezone, uiLanguage

### Shared User Ports and Services

 - [ ] T073 [P] [Shared] Create src/core/ports/secondary/ directory and IUserRepository.ts (create, findById, update, findByEmail, findByUserHandle)
 - [ ] T074 [P] [Shared] Create src/core/ports/primary/ directory and IUserService.ts
 - [ ] T075 [Shared] Create src/adapters/secondary/dynamodb/ directory and implement DynamoDbUserRepository.ts with `pk = A#{userId}`, `sk = A`, sequential attrs (`A=userId`, `B=userHandle`, `C=accountState`, `D=displayName`, `E=email`, `F=emailVerifiedAt`, `G=createdAt`, `H=updatedAt`, `I=lastActiveAt`, `J=promotedAt`, `K=learningProfile`, `L=usageStats`, `M=preferences`), transactional lookup items at `pk = C#{normalizedEmail}`, `sk = A` for unique email, and a `B`-backed GSI for WebAuthn handle resolution (depends on T017, T073)
 - [ ] T076 [Shared] Create src/core/services/ directory and implement UserService.ts: create or reuse a shadow user with default nested `learningProfile`, `usageStats`, and `preferences` on the core singleton (`sk = A`) (used by AuthService), get full user profile, update displayName, update learning profile, update preferences (depends on T073, T074)

**Checkpoint**: Shared user domain is ready. Authentication and profile implementation can proceed.

---

## Phase 4: User Story 1 - Authentication: Magic Link & Passkey (Priority: P1)

**Goal**: Enable shadow-account onboarding via magic links, mandatory initial WebAuthn passkey registration, passkey recovery by email link, and fresh email verification required after logout.

**Independent Test**: A new user requests a magic link, gets a shadow account, verifies the email, registers a passkey, uses the app, logs out, and must verify the email again before a later login. A user who lost a passkey can request a recovery email and be taken directly into passkey registration.

### Domain Entities for User Story 1

 - [ ] T084 [P] [US1] Create MagicLinkTokenEntity.ts in src/core/entities/ (tokenId, userId, email, purpose, tokenHash, expiresAt, usedAt, createdAt)
 - [ ] T085 [P] [US1] Create UserPasskeyEntity.ts in src/core/entities/ (passkeyId, userId, credentialId, publicKey, counter, deviceName, createdAt, lastUsedAt)
 - [ ] T086 [P] [US1] Create RefreshTokenEntity.ts in src/core/entities/ (tokenId, userId, tokenHash, expiresAt, revokedAt, createdAt)

### Domain DTOs for User Story 1

 - [ ] T087 [P] [US1] Create src/core/dtos/ directory and MagicLinkRequestDto.ts with Zod schema (email)
 - [ ] T088 [P] [US1] Create VerifyMagicLinkDto.ts in src/core/dtos/ with Zod schema (token)
 - [ ] T089 [P] [US1] Create AuthResponseDto.ts in src/core/dtos/ (accessToken, refreshToken, userId, accountState, passkeyRegistrationRequired)
 - [ ] T090 [P] [US1] Create RefreshTokenRequestDto.ts in src/core/dtos/ with Zod schema (refreshToken)

### Port Interfaces for User Story 1

 - [ ] T091 [P] [US1] Create IMagicLinkTokenRepository.ts in src/core/ports/secondary/ (create, findByTokenHash, markUsed)
 - [ ] T092 [P] [US1] Create IUserPasskeyRepository.ts in src/core/ports/secondary/ (create, findByUserId, findByUserIdAndCredentialId, updateCounter)
 - [ ] T093 [P] [US1] Create IRefreshTokenRepository.ts in src/core/ports/secondary/ (create, findByTokenHash, revoke, revokeAllForUser)
 - [ ] T094 [P] [US1] Create IEmailService.ts in src/core/ports/secondary/ (sendMagicLinkEmail, sendPasskeyRecoveryEmail)
 - [ ] T095 [P] [US1] Create IAuthService.ts in src/core/ports/primary/ 

### DynamoDB Adapters for User Story 1

 - [ ] T102 [US1] Implement DynamoDbMagicLinkTokenRepository.ts in src/adapters/secondary/dynamodb/ using `pk = B#{tokenId}`, `sk = A` and sequential attrs (`Y=tokenId`, `A=userId`, `E=email`, `Z=purpose`, `AA=tokenHash`, `AB=expiresAt`, `AC=usedAt`, `G=createdAt`) (depends on T091)
 - [ ] T103 [US1] Implement DynamoDbUserPasskeyRepository.ts in src/adapters/secondary/dynamodb/ using `pk = A#{userId}`, `sk = B#{credentialId}` and sequential attrs (`AE=passkeyId`, `A=userId`, `AF=credentialId`, `AG=publicKey`, `AH=counter`, `AI=deviceName`, `G=createdAt`, `AJ=lastUsedAt`) (depends on T092)
 - [ ] T104 [US1] Implement DynamoDbRefreshTokenRepository.ts in src/adapters/secondary/dynamodb/ using `pk = A#{userId}`, `sk = C#{tokenId}`, sequential attrs (`Y=tokenId`, `A=userId`, `AA=tokenHash`, `AB=expiresAt`, `AD=revokedAt`, `G=createdAt`), and the shared `AA`-backed GSI for token verification (depends on T093)

### Email Adapter for User Story 1

 - [ ] T105 [US1] Create src/adapters/secondary/email/ directory and implement StubEmailService.ts behind IEmailService port (depends on T094). Logs sign-in and passkey-recovery links to console for local development.

### Services for User Story 1

 - [ ] T106 [US1] Implement AuthService in src/core/services/AuthService.ts: sign-in magic link request/verify, passkey recovery email request, purpose-aware email-link verification (create or reuse shadow users on first email submission, mark email verified on successful verify, set `passkeyRegistrationRequired` on recovery tokens), JWT issuance (access+refresh), token refresh, logout/revocation with fresh email verification required on next login, rate limiting (5/email/hour) (depends on T091-T095, T073, T076)
 - [ ] T107 [US1] Implement PasskeyService in src/core/services/PasskeyService.ts: registration options, register credential, authentication options, authenticate. First verified shadow session must complete passkey registration before onboarding is complete, verified recovery links must be allowed to enter replacement passkey registration without a current passkey assertion, and passkey auth must not bypass post-logout email re-verification. Use @simplewebauthn/server, store an opaque per-user `userHandle` on the core item (`sk = A`) as attr `B`, and resolve assertions via `findByUserHandle` before loading the passkey by `(userId, credentialId)` (depends on T092, T095)

### Auth Guard for User Story 1

 - [ ] T108 [US1] Implement JwtAuthGuard in src/config/JwtAuthGuard.ts to validate Bearer tokens on protected routes (depends on T106)

### Primary Adapters (REST Controllers) for User Story 1

 - [ ] T109 [US1] Create AuthController in src/adapters/primary/rest/AuthController.ts with endpoints: POST /api/v1/auth/magic-link, POST /api/v1/auth/verify-magic-link, POST /api/v1/auth/refresh, POST /api/v1/auth/logout (depends on T106)
 - [ ] T110 [US1] Create PasskeyController in src/adapters/primary/rest/PasskeyController.ts with endpoints: POST recovery-link, POST registration-options, POST register, POST authentication-options, POST authenticate (depends on T107)

### Integration & Validation for User Story 1

 - [ ] T111 [US1] Create AuthModule in src/AuthModule.ts and wire AuthService, PasskeyService, repositories, EmailService, controllers (depends on T102-T110)
 - [ ] T112 [US1] Register AuthModule in root AppModule.ts (depends on T111)
 - [ ] T114 [US1] Add OpenAPI documentation and security scheme decorators to auth endpoints (depends on T109, T110)
 - [ ] T115 [US1] Validate first-time magic link request creates a shadow User with defaults and verify reuses that existing user record (depends on T109)
 - [ ] T116 [US1] Validate 429 rate limiting on magic-link and passkey-recovery email requests (depends on T109, T110)
 - [ ] T117 [US1] Validate 401 on expired/used emailed auth token for both sign-in and recovery flows (depends on T109, T110)
 - [ ] T118 [US1] Add JWT_SECRET and WEBAUTHN_RP_ID, WEBAUTHN_RP_NAME, WEBAUTHN_ORIGIN to ConfigService (JWT_SECRET injected via AWS Secrets Manager in deployed environments, .env locally) (depends on T106, T107)
 - [ ] T119 [US1] Validate explicit logout requires fresh magic link verification before a later login, even if passkeys exist (depends on T109, T110)
 - [ ] T120 [US1] Validate `POST /api/v1/auth/passkeys/recovery-link` sends a recovery email link for an existing user without exposing account existence (depends on T110)
 - [ ] T122 [US1] Validate verifying a recovery token sets `passkeyRegistrationRequired = true` and sends the client directly into replacement passkey registration even if other passkeys already exist (depends on T109, T110)

**Checkpoint**: User Story 1 complete. Shadow users can sign up via magic link, must register a passkey during onboarding, can recover lost passkeys via email links, can obtain JWTs for protected APIs, and must re-verify email after explicit logout.

---

## Phase 5: User Story 2 - User Profile Management API (Priority: P1)

**Goal**: Enable authenticated shadow or full users to retrieve their full profile and update learning preferences and app preferences via REST APIs. Shadow account creation is handled by the first magic link request (User Story 1).

**Independent Test**: An authenticated client can retrieve full user profile via GET /api/v1/users/:userId, update display name, learning profile, and preferences via PUT.

### Domain DTOs for User Story 2

 - [ ] T070 [P] [US2] Create UpdateUserDto.ts in src/core/dtos/ with Zod schema (displayName)
 - [ ] T071 [P] [US2] Create UpdateLearningProfileDto.ts in src/core/dtos/ with Zod schema (learningLevel, nativeLanguage, learningGoals, correctionStyle, topicsOfInterest)
 - [ ] T072 [P] [US2] Create UpdatePreferencesDto.ts in src/core/dtos/ with Zod schema (timezone, uiLanguage)

### Primary Adapters (REST Controllers) for User Story 2

 - [ ] T077 [US2] Create UserController.ts in src/adapters/primary/rest/ with endpoints: GET /api/v1/users/:userId (full profile with nested learningProfile, usageStats, preferences), PUT /api/v1/users/:userId, PUT /api/v1/users/:userId/learning-profile, PUT /api/v1/users/:userId/preferences (depends on T076)

### Integration & Validation for User Story 2

 - [ ] T078 [US2] Create UserModule.ts in src/ and wire UserService, UserRepository, UserController (depends on T075-T077)
 - [ ] T079 [US2] Register UserModule in root AppModule.ts (depends on T078)
 - [ ] T080 [US2] Add OpenAPI documentation decorators to UserController endpoints (depends on T077)
 - [ ] T081 [US2] Validate all GET/PUT endpoints return 404 for non-existent userId (depends on T077)
 - [ ] T113 [US2] Apply JwtAuthGuard to UserController (depends on T077, T108)

**Checkpoint**: User Story 2 complete. Users can view their full profile, configure learning preferences, and set app preferences through authenticated endpoints.

---

## Phase 6: User Story 3 - Interactive Conversation API (Priority: P1)

**Goal**: Enable client applications to initiate, conduct, and terminate interactive AI conversation sessions via WebSocket with real-time transcription and AI responses.

**Independent Test**: A client application can successfully create a conversation via POST /api/v1/conversations, connect to the WebSocket, stream German audio, receive real-time transcription updates, get AI text and audio responses, and gracefully terminate the session.

### Core Domain Entities for User Story 3

 - [ ] T018 [P] [US3] Create MessageEntity.ts in src/core/entities/
 - [ ] T019 [P] [US3] Create ConversationEntity.ts in src/core/entities/

### Domain DTOs for User Story 3

 - [ ] T020 [P] [US3] Create CreateConversationDto.ts in src/core/dtos/
 - [ ] T021 [P] [US3] Create ConversationResponseDto.ts in src/core/dtos/
 - [ ] T022 [P] [US3] Create TranscriptionUpdateDto.ts in src/core/dtos/
 - [ ] T023 [P] [US3] Create AIResponseDto.ts in src/core/dtos/

### Port Interfaces for User Story 3

 - [ ] T025 [P] [US3] Create IConversationRepository.ts in src/core/ports/secondary/
 - [ ] T028 [P] [US3] Create ILiveSession.ts in src/core/ports/secondary/

### Persistence Adapter for User Story 3

 - [ ] T029 [US3] Implement InMemoryConversationRepository.ts in src/adapters/secondary/inmemory/ to hold conversation state for the duration of the active session (depends on T019, T025)

### External Service Adapters for User Story 3

 - [ ] T030 [P] [US3] Create src/adapters/secondary/gemini-live/ directory and GeminiLiveModule.ts
 - [ ] T031 [US3] Implement GeminiLiveService.ts in src/adapters/secondary/gemini-live/ with German persona configuration (depends on T028, T030)

### Transcription Implementation for User Story 3

 - [ ] T057 [US3] Enable inputAudioTranscription and outputAudioTranscription in GeminiLiveService session setup (depends on T031)
 - [ ] T058 [US3] Parse inputTranscription messages from Gemini Live server responses in GeminiLiveService (depends on T057)
 - [ ] T059 [US3] Parse outputTranscription messages from Gemini Live server responses in GeminiLiveService (depends on T057)
 - [ ] T060 [US3] Add transcription callback types (InputTranscriptionCallback, OutputTranscriptionCallback) to AISessionEntity (depends on T058, T059)
 - [ ] T061 [US3] Implement addInputTranscriptionCallback/removeInputTranscriptionCallback in IAISessionRepository (depends on T060)
 - [ ] T062 [US3] Implement addOutputTranscriptionCallback/removeOutputTranscriptionCallback in IAISessionRepository (depends on T060)
 - [ ] T063 [US3] Emit input_transcription WebSocket events to clients in ConversationGateway (depends on T061)
 - [ ] T064 [US3] Emit output_transcription WebSocket events to clients in ConversationGateway (depends on T062)
 - [ ] T065 [US3] Persist user transcriptions to ConversationRepository as Message entities (depends on T063)
 - [ ] T066 [US3] Persist AI transcriptions to ConversationRepository as Message entities (depends on T064)

### Services for User Story 3

 - [ ] T032 [US3] Implement ConversationService.ts in src/core/services/ for conversation lifecycle management (depends on T019, T025, T028-T031)
 - [ ] T033 [US3] Create src/adapters/primary/websocket/ directory and implement ConversationGateway.ts with WebSocket audio streaming flow, `audio_chunk` handling, and `audio_stream_end` signaling (depends on T029, T031, T032)
 - [ ] T034 [US3] Implement Gemini Live response handling in GeminiLiveService.ts (depends on T028, T030)
 - [ ] T035 [US3] Implement ConversationEventDispatcher.ts in src/adapters/primary/websocket/ for conversation event dispatch integration (depends on T033, T034)

### Primary Adapters (REST Controllers) for User Story 3

- [ ] T036 [US3] Create ConversationController.ts in src/adapters/primary/rest/ (depends on T032)

### Session Management & Edge Cases for User Story 3

 - [ ] T037 [US3] Implement 5-minute session timeout mechanism in ConversationGateway.ts (depends on T033)

 - [ ] T039 [US3] Configure AI to enforce German/English only conversation in GeminiLiveService.ts (depends on T031)

### Integration & Validation for User Story 3

 - [ ] T040 [US3] Create src/ConversationModule.ts and src/ConversationEventDispatcherModule.ts and wire all services and adapters (depends on T032-T036)
 - [ ] T041 [US3] Register ConversationModule and ConversationEventDispatcherModule in root AppModule.ts (depends on T040)
 - [ ] T121 [US3] Apply JwtAuthGuard to ConversationController (depends on T036, T108)
 - [ ] T042 [US3] Add OpenAPI documentation for POST /api/v1/conversations endpoint (depends on T036)
 - [ ] T043 [US3] Validate conversation creation returns conversation_id and websocket_url per contract (depends on T036)
 - [ ] T044 [US3] Add error handling for missing or invalid API keys (depends on T032, T034, T031)
 - [ ] T045 [US3] Add request/response logging with correlation IDs for debugging (depends on T036)

**Checkpoint**: At this point, User Story 3 should be fully functional and testable independently. The backend can handle full conversation lifecycle: creation, real-time audio streaming, transcription, AI responses, and session termination.
