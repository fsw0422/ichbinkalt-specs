# Feature Specification: AI Conversation API for German Learning App

**Feature Branch**: `001-backend-api`
**Created**: 2025-11-09
**Status**: Draft
**Input**: User description: "I'm a technical product manager now, and would want to build a backend for an application for learning German. The backend will be providing APIs for users to have interactive conversation with the AI. User should be able to read the transcription in realtime. Let's just start from here as I'll add more specs so we can incrementally build the application together"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Interactive Conversation API (Priority: P1)

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

### User Story 2 - User Profile Management API (Priority: P1)

An authenticated user can manage their profile, learning preferences, and view their usage statistics via REST API endpoints. User creation happens during magic link verification (User Story 3).

**Why this priority**: User profile and learning preferences drive AI personalization (correction style, topics, native language). Without these, conversations cannot be tailored to the learner.

**Independent Test**: An authenticated client can retrieve their full profile, update display name, set learning preferences and app preferences via dedicated endpoints.

**Acceptance Scenarios**:

1.  **Given** an authenticated user, **When** the client sends a `GET` request to `/api/v1/users/{userId}`, **Then** the backend returns a `200 OK` response with the full user object including nested `learningProfile`, `usageStats`, and `preferences`.
2.  **Given** an authenticated user, **When** the client sends a `PUT` request to `/api/v1/users/{userId}` with updated `displayName`, **Then** the backend updates the user record and returns `200 OK` with the updated user.
3.  **Given** an authenticated user, **When** the client sends a `PUT` request to `/api/v1/users/{userId}/learning-profile` with `learningLevel`, `nativeLanguage`, and `correctionStyle`, **Then** the backend creates/updates the learning profile and returns `200 OK`.
4.  **Given** an authenticated user, **When** the client sends a `PUT` request to `/api/v1/users/{userId}/preferences` with `timezone` and `uiLanguage`, **Then** the backend creates/updates the preferences and returns `200 OK`.

### User Story 3 - Authentication: Magic Link & Passkey (Priority: P1)

Users can sign up and log in using magic links sent to their email. After first sign-up, a passkey registration flow is triggered so subsequent logins can use WebAuthn passkeys. Magic link remains as a fallback. No passwords or OAuth.

**Why this priority**: Authentication is required before any user-facing feature can be accessed securely. All other user stories depend on knowing who the user is.

**Independent Test**: A new user can sign up via magic link, register a passkey, log out, and log back in using either passkey or magic link fallback.

**Acceptance Scenarios**:

1.  **Given** a user (new or existing) wants to log in, **When** the client sends a `POST` request to `/api/v1/auth/magic-link` with `email`, **Then** the backend generates a single-use token (15 min expiry), sends an email with the magic link, and returns `200 OK`.
2.  **Given** a user clicks the magic link, **When** the client sends a `POST` request to `/api/v1/auth/verify-magic-link` with the `token`, **Then** the backend validates the token (not expired, not used), marks it consumed, and returns `200 OK` with `accessToken` (JWT, 15 min), `refreshToken` (7 day), `userId`, and `passkeyRegistrationRequired` (true if user has no passkeys).
3.  **Given** a new user verifying a magic link for the first time, **When** the token is valid and no `User` record exists for that email, **Then** the backend creates a new `User` (with `displayName` defaulting to the email local part) along with default `UserLearningProfile`, `UserUsageStats`, and `UserPreferences`, and returns the response with `passkeyRegistrationRequired: true`.
4.  **Given** an authenticated user with `passkeyRegistrationRequired: true`, **When** the client sends a `POST` request to `/api/v1/auth/passkeys/registration-options`, **Then** the backend generates and returns WebAuthn registration challenge options.
5.  **Given** the client has completed the WebAuthn ceremony, **When** it sends a `POST` request to `/api/v1/auth/passkeys/register` with the credential, **Then** the backend verifies the credential and stores the public key, returning `201 Created`.
6.  **Given** a returning user with a registered passkey, **When** the client sends a `POST` request to `/api/v1/auth/passkeys/authentication-options` with `email`, **Then** the backend returns WebAuthn authentication challenge options.
7.  **Given** the client has signed the challenge with a passkey, **When** it sends a `POST` request to `/api/v1/auth/passkeys/authenticate` with the signed response, **Then** the backend verifies the signature, updates the passkey `lastUsedAt` and counter, and returns `200 OK` with `accessToken`, `refreshToken`, and `userId`.
8.  **Given** an authenticated user with a valid refresh token, **When** the client sends a `POST` request to `/api/v1/auth/refresh` with the `refreshToken`, **Then** the backend issues a new `accessToken` and `refreshToken`, revokes the old refresh token, and returns `200 OK`.
9.  **Given** an authenticated user, **When** the client sends a `POST` request to `/api/v1/auth/logout` with the `refreshToken`, **Then** the backend revokes the refresh token and returns `200 OK`.
10. **Given** a user requests a magic link, **When** more than 5 requests are made for the same email within 1 hour, **Then** the backend returns `429 Too Many Requests`.

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
- **FR-015**: When a new user is created during magic link verification, the system MUST initialize default records for `UserLearningProfile` (correctionStyle: `moderate`), `UserUsageStats` (all zeros), and `UserPreferences` (timezone: `UTC`, uiLanguage: `en`). The `displayName` defaults to the email local part.
- **FR-016**: The system MUST support magic link authentication. A single-use token (hashed) with 15-minute expiry is generated, emailed to the user, and verified on click. If no `User` exists for the email, one is created with defaults.
- **FR-017**: The system MUST support WebAuthn passkey registration and authentication. A user can register multiple passkeys (one-to-many). Passkey login returns JWT tokens.
- **FR-018**: The system MUST issue JWT access tokens (15 min expiry) and refresh tokens (7 day expiry). Refresh tokens are stored hashed and support rotation and revocation.
- **FR-019**: All authenticated endpoints (User Story 1 and 2) MUST be protected by a JWT auth guard that validates the access token from the `Authorization: Bearer <token>` header.
- **FR-020**: The system MUST rate-limit magic link requests to 5 per email per hour to prevent abuse.
- **FR-021**: The email delivery MUST be abstracted behind a port interface (`IEmailService`) so the provider can be swapped without changing business logic.
- **FR-022**: Magic link MUST remain available as a login fallback even after a user has registered passkeys.

### Key Entities *(include if feature involves data)*

- **User**: Represents a learner. Key attributes: `user_id`, `display_name`, `email`, `created_at`, `updated_at`, `last_active_at`.
- **UserLearningProfile**: One-to-one with User. Key attributes: `user_id`, `learning_level` (CEFR A1–C2), `native_language`, `learning_goals`, `correction_style`, `topics_of_interest`.
- **UserUsageStats**: One-to-one with User. Key attributes: `user_id`, `total_conversation_count`, `total_practice_minutes`, `current_streak`, `longest_streak`.
- **UserPreferences**: One-to-one with User. Key attributes: `user_id`, `timezone`, `ui_language`.
- **MagicLinkToken**: Single-use authentication token. Key attributes: `token_id`, `email`, `token_hash`, `expires_at`, `used_at`, `created_at`.
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
