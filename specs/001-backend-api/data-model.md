# Data Model: AI Conversation API

This document defines the key data entities for the AI Conversation API feature, based on the feature specification. Auth-related entities (`UserCore`, `UserLearningProfile`, `UserUsageStats`, `UserPreferences`, `UserPasskey`, `MagicLinkToken`, `RefreshToken`) are stored in a single DynamoDB table (`ichbinkalt`) using `pk`/`sk` string keys. The user partition now centers on a minimal user core item plus separate singleton child items for profile, usage, and preferences. Passkey registration is the first sign-up step after email entry, while emailed tokens are reserved for later inbox verification and passkey recovery.

## 0. Auth Storage Layout

Auth data uses the following item shapes:

- User core: `pk = A#{userId}`, `sk = A`
- User passkey: `pk = A#{userId}`, `sk = B#{credentialId}`
- Refresh token: `pk = A#{userId}`, `sk = C#{tokenId}`
- User learning profile: `pk = A#{userId}`, `sk = D`
- User usage stats: `pk = A#{userId}`, `sk = E`
- User preferences: `pk = A#{userId}`, `sk = F`
- Magic link token: `pk = B#{tokenId}`, `sk = A`
- Email lookup item: `pk = C#{normalizedEmail}`, `sk = A` to enforce exactly one user core per normalized email address

Secondary indexes:

- `token-hash-index`: resolves hashed emailed bootstrap and refresh tokens during verification.

### 0.1 Physical Key and Item Map

The entity sections below keep the domain-field names. The key markers below are literal values stored inside `pk` and `sk`. The important invariant for this phase is item separation: the user core row stays minimal, and profile, usage, and preferences live in separate singleton child items rather than as nested documents on `sk = A`.

#### Partition-Key and Sort-Key Marker Values

| Meaning | Key | Physical value | Notes |
|---------|-----|----------------|-------|
| User partition | `pk` | `A#<userId>` | Root partition for all auth data owned by a user. |
| Email-link token partition | `pk` | `B#<tokenId>` | Standalone partition for emailed verification and recovery tokens. |
| Email lookup partition | `pk` | `C#<normalizedEmail>` | Standalone lookup partition for uniqueness and reverse resolution. |
| User core item | `sk` | `A` | Minimal identity row within a user partition. |
| Passkey item | `sk` | `B#<credentialId>` | Sort key for a passkey child item within a user partition. |
| Refresh token item | `sk` | `C#<tokenId>` | Sort key for a refresh-token child item within a user partition. |
| Learning profile item | `sk` | `D` | Singleton learning-profile row within a user partition. |
| Usage stats item | `sk` | `E` | Singleton usage-stats row within a user partition. |
| Preferences item | `sk` | `F` | Singleton preferences row within a user partition. |

#### User Core Fields

| Domain field | Notes |
|--------------|-------|
| `userId` | User identifier used across all user-owned auth items. |
| `email` / `normalizedEmail` | Stored normalized for lookup paths. |
| `displayName` | User-facing name. |
| `createdAt` | Creation timestamp. |
| `updatedAt` | Last update timestamp. |
| `lastActiveAt` | Last active timestamp. |
| `emailConfirmed` | Durable email-confirmation flag. |

#### User-Owned Singleton Child Items

| Item | `sk` | Stored fields |
|------|------|---------------|
| `UserLearningProfile` | `D` | `learningLevel`, `nativeLanguage`, `learningGoals`, `correctionStyle`, `topicsOfInterest` |
| `UserUsageStats` | `E` | `totalConversationCount`, `totalPracticeMinutes`, `currentStreak`, `longestStreak` |
| `UserPreferences` | `F` | `timezone`, `uiLanguage` |

#### Token and Passkey Fields

| Item | Stored fields |
|------|---------------|
| `MagicLinkToken` | `tokenId`, `userId`, `email`, `purpose`, `tokenHash`, `expiresAt`, `usedAt`, `createdAt` |
| `RefreshToken` | `tokenId`, `userId`, `tokenHash`, `expiresAt`, `revokedAt`, `createdAt` |
| `UserPasskey` | `passkeyId`, `userId`, `credentialId`, `publicKey`, `counter`, `deviceName`, `createdAt`, `lastUsedAt` |

### 0.2 Registration, Verification, and Recovery Lifecycle

- The first `POST /api/v1/auth/passkeys/registration-options` for an email creates or reuses a `UserCore` item with `emailConfirmed = false`.
- A normalized email address maps to exactly one `UserCore` item across sign-up, later email verification, recovery, reclaim, and passkey-authentication lookup.
- `POST /api/v1/auth/passkeys/register` stores the first passkey and issues a full application session without waiting for email verification.
- Returning users with a registered passkey sign in directly through the passkey authentication flow.
- `POST /api/v1/auth/email-verification-link` sends a single-use inbox-verification email for the currently signed-in user.
- `POST /api/v1/auth/verify-email` marks `emailConfirmed = true` for that same user record and unlocks subscription-based service eligibility.
- `POST /api/v1/auth/passkeys/recovery-link` sends a recovery email whose verified token auto-confirms the email address and returns a scoped bootstrap token for passkey re-registration.
- Recovery verification may also reclaim inbox ownership by revoking prior passkeys and refresh tokens and resetting temporary learner data before a fresh passkey is registered.
- Recovery-driven replacement passkey registration assumes the inbox has already been re-verified by the consumed recovery link and therefore does not require another email-confirmation step.
- Hard lifecycle control should be represented separately from the minimal user core row, for example by `accountStatus` such as `active` or `suspended`.
- Onboarding should be derived from whether at least one `UserPasskey` child item currently exists for the user.
- Product access should be represented separately by `subscriptionStatus` such as `free`, `trial`, `paid`, `grace`, or `past_due`.
- Paid services and revenue-bearing upgrades should require `emailConfirmed = true` in addition to whatever entitlement state is later introduced.
- Any future billing/paywall promotion should continue using the same user record without requiring a second user row.
- Explicit logout revokes session tokens; a later login resumes through passkey authentication unless passkey recovery is needed.

## 1. User Core and User-Owned Items

Represents a learner using the application. The `UserCore` item is the minimal identity row anchored at `sk = A`. `UserLearningProfile`, `UserUsageStats`, and `UserPreferences` are modeled as separate singleton child items under the same user partition at `sk = D`, `sk = E`, and `sk = F`.

### 1.1 UserCore

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `userId` | `string` (ULID) | yes | Unique identifier for the user. The same stable identifier is used as the WebAuthn user ID. |
| `emailConfirmed` | `boolean` | yes | Whether the user has completed a successful email-verification or recovery verification. This is the durable inbox-ownership proof and later the prerequisite for paid services. |
| `displayName` | `string` | yes | The user's display name. |
| `email` | `string` | yes | Unique email address used as the account identifier. Uniqueness and lookups use a trimmed lowercase normalized form. |
| `createdAt` | `Date` | yes | Account creation timestamp. |
| `updatedAt` | `Date` | yes | Last profile update timestamp. |
| `lastActiveAt` | `Date` | yes | Last conversation activity timestamp. |

Notes:
- `onboardingComplete` is derived from whether at least one `UserPasskey` exists for the user and is returned in API responses as a convenience snapshot.
- `accountStatus` and `subscriptionStatus` are logical user state, but they do not belong in the minimal `UserCore` item described here.
- The informal “shadow user” label maps to an unverified claimant to the unique email-bound account; it is not the primary authorization field.
- Recovery and reclaim always target the same unique user record bound to that normalized email address.
- Recovery verification also refreshes the authoritative `emailConfirmed` state for that same user record before replacement passkey registration occurs.
- Any learner/profile data created before the first successful email verification remains reclaimable and may be wiped if inbox ownership is later re-established through recovery.
- The full user response returned by APIs is assembled from the `UserCore` row plus the dedicated learning-profile, usage-stats, and preferences items.

### 1.2 UserLearningProfile

Stored as a singleton child item at `pk = A#{userId}`, `sk = D`. Captures language learning context used to personalize AI conversations.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `userId` | `string` (ULID) | yes | ID of the owning User. |
| `learningLevel` | `enum` (A1, A2, B1, B2, C1, C2) | yes | CEFR proficiency level. |
| `nativeLanguage` | `string` | yes | The user's native language, so the AI can tailor explanations. |
| `learningGoals` | `enum[]` (`travel`, `work`, `academic`, `immigration`, `hobby`) | no | Why the user is learning German. |
| `correctionStyle` | `enum` (`gentle`, `moderate`, `strict`) | yes | How aggressively the AI corrects mistakes. Default: `moderate`. |
| `topicsOfInterest` | `string[]` | no | Preferred conversation topics (e.g. "cooking", "technology"). |

### 1.3 UserUsageStats

Stored as a singleton child item at `pk = A#{userId}`, `sk = E`. Denormalized usage statistics updated after each conversation.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `userId` | `string` (ULID) | yes | ID of the owning User. |
| `totalConversationCount` | `int` | yes | Number of completed conversations. Default: 0. |
| `totalPracticeMinutes` | `int` | yes | Cumulative practice time in minutes. Default: 0. |
| `currentStreak` | `int` | yes | Consecutive days with at least one conversation. Default: 0. |
| `longestStreak` | `int` | yes | All-time best streak. Default: 0. |

### 1.4 UserPreferences

Stored as a singleton child item at `pk = A#{userId}`, `sk = F`. Non-learning settings.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `userId` | `string` (ULID) | yes | ID of the owning User. |
| `timezone` | `string` | yes | IANA timezone (e.g. `Europe/Berlin`). Used for streak calculation. |
| `uiLanguage` | `string` | yes | Language for non-AI UI text. Default: `en`. |

### 1.5 MagicLinkToken

Single-use token for emailed verification and recovery links. Token value is hashed before storage. Tokens are issued for an existing signed-up or already-onboarded user and therefore carry the owning `userId` in addition to the email address.

Storage key: `pk = B#{tokenId}`, `sk = A`. TTL is applied on the table's configured token-expiry attribute.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `tokenId` | `string` (UUID) | yes | Unique identifier. |
| `userId` | `string` (ULID) | yes | Owning user record targeted by this magic link. |
| `email` | `string` | yes | Email address the link was sent to. It resolves to the same unique user via normalized lookup. |
| `purpose` | `enum` (`email_verification`, `passkey_recovery`) | yes | Whether the token is for later inbox verification or for direct entry into passkey recovery, account reclaim, or replacement registration. |
| `tokenHash` | `string` | yes | SHA-256 hash of the token sent in the email. |
| `expiresAt` | `Date` | yes | Token expiry (15 minutes from creation). |
| `usedAt` | `Date` | no | Timestamp when token was consumed. Null if unused. |
| `createdAt` | `Date` | yes | Creation timestamp. |

### 1.6 BootstrapSession

Scoped auth token returned after verifying a recovery link. This is not stored as a DynamoDB item; it is a short-lived signed token or equivalent server-issued auth context that only authorizes passkey re-registration endpoints.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `token` | `string` | yes | The scoped bootstrap token returned to the client. |
| `tokenType` | `enum` (`bootstrap`) | yes | Explicit token category so bootstrap credentials cannot be confused with access tokens. |
| `tokenId` | `string` (UUID or ULID) | yes | Unique token identifier for audit and replay protection. |
| `userId` | `string` (ULID) | yes | ID of the user who is allowed to continue passkey registration. |
| `purpose` | `enum` (`passkey_recovery`) | yes | Indicates that the bootstrap token came from passkey recovery or account reclaim. |
| `scope` | `enum` (`passkey_registration`) | yes | Restricts the token to passkey-registration actions only. |
| `expiresAt` | `Date` | yes | Short-lived expiry for the bootstrap flow. |

### 1.7 AccessSession

Verified application session returned after successful first-passkey registration, recovery re-registration, or passkey authentication. The access token should carry identity and session facts only; mutable lifecycle and subscription decisions remain server-side.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `token` | `string` | yes | The signed access token returned to the client. |
| `tokenType` | `enum` (`access`) | yes | Explicit token category for protected API access. |
| `sessionId` | `string` (UUID or ULID) | yes | Session identifier used for tracing and optional revocation hooks. |
| `userId` | `string` (ULID) | yes | ID of the authenticated user. |
| `authLevel` | `enum` (`verified`) | yes | Confirms the user passed the required passkey ceremony. |
| `issuedAt` | `Date` | yes | Token issue timestamp. |
| `expiresAt` | `Date` | yes | Access-token expiry. |

### 1.8 UserPasskey

WebAuthn credential for passkey-based login. One-to-many with User (multiple devices).

Storage key: `pk = A#{userId}`, `sk = B#{credentialId}`.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `passkeyId` | `string` (UUID) | yes | Unique identifier. |
| `userId` | `string` (ULID) | yes | ID of the owning User. |
| `credentialId` | `string` | yes | WebAuthn credential ID (base64url-encoded). |
| `publicKey` | `string` | yes | COSE public key (base64url-encoded). |
| `counter` | `int` | yes | Signature counter for clone detection. |
| `deviceName` | `string` | no | User-provided label (e.g. "My Phone"). |
| `createdAt` | `Date` | yes | Registration timestamp. |
| `lastUsedAt` | `Date` | yes | Last successful authentication timestamp. |

### 1.9 RefreshToken

JWT refresh token stored hashed for rotation and revocation.

Storage key: `pk = A#{userId}`, `sk = C#{tokenId}`. TTL is applied on the table's configured token-expiry attribute, while `revokedAt` handles immediate invalidation.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `tokenId` | `string` (UUID) | yes | Unique identifier. |
| `userId` | `string` (ULID) | yes | ID of the owning User. |
| `tokenHash` | `string` | yes | SHA-256 hash of the refresh token. |
| `expiresAt` | `Date` | yes | Token expiry (7 days from creation). |
| `revokedAt` | `Date` | no | Timestamp when token was revoked. Null if active. |
| `createdAt` | `Date` | yes | Creation timestamp. |

## 2. Conversation

Represents a single interactive session between a user and the AI. This will be the primary object managed in memory.

- **Entity**: `Conversation`
- **Attributes**:
  - `conversationId`: `string` (UUID) - Unique identifier for the session.
  - `userId`: `string` (ULID) - The ID of the user who initiated the conversation.
  - `startTime`: `Date` - The timestamp when the conversation was created.
  - `lastActivityTime`: `Date` - The timestamp of the last message, used for session timeout.
  - `history`: `Message[]` - A list of all messages exchanged during the session.

## 3. Message

Represents a single turn in a conversation, either from the user or the AI. Messages contain the transcribed text content which is derived from speech-to-text processing of audio.

- **Entity**: `Message`
- **Attributes**:
  - `messageId`: `string` (UUID) - Unique identifier for the message.
  - `sender`: `'user' | 'ai'` - The origin of the message.
  - `textContent`: `string` - The transcribed text of the speech (from Gemini Live's `inputAudioTranscription` for user messages, `outputAudioTranscription` for AI messages).
  - `audioData?`: `string` - Base64-encoded audio data for AI responses (optional, for caching/playback).
  - `timestamp`: `Date` - The time the message was created.

## 4. Transcription Flow

The system uses Gemini Live's native transcription capabilities:

1. **User Speech → Text**: The client streams an utterance via `audio_chunk` events and then sends `audio_stream_end` to mark that utterance as complete. Gemini Live then finalizes processing and returns transcriptions via `inputTranscription` in the server messages. These are stored as `Message` entities with `sender: 'user'`.

2. **AI Speech → Text**: When Gemini Live generates audio responses, it simultaneously provides transcriptions via `outputTranscription`. These are stored as `Message` entities with `sender: 'ai'`.

### WebSocket Events for Transcription

| Event | Direction | Payload | Description |
|-------|-----------|---------|-------------|
| `audio_stream_end` | Client → Server | `{}` | Explicit signal that the current user audio stream has ended |
| `input_transcription` | Server → Client | `{ text: string, conversationId: string }` | Transcription of user's speech |
| `output_transcription` | Server → Client | `{ text: string, conversationId: string }` | Transcription of AI's speech |
| `ai_audio_response` | Server → Client | `{ audioData: string, mimeType: string }` | AI's audio response |

### Storage Responsibility

Transcriptions are persisted to the `ConversationRepository`:
- Each transcription creates a new `Message` entity
- Messages are appended to the `Conversation.history` array
- The repository's `update()` method is called to persist changes

