# Feature Specification: AI Conversation API for German Learning App

**Feature Branch**: `001-backend-api`
**Created**: 2025-11-09
**Status**: In Progress
**Input**: User description: "I'm a technical product manager now, and would want to build a backend for an application for learning German. The backend will be providing APIs for users to have interactive conversation with the AI. User should be able to read the transcription in realtime. Let's just start from here as I'll add more specs so we can incrementally build the application together"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Authentication: Magic Link & Passkey (Priority: P1)

Users begin with email-based sign-up. The first magic-link request creates or reuses a shadow account tied to the email. After email verification, the user receives a session and must register a passkey during onboarding so the account is bound to a device. The shadow account can use the app until a later billing/paywall flow promotes it to a full account. If a user loses their passkey, they can request a recovery email link that takes them directly back into passkey registration after the email link is verified. After an explicit logout, the next login must start with email verification again. No passwords or OAuth.

**Why this priority**: Authentication is required before any user-facing feature can be accessed securely. All other user stories depend on knowing who the user is.

**Independent Test**: A new user can request a magic link, cause a shadow account to be created, verify the email, register a passkey, use the app, log out, and then be required to verify the email again before a later login. A user who loses a passkey can request a recovery email, click the recovery link, and be taken directly into replacement passkey registration.

**Acceptance Scenarios**:

1.  **Given** a first-time user enters an email, **When** the client sends a `POST` request to `/api/v1/auth/magic-link` with `email`, **Then** the backend creates a shadow `User` for that email with default `UserLearningProfile`, `UserUsageStats`, and `UserPreferences`, generates a single-use token (15 min expiry), sends an email with the magic link, and returns `200 OK`.
2.  **Given** an existing shadow or full user wants to log in, **When** the client sends a `POST` request to `/api/v1/auth/magic-link` with `email`, **Then** the backend reuses the existing user record, generates a single-use token (15 min expiry), sends an email with the magic link, and returns `200 OK`.
3.  **Given** a user clicks an emailed sign-in or recovery link, **When** the client sends a `POST` request to `/api/v1/auth/verify-magic-link` with the `token`, **Then** the backend validates the token (not expired, not used), marks it consumed, updates the user's `emailVerifiedAt`, and returns `200 OK` with `accessToken` (JWT, 15 min), `refreshToken` (7 day), `userId`, `accountState`, and `passkeyRegistrationRequired` (true if user has no passkeys or if the verified token was issued for passkey recovery).
4.  **Given** an authenticated user with `passkeyRegistrationRequired: true`, **When** the client sends a `POST` request to `/api/v1/auth/passkeys/registration-options`, **Then** the backend generates and returns WebAuthn registration challenge options, and the client proceeds directly into passkey registration.
5.  **Given** the client has completed the WebAuthn ceremony, **When** it sends a `POST` request to `/api/v1/auth/passkeys/register` with the credential, **Then** the backend verifies the credential, stores the new or replacement public key, and returns `201 Created`.
6.  **Given** a shadow user has completed first-passkey registration, **When** that user calls authenticated profile and conversation APIs before the paywall flow, **Then** the backend authorizes those requests against the same shadow account.
7.  **Given** a returning user with a registered passkey and an active email-verified authentication flow, **When** the client sends a `POST` request to `/api/v1/auth/passkeys/authentication-options` with `email`, **Then** the backend returns WebAuthn authentication challenge options.
8.  **Given** the client has signed the challenge during that active email-verified authentication flow, **When** it sends a `POST` request to `/api/v1/auth/passkeys/authenticate` with the signed response, **Then** the backend verifies the signature, updates the passkey `lastUsedAt` and counter, and returns `200 OK` with `accessToken`, `refreshToken`, `userId`, and `accountState`.
9.  **Given** an authenticated user, **When** the client sends a `POST` request to `/api/v1/auth/logout` with the `refreshToken`, **Then** the backend revokes the refresh token and returns `200 OK`, and any later login attempt for that user must begin with a new email verification via `/api/v1/auth/magic-link`.
10. **Given** a user requests a magic link, **When** more than 5 requests are made for the same email within 1 hour, **Then** the backend returns `429 Too Many Requests`.
11. **Given** a user has lost access to their passkey, **When** the client sends a `POST` request to `/api/v1/auth/passkeys/recovery-link` with `email`, **Then** the backend generates a single-use recovery token (15 min expiry), sends an email containing a recovery link to the client passkey-registration route, and returns `200 OK`.
12. **Given** a user clicks the recovery link from that email, **When** the client verifies the token via `POST /api/v1/auth/verify-magic-link`, **Then** the backend treats that as email verification for passkey recovery and returns a response that drives the client directly into passkey registration even if the user already has other passkeys stored.
13. **Given** a user requests passkey recovery, **When** more than 5 recovery-link requests are made for the same email within 1 hour, **Then** the backend returns `429 Too Many Requests`.

### User Story 2 - User Profile Management API (Priority: P1)

An authenticated shadow or full user can manage their profile, learning preferences, and view their usage statistics via REST API endpoints. Shadow account creation happens during the first magic link request (User Story 1), and the same user record remains in use after email verification and mandatory first passkey registration.

**Why this priority**: User profile and learning preferences drive AI personalization (correction style, topics, native language). Without these, conversations cannot be tailored to the learner.

**Independent Test**: An authenticated client can retrieve their full profile, update display name, set learning preferences and app preferences via dedicated endpoints.

**Acceptance Scenarios**:

1.  **Given** an authenticated user, **When** the client sends a `GET` request to `/api/v1/users/{userId}`, **Then** the backend returns a `200 OK` response with the full user object including nested `learningProfile`, `usageStats`, and `preferences`.
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
- **FR-011**: The system MUST provide a REST API endpoint to retrieve the full user profile (`GET /api/v1/users/{userId}`) including nested `learningProfile`, `usageStats`, and `preferences` in a single response.
- **FR-012**: The system MUST provide a REST API endpoint to update the user's core identity (`PUT /api/v1/users/{userId}`), specifically `displayName`.
- **FR-013**: The system MUST provide a REST API endpoint to update a user's learning profile (`PUT /api/v1/users/{userId}/learning-profile`), including `learningLevel`, `nativeLanguage`, `learningGoals`, `correctionStyle`, and `topicsOfInterest`.
- **FR-014**: The system MUST provide a REST API endpoint to update a user's preferences (`PUT /api/v1/users/{userId}/preferences`), including `timezone` and `uiLanguage`. Usage stats are read-only for the client and updated by the system after conversations.
- **FR-015**: When a first-time email is submitted to `/api/v1/auth/magic-link`, the system MUST create a shadow `User` with `accountState = shadow`, initialize default records for `UserLearningProfile` (correctionStyle: `moderate`), `UserUsageStats` (all zeros), and `UserPreferences` (timezone: `UTC`, uiLanguage: `en`), and default `displayName` to the email local part.
- **FR-016**: The system MUST support single-use emailed auth links. Each emailed token is hashed, expires after 15 minutes, and carries a `purpose` of either `sign_in` or `passkey_recovery`. Successful verification updates `emailVerifiedAt` on that same user record.
- **FR-017**: The system MUST support WebAuthn passkey registration and authentication. A user can register multiple passkeys (one-to-many). Each user has an opaque unique `userHandle` used during WebAuthn ceremonies. Passkey-based authentication returns JWT tokens when used in an allowed authentication flow.
- **FR-018**: The system MUST issue JWT access tokens (15 min expiry) and refresh tokens (7 day expiry). Refresh tokens are stored hashed and support rotation and revocation.
- **FR-019**: All authenticated endpoints (User Story 2 and 3) MUST be protected by a JWT auth guard that validates the access token from the `Authorization: Bearer <token>` header.
- **FR-020**: The system MUST rate-limit magic link and passkey recovery email requests to 5 per email per hour to prevent abuse.
- **FR-021**: The email delivery MUST be abstracted behind a port interface (`IEmailService`) so the provider can be swapped without changing business logic.
- **FR-022**: After explicit logout, the next login MUST begin with a new magic link email verification even if the user has one or more registered passkeys.
- **FR-023**: A first-time email-verified shadow account MUST complete passkey registration before onboarding is considered complete.
- **FR-024**: A shadow account that has completed email verification and mandatory first-passkey registration MUST be allowed to use authenticated user-profile and conversation APIs until a later billing/paywall flow promotes the same user record to `full`.
- **FR-025**: The system MUST provide a `POST /api/v1/auth/passkeys/recovery-link` endpoint that sends a single-use email recovery link to a user who has lost their passkey.
- **FR-026**: Verifying a token with `purpose = passkey_recovery` via `POST /api/v1/auth/verify-magic-link` MUST return a trusted session that sets `passkeyRegistrationRequired = true`, so the client can route directly into passkey registration even when the user already has stored passkeys.
- **FR-027**: Passkey recovery MUST allow a verified user to register a replacement passkey without presenting an existing passkey assertion first.

### Key Entities *(include if feature involves data)*

- **User**: Represents a learner and the shadow-account lifecycle state. Key attributes: `user_id`, `user_handle`, `account_state` (`shadow` or `full`), `display_name`, `email`, `email_verified_at`, `created_at`, `updated_at`, `last_active_at`, `promoted_at`.
- **UserLearningProfile**: One-to-one with User. Key attributes: `user_id`, `learning_level` (CEFR A1–C2), `native_language`, `learning_goals`, `correction_style`, `topics_of_interest`.
- **UserUsageStats**: One-to-one with User. Key attributes: `user_id`, `total_conversation_count`, `total_practice_minutes`, `current_streak`, `longest_streak`.
- **UserPreferences**: One-to-one with User. Key attributes: `user_id`, `timezone`, `ui_language`.
- **MagicLinkToken**: Single-use emailed auth token. Key attributes: `token_id`, `user_id`, `email`, `purpose` (`sign_in` or `passkey_recovery`), `token_hash`, `expires_at`, `used_at`, `created_at`.
- **UserPasskey**: WebAuthn credential (one-to-many with User). Key attributes: `passkey_id`, `user_id`, `credential_id`, `public_key`, `counter`, `device_name`, `created_at`, `last_used_at`.
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
