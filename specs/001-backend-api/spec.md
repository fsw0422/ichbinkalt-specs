# Feature Specification: AI Conversation API for German Learning App

**Feature Branch**: `001-backend-api`
**Created**: 2025-11-09
**Status**: In Progress
**Input**: User description: "I'm a technical product manager now, and would want to build a backend for an application for learning German. The backend will be providing APIs for users to have interactive conversation with the AI. User should be able to read the transcription in realtime. Let's just start from here as I'll add more specs so we can incrementally build the application together"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Authentication: Passkey-First Registration, Email Verification, and Recovery (Priority: P1)

Users register by entering an email address and are immediately prompted to create a passkey. A normalized email address maps to exactly one user record and remains the durable account identifier across registration, verification, recovery, and reclaim. The first passkey-registration request creates or reuses the single user record bound to that normalized email and returns WebAuthn registration options right away. After the first passkey is registered, the backend issues the normal app session even if the email has not yet been verified. Email verification is a separate follow-up flow used to confirm inbox control and unlock subscription-based services and upgrades. Before `emailConfirmed = true`, the product may allow only free or trial usage, and the inbox owner may still reclaim the email-bound account through the recovery flow. Returning users sign in with passkeys, not magic links. If a user loses their passkey, they can request a recovery email link, verify it, and be taken directly into replacement passkey registration. A successful recovery verification auto-confirms the email for that account. The client may surface that entrypoint with familiar copy such as `Forgot Password?`, but the backend remains passwordless and may use the same recovery flow to revoke prior passkeys and refresh tokens before handing the account to the verified inbox owner. No passwords or OAuth.

**Why this priority**: Authentication is required before any user-facing feature can be accessed securely. All other user stories depend on knowing who the user is.

**Independent Test**: A new user can enter an email, receive WebAuthn registration options immediately, register a first passkey, receive a full app session, later verify the email through a separate inbox-verification flow, and then become eligible for subscription-based service. A user who loses a passkey, or who needs to reclaim control of an email-bound account, can request a recovery email, click the recovery link, receive a recovery bootstrap token, and be taken directly into replacement passkey registration.

**Acceptance Scenarios**:

Initial sign-up is fully covered by scenarios 1-4 and does not send any email. Scenarios 5-7 are a separate later inbox-verification flow for a user who has already finished passkey registration and signed in.

1.  **Given** a first-time user enters an email whose normalized value is not already bound to an account with a registered passkey, **When** the client sends a `POST` request to `/api/v1/auth/passkeys/registration-options` with `email`, **Then** the backend creates or reuses the single `UserCore` for that normalized email, initializes default `UserLearningProfile`, `UserUsageStats`, and `UserPreferences` child items, and returns WebAuthn registration challenge options immediately.
2.  **Given** an existing user for that normalized email who still has no registered passkey, **When** the client sends a `POST` request to `/api/v1/auth/passkeys/registration-options` with the same email identity, even if casing differs, **Then** the backend reuses that same user record and returns registration options again instead of creating a second user.
3.  **Given** a normalized email is already bound to a user who has a registered passkey, **When** the client attempts the sign-up registration flow again with that email, **Then** the backend MUST NOT create a second user for the same normalized email and MUST direct the client toward passkey authentication or the recovery flow.
4.  **Given** the client has completed the WebAuthn ceremony for initial registration, **When** it sends a `POST` request to `/api/v1/auth/passkeys/register` with the `email` and credential, **Then** the backend verifies the credential, stores the first passkey, and returns `201 Created` with `accessToken` (JWT, 15 min), `refreshToken` (7 day), `userId`, `accountStatus`, `onboardingComplete = true`, and `subscriptionStatus`.
5.  **Given** a user has already completed initial passkey sign-up, is signed in, and still has `emailConfirmed = false`, **When** the client sends a `POST` request to `/api/v1/auth/email-verification-link` with `email`, **Then** the backend generates a single-use email-verification token (15 min expiry), sends an inbox-verification email, and returns `200 OK`.
6.  **Given** a user clicks an emailed verification link, **When** the client sends a `POST` request to `/api/v1/auth/verify-email` with the `token`, **Then** the backend validates the token (not expired, not used), marks it consumed, marks that user's `emailConfirmed = true`, and returns `200 OK`.
7.  **Given** a user has not yet completed email verification, **When** the client attempts to enable subscription-based service or move beyond the free or trial allowance, **Then** the backend rejects that upgrade path and requires successful email verification first.
8.  **Given** a returning user with a registered passkey, **When** the client sends a `POST` request to `/api/v1/auth/passkeys/authentication-options` with `email`, **Then** the backend returns WebAuthn authentication challenge options.
9.  **Given** the client has signed the challenge for that returning-user passkey flow, **When** it sends a `POST` request to `/api/v1/auth/passkeys/authenticate` with the signed response, **Then** the backend verifies the signature, updates the passkey `lastUsedAt` and counter, and returns `200 OK` with `accessToken`, `refreshToken`, `userId`, `accountStatus`, `onboardingComplete`, and `subscriptionStatus`.
10. **Given** an authenticated user, **When** the client sends a `POST` request to `/api/v1/auth/logout` with the `refreshToken`, **Then** the backend revokes the refresh token and returns `200 OK`, and any later login attempt for that user begins with passkey authentication unless the user needs passkey recovery.
11. **Given** a user has lost access to their passkey or needs to reclaim an email-bound account, **When** the client sends a `POST` request to `/api/v1/auth/passkeys/recovery-link` with `email`, **Then** the backend generates a single-use recovery token (15 min expiry), sends an email containing a recovery link to the client re-registration route, and returns `200 OK`.
12. **Given** a user clicks the recovery link from that email, **When** the client sends a `POST` request to `/api/v1/auth/passkeys/verify-recovery` with the `token`, **Then** the backend validates the token, marks the user's `emailConfirmed = true`, revokes existing passkeys and refresh tokens for that account, and returns `200 OK` with a scoped `bootstrapToken`, `userId`, `purpose = passkey_recovery`, and `passkeyRegistrationRequired = true`.
13. **Given** a user with a valid recovery bootstrap token, **When** the client sends a `POST` request to `/api/v1/auth/passkeys/re-registration-options`, **Then** the backend generates and returns WebAuthn registration challenge options for replacement passkey registration.
14. **Given** the client has completed the WebAuthn ceremony while using a valid recovery bootstrap token, **When** it sends a `POST` request to `/api/v1/auth/passkeys/re-register` with the credential, **Then** the backend verifies the credential, stores the replacement public key, invalidates the recovery bootstrap flow, and returns `201 Created` with `accessToken`, `refreshToken`, `userId`, `accountStatus`, `onboardingComplete`, and `subscriptionStatus`.
15. **Given** a user requests email verification or passkey recovery, **When** more than 5 verification-link or recovery-link requests are made for the same email within 1 hour, **Then** the backend returns `429 Too Many Requests`.

### User Story 2 - User Profile Management API (Priority: P1)

An authenticated user whose `accountStatus = active` can manage their profile, learning preferences, and view their usage statistics via REST API endpoints. The same user record remains in use after immediate first-passkey registration and any later email-verification step, and the API returns server-calculated snapshots such as `onboardingComplete` and `subscriptionStatus` for client routing and UI.

**Why this priority**: User profile and learning preferences drive AI personalization (correction style, topics, native language). Without these, conversations cannot be tailored to the learner.

**Independent Test**: An authenticated client can retrieve their full profile, update display name, set learning preferences and app preferences via dedicated endpoints.

**Acceptance Scenarios**:

1.  **Given** an authenticated user, **When** the client sends a `GET` request to `/api/v1/users/{userId}`, **Then** the backend returns a `200 OK` response with the full user object assembled from `UserCore`, `UserLearningProfile`, `UserUsageStats`, and `UserPreferences`.
2.  **Given** an authenticated user, **When** the client sends a `PUT` request to `/api/v1/users/{userId}` with updated `displayName`, **Then** the backend updates the user record and returns `200 OK` with the updated user.
3.  **Given** an authenticated user, **When** the client sends a `PUT` request to `/api/v1/users/{userId}/learning-profile` with `learningLevel`, `nativeLanguage`, and `correctionStyle`, **Then** the backend creates/updates the learning profile and returns `200 OK`.
4.  **Given** an authenticated user, **When** the client sends a `PUT` request to `/api/v1/users/{userId}/preferences` with `timezone` and `uiLanguage`, **Then** the backend creates/updates the preferences and returns `200 OK`.

### User Story 3 - Interactive Conversation API (Priority: P1)

The client application can initiate, conduct, and terminate an interactive audio-based conversation session with the AI backend via API endpoints.

**Why this priority**: This provides the core real-time conversation functionality required by the client application.

**Independent Test**: This can be tested by a client application successfully creating a conversation, streaming audio, sending an explicit audio stream end signal, receiving real-time transcription and AI text responses, and closing the session.

**Acceptance Scenarios**:

1.  **Given** the client application needs to start a new conversation, **When** it sends a `POST` request to the `/api/v1/conversations` endpoint, **Then** the backend returns a `201 Created` response containing a unique `conversation_id` and the WebSocket URL for real-time communication.
2.  **Given** an active WebSocket connection for a conversation, **When** the client application streams audio data via `audio_chunk` events, **Then** the backend accepts and buffers the audio for Gemini Live processing.
3.  **Given** the client has finished sending the current utterance, **When** it sends an `audio_stream_end` message, **Then** the backend signals to Gemini Live that the current audio stream has ended so the utterance can be finalized and a response can be generated.
4.  **Given** the backend is processing finalized audio input, **When** Gemini generates transcriptions for user input, **Then** the backend sends `input_transcription` messages over the WebSocket with the transcribed text and saves the transcription to the conversation history.
5.  **Given** the AI is generating an audio response, **When** Gemini produces audio output and its transcription, **Then** the backend sends `output_transcription` messages over the WebSocket with the AI's transcribed text and saves the transcription to the conversation history.
6.  **Given** the backend has received a complete user utterance and its `audio_stream_end` signal, **When** the AI generates a response, **Then** the backend sends `ai_audio_response` messages over the WebSocket containing the AI's audio response along with corresponding transcription updates.
7.  **Given** an active WebSocket connection, **When** the client sends a `terminate_session` message, **Then** the backend closes the WebSocket connection and terminates the AI service session, returning a `1000 Normal Closure` status.

### Edge Cases

- **WebSocket Disconnections**: If the WebSocket connection drops, the backend will maintain the session state for a 5-minute grace period. The client is responsible for attempting to reconnect to the same session. The user can also explicitly terminate the session via a client-side action.

- **Unsupported Language**: The AI will be instructed to respond only in German or English. If the user speaks another language, the AI will gently guide them back to using German or English.

### Authorization Model

- **Recovery Bootstrap Token**: Short-lived, narrow-scoped, and only valid for passkey re-registration endpoints after a verified recovery link. It carries the minimum claims needed for that flow: `sub`, a unique token ID such as `jti`, `tokenType = bootstrap`, `purpose = passkey_recovery`, `scope = passkey_registration`, and `exp`.
- **Access Token**: Issued after successful first-passkey registration, recovery re-registration, or passkey authentication. It carries identity and session facts such as `sub`, a session ID such as `jti`, `tokenType = access`, `authLevel = verified`, `iat`, and `exp`.
- **Lifecycle Status**: Hard authorization states such as `active` or `suspended` are represented server-side by `accountStatus` and are not trusted solely from JWT claims.
- **Onboarding State**: Onboarding is modeled separately from lifecycle status via derived `onboardingComplete`, which is based on whether the user currently has at least one registered passkey.
- **Product Access**: Product access is modeled separately from authentication via `subscriptionStatus` (`free`, `trial`, `paid`, `grace`, `past_due`). Later premium entitlements are checked against current server-side data rather than treated as static JWT truth.
- **Verification-Gated Paid Access**: Email verification is a separate prerequisite for subscription-based services and upgrades. Before inbox proof exists, the account may use only free or trial paths and MUST NOT unlock paid entitlements.
- **Client Snapshots**: Auth and profile responses may include `accountStatus`, `onboardingComplete`, and `subscriptionStatus` as client-facing snapshots, but authorization decisions continue to use current server-side data.

### Email Verification And Recovery

- **Passkey-First Registration**: Initial sign-up does not wait for email verification. The user creates a passkey immediately after entering their email and receives an app session once registration succeeds.
- **Separate Email Verification**: Email verification is a later inbox-confirmation flow that marks `emailConfirmed = true` and unlocks subscription-based services and upgrades for that same account.
- **Password Reset as Recovery**: Although the backend is passwordless, the client may surface the recovery entrypoint with familiar copy such as `Forgot Password?`. A successful recovery-email verification is treated as proof of inbox ownership.
- **Auto-confirmed Recovery**: Once a recovery link has been successfully verified, the corresponding user record is considered email-confirmed for that recovery flow. Replacement passkey registration reuses that confirmed state and does not require another inbox-confirmation round trip.
- **Recovery as Reclaim**: If the inbox owner uses the recovery flow to reclaim an email-bound account, the system revokes prior passkeys and sessions, may wipe temporary non-billing learner data, and hands control to the verified inbox owner through a fresh bootstrap token and passkey re-registration flow.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The system MUST provide an API endpoint to start and end a conversation session.
- **FR-002**: The system MUST accept streaming audio data from the client application over a WebSocket connection via `audio_chunk` events.
- **FR-003**: The system MUST use Google Gemini's Live API with automatic speech-to-text (STT) transcription to process audio input and generate conversational responses in German.
- **FR-004**: The system MUST send real-time transcription updates for both user input (`input_transcription`) and AI output (`output_transcription`) to the client via WebSocket as audio is being processed.
- **FR-005**: The system MUST provide the AI's audio response to the client via WebSocket. The AI audio is produced by Gemini Live's native audio model.
- **FR-006**: The API MUST manage the state of the conversation, including the history of messages. Sessions timeout after 5 minutes of inactivity. Conversation history is only kept for the active session and is discarded when the session ends.
- **FR-007**: The backend MUST provide a German language teacher persona to the AI and enforce a strict rule that the AI will only converse in German or English.
- **FR-008**: The system MUST persist transcriptions (both user and AI) to the conversation history as `Message` entities for future retrieval and display.
- **FR-009**: The system MUST enable both `inputAudioTranscription` and `outputAudioTranscription` when configuring the Gemini Live session to receive transcriptions synchronously with audio streaming.
- **FR-010**: The client MUST send an explicit `audio_stream_end` WebSocket message after the last `audio_chunk` of an utterance, and the backend MUST forward that signal to Gemini Live so the utterance is finalized for transcription and response generation.
- **FR-011**: The system MUST provide a REST API endpoint to retrieve the full user profile (`GET /api/v1/users/{userId}`) by assembling `UserCore`, `UserLearningProfile`, `UserUsageStats`, and `UserPreferences` into a single response.
- **FR-012**: The system MUST provide a REST API endpoint to update the user's core identity (`PUT /api/v1/users/{userId}`), specifically `displayName`.
- **FR-013**: The system MUST provide a REST API endpoint to update a user's learning profile (`PUT /api/v1/users/{userId}/learning-profile`), including `learningLevel`, `nativeLanguage`, `learningGoals`, `correctionStyle`, and `topicsOfInterest`.
- **FR-014**: The system MUST provide a REST API endpoint to update a user's preferences (`PUT /api/v1/users/{userId}/preferences`), including `timezone` and `uiLanguage`. Usage stats are read-only for the client and updated by the system after conversations.
- **FR-015**: When a first-time email is submitted to `/api/v1/auth/passkeys/registration-options`, the system MUST create or reuse the single `UserCore` bound to that normalized email, keep that core item minimal (`userId`, `email`, `displayName`, `createdAt`, `updatedAt`, `lastActiveAt`, `emailConfirmed`), initialize default records for `UserLearningProfile` (correctionStyle: `moderate`), `UserUsageStats` (all zeros), and `UserPreferences` (timezone: `UTC`, uiLanguage: `en`) as separate user-owned items, and default `displayName` to the email local part. If the same normalized email has not yet completed first-passkey registration, later registration attempts MUST reuse that same user record.
- **FR-016**: The system MUST support single-use emailed verification links. Each emailed token is hashed, expires after 15 minutes, and carries a `purpose` of either `email_verification` or `passkey_recovery`. Successful verification via the email-verification flow marks `emailConfirmed = true` on that same user record.
- **FR-017**: The system MUST support WebAuthn passkey registration and authentication. Initial registration begins immediately after email entry, and a user can later register replacement passkeys through the recovery flow. Each user uses the same stable `userId` as the WebAuthn user ID during ceremonies. Full application sessions are issued after successful first-passkey registration, recovery re-registration, or passkey authentication.
- **FR-018**: The system MUST issue JWT access tokens (15 min expiry) and refresh tokens (7 day expiry) after successful passkey registration or passkey authentication. Refresh tokens are stored hashed and support rotation and revocation.
- **FR-019**: All authenticated endpoints (User Story 2 and 3) MUST be protected by a JWT auth guard that validates the access token from the `Authorization: Bearer <token>` header.
- **FR-020**: The system MUST rate-limit email-verification and passkey-recovery email requests to 5 per email per hour to prevent abuse.
- **FR-021**: The email delivery MUST be abstracted behind a port interface (`IEmailService`) so the provider can be swapped without changing business logic.
- **FR-022**: After explicit logout, the current refresh token MUST be revoked. A later login for a user with a registered passkey MUST begin with passkey authentication, and a user who no longer has a usable passkey, or who needs to reclaim control of the email-bound account, MUST use the recovery flow.
- **FR-023**: A first-time user MUST be able to complete passkey registration and receive a full application session before email verification. `onboardingComplete = true` once at least one passkey exists for that user.
- **FR-024**: A user with `accountStatus = active`, `onboardingComplete = true`, and `subscriptionStatus = free` or `trial` MUST be allowed to use authenticated user-profile and conversation APIs until a later billing/paywall flow changes the same user record's product access. Subscription-based services still require `emailConfirmed = true`.
- **FR-025**: The system MUST provide a `POST /api/v1/auth/passkeys/recovery-link` endpoint that sends a single-use email recovery link to a user who has lost their passkey.
- **FR-026**: Verifying a token with `purpose = passkey_recovery` via `POST /api/v1/auth/passkeys/verify-recovery` MUST mark `emailConfirmed = true` on that user record, revoke existing passkeys and refresh tokens for that account, and return a scoped bootstrap session that sets `passkeyRegistrationRequired = true`, so the client can route directly into passkey re-registration.
- **FR-027**: Passkey recovery MUST allow a verified user to register a replacement passkey without presenting an existing passkey assertion first.
- **FR-038**: After successful recovery-email verification, replacement passkey registration MUST reuse that confirmed email state and MUST NOT require a second email-confirmation step before issuing the replacement full app session.
- **FR-028**: The system MUST model hard lifecycle authorization separately from onboarding and billing via `accountStatus` with, at minimum, `active` and `suspended` values.
- **FR-029**: The system MUST model onboarding progress separately from lifecycle status via derived `onboardingComplete` based on whether the user currently has at least one registered passkey, rather than through a broad `accountState` field.
- **FR-030**: The system MUST model product access separately from authentication via `subscriptionStatus` with, at minimum, `free`, `trial`, `paid`, `grace`, and `past_due` values.
- **FR-031**: Recovery bootstrap tokens MUST contain only the minimum claims required for passkey re-registration (`sub`, a unique token ID such as `jti`, `tokenType = bootstrap`, `purpose = passkey_recovery`, `scope = passkey_registration`, and `exp`) and MUST NOT authorize profile, conversation, billing, or refresh endpoints.
- **FR-032**: Access tokens MUST contain identity and session/auth facts only (`sub`, a session ID such as `jti`, `tokenType = access`, `authLevel = verified`, `iat`, and `exp`). Mutable authorization facts such as suspension or paid entitlements MUST be enforced against current server-side data and MUST NOT rely solely on JWT claims.
- **FR-033**: Auth and user-profile responses MUST return current `accountStatus`, `onboardingComplete`, and `subscriptionStatus` as server-calculated snapshots for client UX, while authorization continues to use current server-side checks.
- **FR-034**: The recovery flow MAY be surfaced in client UX with familiar copy such as `Forgot Password?`, but the backend MUST treat it as passwordless email-based account recovery rather than literal password reset.
- **FR-035**: Successful recovery-email verification MUST be treated as authoritative inbox verification for that email address and MUST be allowed to reclaim control of the corresponding account from a prior stale or incorrect passkey holder.
- **FR-036**: During account reclaim, the system MUST revoke existing passkeys and refresh tokens and reset temporary learner data to defaults before allowing fresh bootstrap-backed passkey re-registration.
- **FR-037**: The system MUST enforce exactly one durable `UserCore` per normalized email address, using trimmed lowercase normalization for lookup and uniqueness. Registration, email verification, recovery, account reclaim, and passkey-authentication lookup by email MUST always resolve to that same user record and MUST NOT create a second user for the same normalized email.
- **FR-039**: Before email verification, the product MAY expose only free or trial access. Subscription-based services, plan upgrades, and other paid entitlements MUST remain disabled until `emailConfirmed = true`.
- **FR-040**: If recovery is used to reclaim an unverified or stale email-bound account, the system MUST be able to wipe temporary pre-confirmation learner data before the verified recovery claimant completes passkey re-registration.
- **FR-041**: Email verification MUST be required before enabling subscription-based services, plan upgrades, or non-trial entitlements.

### Key Entities *(include if feature involves data)*

- **UserCore**: Represents the minimal learner identity row. Key attributes: `user_id` (ULID), `display_name`, `email` (unique by normalized trimmed lowercase value), `email_confirmed`, `created_at`, `updated_at`, `last_active_at`.
- **UserLearningProfile**: One-to-one with `UserCore` and stored as its own user-owned item. Key attributes: `user_id`, `learning_level` (CEFR A1–C2), `native_language`, `learning_goals`, `correction_style`, `topics_of_interest`.
- **UserUsageStats**: One-to-one with `UserCore` and stored as its own user-owned item. Key attributes: `user_id`, `total_conversation_count`, `total_practice_minutes`, `current_streak`, `longest_streak`.
- **UserPreferences**: One-to-one with `UserCore` and stored as its own user-owned item. Key attributes: `user_id`, `timezone`, `ui_language`.
- **MagicLinkToken**: Single-use emailed verification or recovery token. Key attributes: `token_id`, `user_id`, `email`, `purpose` (`email_verification` or `passkey_recovery`), `token_hash`, `expires_at`, `used_at`, `created_at`.
- **BootstrapSession**: Short-lived scoped token returned after verifying a recovery link. Key attributes: `token`, `token_type` (`bootstrap`), `token_id`, `user_id`, `purpose` (`passkey_recovery`), `scope` (`passkey_registration`), `expires_at`.
- **AccessSession**: Short-lived verified session token returned after successful first-passkey registration, recovery re-registration, or passkey authentication. Key attributes: `token`, `token_type` (`access`), `session_id`, `user_id`, `auth_level` (`verified`), `issued_at`, `expires_at`.
- **UserPasskey**: WebAuthn credential (one-to-many with User). Key attributes: `passkey_id`, `user_id`, `credential_id`, `public_key`, `counter`, `device_name`, `created_at`, `last_used_at`. `onboardingComplete` is derived from whether at least one passkey currently exists for the user.
- **RefreshToken**: JWT refresh token for session rotation/revocation. Key attributes: `token_id`, `user_id`, `token_hash`, `expires_at`, `revoked_at`, `created_at`.
- **Conversation**: Represents a single interactive session between a user and the AI. Key attributes: `conversation_id`, `user_id`, `start_time`, `end_time`.
- **Message**: Represents a single turn in a conversation. Key attributes: `message_id`, `conversation_id`, `sender` (user or AI), `text_content`, `audio_data` (optional), `timestamp`. Transcriptions from both user input audio and AI output audio are stored as Message entities.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 95% of user audio utterances should be transcribed with acceptable accuracy using Gemini's automatic STT.
- **SC-002**: Latency from the end of a user's speech to receiving the first transcription event must be less than 500ms.
- **SC-003**: The p95 latency for the AI's text response to be delivered to the client must be under 3 seconds after the user's turn ends.
- **SC-004**: The backend must support at least 100 concurrent active conversations while maintaining the defined performance success criteria.
- **SC-005**: The WebSocket connection must reliably maintain state for at least 5 minutes of user inactivity.
