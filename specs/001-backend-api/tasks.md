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

 - [x] T200 Create infra/aws/ directory structure and root terragrunt.hcl with S3 remote state backend pattern (s3://ichbinkalt-tfstate-${account_name}/${region}/${path}/terraform.tfstate) and DynamoDB lock table

### Terraform Modules

 - [x] T201 [P] Create infra/aws/modules/vpc/ module (VPC, public/private subnets across AZs, route tables, IGW, NAT gateway) with variables.tf, main.tf, outputs.tf
 - [x] T202 [P] Create infra/aws/modules/ecr/ module (ECR repository with lifecycle policy) with variables.tf, main.tf, outputs.tf
 - [x] T204 [P] Create infra/aws/modules/dynamodb/ module (single DynamoDB table `ichbinkalt` with pk/sk string keys, PK markers `A#`, `B#`, `C#`, SK markers `A`, `B#`, `C#`, compact attribute aliases with `B=email`, GSIs on `N` and `B`+`D`, TTL on `R`, and support for user/email lookup-item patterns; outputs table ARN) with variables.tf, main.tf, outputs.tf
 - [x] T205 [P] Create infra/aws/modules/secrets-manager/ module (Secrets Manager secrets for JWT signing key; outputs secret ARNs) with variables.tf, main.tf, outputs.tf
 - [x] T206 [P] Create infra/aws/modules/alb/ module (Application Load Balancer, HTTPS listener, target group, security group) with variables.tf, main.tf, outputs.tf
 - [x] T207 Create infra/aws/modules/ecs/ module (ECS Fargate cluster, task definition, service, Task Execution Role with secretsmanager:GetSecretValue, Task Role with DynamoDB access via task_role_extra_policy_arns variable) with variables.tf, main.tf, outputs.tf (depends on T201-T206 for input/output variable alignment)

### Prod Environment Terragrunt Configuration

 - [ ] T208 [P] Create infra/aws/accounts/prod/account.hcl with prod AWS account ID and account name locals
 - [x] T209 [P] Create infra/aws/accounts/prod/eu-west-1/region.hcl with region locals (aws_region = "eu-west-1")
 - [x] T210 [P] Create infra/aws/accounts/prod/eu-west-1/vpc/terragrunt.hcl with CIDR and AZ inputs, source = modules/vpc
 - [x] T211 [P] Create infra/aws/accounts/prod/eu-west-1/ecr/terragrunt.hcl with source = modules/ecr
 - [x] T213 [P] Create infra/aws/accounts/prod/eu-west-1/dynamodb/terragrunt.hcl with table name, GSI, and TTL attr `R` inputs
 - [x] T214 [P] Create infra/aws/accounts/prod/eu-west-1/secrets-manager/terragrunt.hcl with secret name inputs (JWT key)
 - [x] T215 Create infra/aws/accounts/prod/eu-west-1/ecs/terragrunt.hcl with dependencies on vpc, ecr, dynamodb, secrets-manager, and environment-specific inputs (depends on T210-T211, T213-T214)

**Checkpoint**: Prod environment infrastructure definition complete. `terragrunt run-all apply` from accounts/prod/eu-west-1/ provisions the full stack in dependency order (vpc → ecr/dynamodb/secrets-manager → ecs). Dev and staging directory structures are kept but not deployed for now.

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

 - [x] T001 Create NestJS project structure at repository root with src/, test/, ci.sh, and cd.sh
 - [x] T002 Initialize Node.js project with package.json and install NestJS dependencies (@nestjs/core, @nestjs/common, @nestjs/platform-express, @nestjs/platform-socket.io, @nestjs/websockets)
 - [x] T003 [P] Configure TypeScript with tsconfig.json for Node.js >= 18 target
 - [x] T004 [P] Configure ESLint and Prettier for code quality
 - [x] T005 [P] Setup Jest testing framework in test/ directory
 - [x] T006 [P] Setup Docker configuration with Dockerfile, Dockerfile.builder, and docker-compose.yml at repository root
 - [x] T007 [P] Create .gitignore file for Node.js project

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

 - [x] T008 Create src/config/ directory and src/AppModule.ts, src/Main.ts entry point files
 - [x] T009 [P] Setup configuration module in src/config/ConfigModule.ts for environment variables
 - [x] T010 [P] Create environment configuration service in src/config/ConfigService.ts
 - [x] T011 [P] Implement structured logging with correlation IDs in src/config/LoggerService.ts
 - [x] T012 [P] Create src/adapters/primary/rest/ directory and HealthController.ts health check endpoint
 - [x] T013 [P] Setup OpenAPI/Swagger documentation configuration in src/Main.ts
 - [x] T014 [P] Create global exception filter in src/config/GlobalExceptionFilter.ts
 - [x] T015 [P] Setup graceful shutdown handler in src/Main.ts
 - [x] T016 Wire ConfigModule, LoggerService, GlobalExceptionFilter, and HealthController into src/AppModule.ts

**Checkpoint**: Foundation ready - user story implementation can now begin in parallel

---

## Phase 3: Core User Domain (Shared Prerequisites)

**Purpose**: Establish the shared user model and services required by authentication and user-facing APIs.

### Shared User Domain Entities

 - [x] T017 [P] [Shared] Create src/core/entities/ directory and UserEntity.ts with userId, emailConfirmed, displayName, email, createdAt, updatedAt, lastActiveAt
 - [x] T067 [P] [Shared] Create UserLearningProfileEntity.ts in src/core/entities/ with userId, learningLevel (CEFR A1–C2), nativeLanguage, learningGoals, correctionStyle, topicsOfInterest
 - [x] T068 [P] [Shared] Create UserUsageStatsEntity.ts in src/core/entities/ with userId, totalConversationCount, totalPracticeMinutes, currentStreak, longestStreak
 - [x] T069 [P] [Shared] Create UserPreferencesEntity.ts in src/core/entities/ with userId, timezone, uiLanguage

### Shared User Ports and Services

 - [x] T073 [P] [Shared] Create src/core/ports/secondary/ directory and IUserRepository.ts (create, findById, update, findByEmail)
 - [x] T074 [P] [Shared] Create src/core/ports/primary/ directory and IUserService.ts
 - [ ] T075 [Shared] Create src/adapters/secondary/dynamodb/ directory and implement DynamoDbUserRepository.ts with a minimal `UserCore` item at `pk = A#{userId}`, `sk = A` (`userId`, `email`, `displayName`, `createdAt`, `updatedAt`, `lastActiveAt`, `emailConfirmed` only), separate singleton child items for `UserLearningProfile` (`sk = D`), `UserUsageStats` (`sk = E`), and `UserPreferences` (`sk = F`), and transactional lookup items at `pk = C#{normalizedEmail}`, `sk = A` to enforce exactly one user core per normalized email (depends on T017, T073)
 - [ ] T076 [Shared] Create src/core/services/ directory and implement UserService.ts: create or reuse a pre-onboarding `UserCore`, initialize default `UserLearningProfile`, `UserUsageStats`, and `UserPreferences` singleton child items under dedicated sort keys rather than nesting them on `sk = A`, get full user profile, update displayName, update learning profile, update preferences (depends on T073, T074)

**Checkpoint**: Shared user domain is ready. Authentication and profile implementation can proceed.

---

## Phase 4: User Story 1 - Authentication: Passkey-First Registration, Email Verification, and Recovery (Priority: P1)

**Goal**: Enable immediate WebAuthn passkey registration as soon as the user enters an email, separate later email verification for subscription enablement, passkey recovery and account reclaim by email link, and passkey-first returning login.

**Independent Test**: A new user enters an email, receives WebAuthn registration options immediately, registers a passkey, receives a full session, uses the app, later verifies the email through a dedicated email-verification flow, and then becomes eligible for subscription-based service. A user who lost a passkey, or needs to reclaim control of an email-bound account, can request a recovery email, verify the recovery link, receive a recovery bootstrap token, and be taken directly into replacement passkey registration.

### Domain Entities for User Story 1

 - [x] T084 [P] [US1] Create MagicLinkTokenEntity.ts in src/core/entities/ (tokenId, userId, email, purpose=`email_verification|passkey_recovery`, tokenHash, expiresAt, usedAt, createdAt)
 - [x] T085 [P] [US1] Create UserPasskeyEntity.ts in src/core/entities/ (passkeyId, userId, credentialId, publicKey, counter, deviceName, createdAt, lastUsedAt)
 - [x] T086 [P] [US1] Create RefreshTokenEntity.ts in src/core/entities/ (tokenId, userId, tokenHash, expiresAt, revokedAt, createdAt)

### Domain DTOs for User Story 1

 - [x] T087 [P] [US1] Create src/core/dtos/ directory and MagicLinkRequestDto.ts with Zod schema (email) for email-verification and recovery link requests
 - [x] T088 [P] [US1] Create VerifyMagicLinkDto.ts in src/core/dtos/ with Zod schema (token) for email-verification and recovery-token verification
 - [x] T089 [P] [US1] Create AuthResponseDto.ts, EmailVerificationResponseDto.ts, and BootstrapAuthResponseDto.ts in src/core/dtos/ (`AuthResponse`: accessToken, refreshToken, userId, accountStatus, onboardingComplete, subscriptionStatus; `EmailVerificationResponse`: userId, emailConfirmed; `BootstrapAuthResponse`: bootstrapToken, userId, purpose, passkeyRegistrationRequired)
 - [x] T090 [P] [US1] Create RefreshTokenRequestDto.ts in src/core/dtos/ with Zod schema (refreshToken)

### Port Interfaces for User Story 1

 - [x] T091 [P] [US1] Create IMagicLinkTokenRepository.ts in src/core/ports/secondary/ (create, findByTokenHash, markUsed)
 - [x] T092 [P] [US1] Create IUserPasskeyRepository.ts in src/core/ports/secondary/ (create, findByUserId, findByUserIdAndCredentialId, updateCounter)
 - [x] T093 [P] [US1] Create IRefreshTokenRepository.ts in src/core/ports/secondary/ (create, findByTokenHash, revoke, revokeAllForUser)
 - [x] T094 [P] [US1] Create IEmailService.ts in src/core/ports/secondary/ (sendEmailVerificationEmail, sendPasskeyRecoveryEmail)
 - [x] T095 [P] [US1] Create IAuthService.ts in src/core/ports/primary/ 

### DynamoDB Adapters for User Story 1

 - [ ] T102 [US1] Implement DynamoDbMagicLinkTokenRepository.ts in src/adapters/secondary/dynamodb/ using `pk = B#{tokenId}`, `sk = A` and compact attrs (`M=tokenId`, `A=userId`, `B=email`, `T=purpose`, `N=tokenHash`, `O=expiresAt`, `P=usedAt`, `D=createdAt`, `R=ttl`) (depends on T091)
 - [ ] T103 [US1] Implement DynamoDbUserPasskeyRepository.ts in src/adapters/secondary/dynamodb/ using `pk = A#{userId}`, `sk = B#{credentialId}` and compact attrs (`G=passkeyId`, `A=userId`, `H=credentialId`, `I=publicKey`, `J=counter`, `K=deviceName`, `D=createdAt`, `L=lastUsedAt`) (depends on T092)
 - [ ] T104 [US1] Implement DynamoDbRefreshTokenRepository.ts in src/adapters/secondary/dynamodb/ using `pk = A#{userId}`, `sk = C#{tokenId}`, compact attrs (`M=tokenId`, `A=userId`, `N=tokenHash`, `O=expiresAt`, `Q=revokedAt`, `D=createdAt`, `R=ttl`), and the shared `N`-backed GSI for token verification (depends on T093)

### Email Adapter for User Story 1

 - [x] T105 [US1] Create src/adapters/secondary/email/ directory and implement StubEmailService.ts behind IEmailService port (depends on T094). Logs email-verification and passkey-recovery links to console for local development.

### Services for User Story 1

 - [x] T106 [US1] Implement AuthService in src/core/services/AuthService.ts: request email-verification link, verify email token, request passkey recovery email, verify recovery token, handle recovery-based account reclaim by revoking prior passkeys/sessions and resetting temporary learner data when necessary, token refresh, logout/revocation, rate limiting (5/email/hour), and auth-response snapshots for `accountStatus`, `onboardingComplete`, and `subscriptionStatus` (depends on T091-T095, T073, T076)
 - [x] T107 [US1] Implement PasskeyService in src/core/services/PasskeyService.ts: initial registration options by email, initial passkey registration, recovery re-registration options, recovery re-registration, authentication options, and authenticate. Initial registration should create or reuse the unique email-bound user immediately, recovery bootstrap sessions must allow replacement passkey registration without a current passkey assertion, and returning login must use passkey authentication directly. Use @simplewebauthn/server and pass the ULID `userId` directly as the WebAuthn `userID`, then load passkeys by `(userId, credentialId)` (depends on T092, T095)

### Auth Guard for User Story 1

 - [x] T108 [US1] Implement JwtAuthGuard and BootstrapAuthGuard in src/config/ to validate Bearer tokens on protected routes and recovery bootstrap tokens on passkey re-registration routes. The JWT guard should enforce current `accountStatus` and later entitlement policy from server-side user data rather than trusting mutable business claims from the token alone (depends on T106)

### Primary Adapters (REST Controllers) for User Story 1

 - [x] T109 [US1] Create AuthController in src/adapters/primary/rest/AuthController.ts with endpoints: POST /api/v1/auth/email-verification-link, POST /api/v1/auth/verify-email, POST /api/v1/auth/refresh, POST /api/v1/auth/logout (depends on T106)
 - [x] T110 [US1] Create PasskeyController in src/adapters/primary/rest/PasskeyController.ts with endpoints: POST registration-options, POST register, POST recovery-link, POST verify-recovery, POST re-registration-options, POST re-register, POST authentication-options, POST authenticate (depends on T107)

### Integration & Validation for User Story 1

 - [x] T111 [US1] Create AuthModule in src/AuthModule.ts and wire AuthService, PasskeyService, repositories, EmailService, controllers (depends on T102-T110)
 - [x] T112 [US1] Register AuthModule in root AppModule.ts (depends on T111)
 - [x] T114 [US1] Add OpenAPI documentation and security scheme decorators to auth endpoints, including the recovery-bootstrap protected passkey re-registration routes (depends on T109, T110)
 - [x] T115 [US1] Validate first-time registration-options requests create a unique email-bound user core plus default profile child items, later requests for the same normalized email reuse that same existing user record, and registered emails are not allowed to create a second user (depends on T110)
 - [x] T116 [US1] Validate 429 rate limiting on email-verification and passkey-recovery email requests (depends on T109, T110)
 - [x] T117 [US1] Validate 401 on expired or used emailed auth tokens for both email-verification and recovery flows (depends on T109, T110)
 - [x] T118 [US1] Add JWT_SECRET and WEBAUTHN_RP_ID, WEBAUTHN_RP_NAME, WEBAUTHN_ORIGIN to ConfigService (JWT_SECRET injected via AWS Secrets Manager in deployed environments, .env locally) (depends on T106, T107)
 - [x] T119 [US1] Validate explicit logout revokes the current session and a later login proceeds through passkey authentication unless passkey recovery is required (depends on T109, T110)
 - [x] T120 [US1] Validate `POST /api/v1/auth/email-verification-link` sends a verification email for the signed-in user and that `POST /api/v1/auth/verify-email` marks `emailConfirmed = true` for that same account (depends on T109)
 - [x] T122 [US1] Validate verifying a recovery token returns a bootstrap token, sets `passkeyRegistrationRequired = true`, and sends the client directly into replacement passkey registration even if other passkeys already exist (depends on T110)
- [x] T123 [US1] Validate recovery-based account reclaim revokes prior passkeys and refresh tokens, resets temporary learner data to defaults, and returns a bootstrap token for fresh passkey re-registration (depends on T106, T110)
 - [x] T124 [US1] Validate subscription-based service or upgrade paths are rejected until `emailConfirmed = true`, even after successful initial passkey registration (depends on T106, T109, T110)

**Checkpoint**: User Story 1 complete. Users can sign up by entering an email and immediately registering a passkey, verify the email later to unlock subscription-based service, recover or reclaim accounts via email links, and use passkey authentication for returning login.

---

## Phase 5: User Story 2 - User Profile Management API (Priority: P1)

**Goal**: Enable authenticated active users to retrieve their full profile and update learning preferences and app preferences via REST APIs. Initial user creation is handled by the first passkey-registration-options request (User Story 1), while profile responses surface current lifecycle and subscription snapshots.

**Independent Test**: An authenticated client can retrieve full user profile via GET /api/v1/users/:userId, update display name, learning profile, and preferences via PUT.

### Domain DTOs for User Story 2

 - [ ] T070 [P] [US2] Create UpdateUserDto.ts in src/core/dtos/ with Zod schema (displayName)
 - [ ] T071 [P] [US2] Create UpdateLearningProfileDto.ts in src/core/dtos/ with Zod schema (learningLevel, nativeLanguage, learningGoals, correctionStyle, topicsOfInterest)
 - [ ] T072 [P] [US2] Create UpdatePreferencesDto.ts in src/core/dtos/ with Zod schema (timezone, uiLanguage)

### Primary Adapters (REST Controllers) for User Story 2

 - [ ] T077 [US2] Create UserController.ts in src/adapters/primary/rest/ with endpoints: GET /api/v1/users/:userId (full profile assembled from `UserCore`, `UserLearningProfile`, `UserUsageStats`, and `UserPreferences`), PUT /api/v1/users/:userId, PUT /api/v1/users/:userId/learning-profile, PUT /api/v1/users/:userId/preferences (depends on T076)

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

 - [x] T018 [P] [US3] Create MessageEntity.ts in src/core/entities/
 - [x] T019 [P] [US3] Create ConversationEntity.ts in src/core/entities/

### Domain DTOs for User Story 3

 - [x] T020 [P] [US3] Create CreateConversationDto.ts in src/core/dtos/
 - [x] T021 [P] [US3] Create ConversationResponseDto.ts in src/core/dtos/
 - [ ] T022 [P] [US3] Create TranscriptionUpdateDto.ts in src/core/dtos/
 - [ ] T023 [P] [US3] Create AIResponseDto.ts in src/core/dtos/

### Port Interfaces for User Story 3

 - [x] T025 [P] [US3] Create IConversationRepository.ts in src/core/ports/secondary/
 - [x] T028 [P] [US3] Create ILiveSession.ts in src/core/ports/secondary/

### Persistence Adapter for User Story 3

 - [x] T029 [US3] Implement InMemoryConversationRepository.ts in src/adapters/secondary/inmemory/ to hold conversation state for the duration of the active session (depends on T019, T025)

### External Service Adapters for User Story 3

 - [ ] T030 [P] [US3] Create src/adapters/secondary/gemini-live/ directory and GeminiLiveModule.ts
 - [ ] T031 [US3] Implement GeminiLiveService.ts in src/adapters/secondary/gemini-live/ with German persona configuration (depends on T028, T030)

### Transcription Implementation for User Story 3

 - [x] T057 [US3] Enable inputAudioTranscription and outputAudioTranscription in GeminiLiveService session setup (depends on T031)
 - [x] T058 [US3] Parse inputTranscription messages from Gemini Live server responses in GeminiLiveService (depends on T057)
 - [x] T059 [US3] Parse outputTranscription messages from Gemini Live server responses in GeminiLiveService (depends on T057)
 - [ ] T060 [US3] Add transcription callback types (InputTranscriptionCallback, OutputTranscriptionCallback) to AISessionEntity (depends on T058, T059)
 - [ ] T061 [US3] Implement addInputTranscriptionCallback/removeInputTranscriptionCallback in IAISessionRepository (depends on T060)
 - [ ] T062 [US3] Implement addOutputTranscriptionCallback/removeOutputTranscriptionCallback in IAISessionRepository (depends on T060)
 - [ ] T063 [US3] Emit input_transcription WebSocket events to clients in ConversationGateway (depends on T061)
 - [ ] T064 [US3] Emit output_transcription WebSocket events to clients in ConversationGateway (depends on T062)
 - [x] T065 [US3] Persist user transcriptions to ConversationRepository as Message entities (depends on T063)
 - [x] T066 [US3] Persist AI transcriptions to ConversationRepository as Message entities (depends on T064)

### Services for User Story 3

 - [x] T032 [US3] Implement ConversationService.ts in src/core/services/ for conversation lifecycle management (depends on T019, T025, T028-T031)
 - [ ] T033 [US3] Create src/adapters/primary/websocket/ directory and implement ConversationGateway.ts with WebSocket audio streaming flow, `audio_chunk` handling, and `audio_stream_end` signaling (depends on T029, T031, T032)
 - [ ] T034 [US3] Implement Gemini Live response handling in GeminiLiveService.ts (depends on T028, T030)
 - [ ] T035 [US3] Implement ConversationEventDispatcher.ts in src/adapters/primary/websocket/ for conversation event dispatch integration (depends on T033, T034)

### Primary Adapters (REST Controllers) for User Story 3

- [x] T036 [US3] Create ConversationController.ts in src/adapters/primary/rest/ (depends on T032)

### Session Management & Edge Cases for User Story 3

 - [ ] T037 [US3] Implement 5-minute session timeout mechanism in ConversationGateway.ts (depends on T033)

 - [ ] T039 [US3] Configure AI to enforce German/English only conversation in GeminiLiveService.ts (depends on T031)

### Integration & Validation for User Story 3

 - [ ] T040 [US3] Create src/ConversationModule.ts and src/ConversationEventDispatcherModule.ts and wire all services and adapters (depends on T032-T036)
 - [ ] T041 [US3] Register ConversationModule and ConversationEventDispatcherModule in root AppModule.ts (depends on T040)
 - [ ] T121 [US3] Apply JwtAuthGuard to ConversationController (depends on T036, T108)
 - [x] T042 [US3] Add OpenAPI documentation for POST /api/v1/conversations endpoint (depends on T036)
 - [x] T043 [US3] Validate conversation creation returns conversation_id and websocket_url per contract (depends on T036)
 - [ ] T044 [US3] Add error handling for missing or invalid API keys (depends on T032, T034, T031)
 - [ ] T045 [US3] Add request/response logging with correlation IDs for debugging (depends on T036)

**Checkpoint**: At this point, User Story 3 should be fully functional and testable independently. The backend can handle full conversation lifecycle: creation, real-time audio streaming, transcription, AI responses, and session termination.
