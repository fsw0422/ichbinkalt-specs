# Data Model: AI Conversation API

This document defines the key data entities for the AI Conversation API feature, based on the feature specification. Auth-related entities (User, UserPasskey, MagicLinkToken, RefreshToken) are stored in a single DynamoDB table (`ichbinkalt`) using `pk`/`sk` string keys, with the shadow-account lifecycle modeled on the User aggregate instead of a separate account type. Domain and API field names remain descriptive; the DynamoDB adapter maps them to sequential physical attribute names using `A`, `B`, `C`, ... `Z`, then `AA`, `AB`, and so on.

## 0. Auth Storage Layout

Auth data uses the following item shapes:

- User aggregate: `pk = A#{userId}`, `sk = A`
- User passkey: `pk = A#{userId}`, `sk = B#{credentialId}`
- Refresh token: `pk = A#{userId}`, `sk = C#{tokenId}`
- Magic link token: `pk = B#{tokenId}`, `sk = A` with TTL on `AB`
- Email lookup item: `pk = C#{normalizedEmail}`, `sk = A` to enforce unique email addresses

Secondary indexes:

- `user-handle-index`: resolves opaque WebAuthn user handles to the owning user via `B`
- `token-hash-index`: resolves hashed emailed-auth and refresh tokens during verification via `AA`

### 0.1 Sequential DynamoDB Physical Map

The entity sections below keep the domain-field names. The physical DynamoDB partition-key markers, sort-key markers, and attribute aliases each use their own alphabetical sequence. The key markers below are literal values stored inside `pk` and `sk`, while the attribute aliases are the field names stored alongside them.

#### Partition-Key and Sort-Key Marker Values

| Meaning | Key | Physical value | Notes |
|---------|-----|----------------|-------|
| User partition | `pk` | `A#<userId>` | Root partition for all auth data owned by a user. |
| Magic-link token partition | `pk` | `B#<tokenId>` | Standalone partition for emailed auth tokens. |
| Email lookup partition | `pk` | `C#<normalizedEmail>` | Standalone lookup partition for uniqueness and reverse resolution. |
| Core singleton item | `sk` | `A` | Root row within a user partition or any standalone singleton partition. |
| Passkey item | `sk` | `B#<credentialId>` | Sort key for a passkey child item within a user partition. |
| Refresh token item | `sk` | `C#<tokenId>` | Sort key for a refresh-token child item within a user partition. |

#### Common and User Aggregate Fields

| Domain field | DDB attr | Notes |
|--------------|----------|-------|
| `userId` | `A` | User identifier used across auth items. |
| `userHandle` | `B` | WebAuthn user handle and GSI key. |
| `accountState` | `C` | `shadow` or `full`. |
| `displayName` | `D` | User-facing name. |
| `email` / `normalizedEmail` | `E` | Stored normalized for lookup paths. |
| `emailVerifiedAt` | `F` | Last email verification timestamp. |
| `createdAt` | `G` | Creation timestamp. |
| `updatedAt` | `H` | Last update timestamp. |
| `lastActiveAt` | `I` | Last active timestamp. |
| `promotedAt` | `J` | Promotion-to-full timestamp. |
| `learningProfile` | `K` | Nested document on the core singleton (`sk = A`). |
| `usageStats` | `L` | Nested document on the core singleton (`sk = A`). |
| `preferences` | `M` | Nested document on the core singleton (`sk = A`). |

#### Nested Document Fields

| Domain field | DDB attr | Notes |
|--------------|----------|-------|
| `learningLevel` | `N` | CEFR level. |
| `nativeLanguage` | `O` | Native language. |
| `learningGoals` | `P` | Learning-goal array. |
| `correctionStyle` | `Q` | Correction mode. |
| `topicsOfInterest` | `R` | Preferred topics. |
| `totalConversationCount` | `S` | Usage counter. |
| `totalPracticeMinutes` | `T` | Aggregated practice minutes. |
| `currentStreak` | `U` | Current streak counter. |
| `longestStreak` | `V` | Longest streak counter. |
| `timezone` | `W` | IANA timezone. |
| `uiLanguage` | `X` | UI language code. |

#### Token and Passkey Fields

| Domain field | DDB attr | Notes |
|--------------|----------|-------|
| `tokenId` | `Y` | Token identifier. |
| `purpose` | `Z` | `sign_in` or `passkey_recovery`. |
| `tokenHash` | `AA` | Hashed token and GSI key. |
| `expiresAt` | `AB` | TTL attribute for expiring items. |
| `usedAt` | `AC` | Consumed-at timestamp. |
| `revokedAt` | `AD` | Revocation timestamp. |
| `passkeyId` | `AE` | Passkey identifier. |
| `credentialId` | `AF` | WebAuthn credential ID. |
| `publicKey` | `AG` | Stored public key. |
| `counter` | `AH` | WebAuthn signature counter. |
| `deviceName` | `AI` | Friendly device label. |
| `lastUsedAt` | `AJ` | Last successful passkey use. |

### 0.2 Shadow Account Lifecycle

- The first `POST /api/v1/auth/magic-link` for an email creates or reuses a User aggregate with `accountState = shadow`.
- `POST /api/v1/auth/verify-magic-link` marks the email verified and issues a session for that same user record.
- The first verified shadow session must register at least one passkey before onboarding is complete.
- `POST /api/v1/auth/passkeys/recovery-link` sends a recovery email whose link is verified through the same token-verification endpoint but forces the next client step to be passkey registration.
- The same user record remains a shadow account until a future billing/paywall flow promotes `accountState` to `full`, setting `promotedAt`.
- Explicit logout revokes session tokens; a later login still begins with fresh email verification, even if passkeys exist.

## 1. User

Represents a learner using the application. Shadow accounts and full accounts share the same User aggregate, with lifecycle tracked by `accountState`. The User aggregate is anchored by a singleton core item with `sk = A` and carries the fields returned by the user profile API. `learningProfile`, `usageStats`, and `preferences` are modeled separately below, but they are stored as nested documents inside that same core item. The tables in this section use the domain names; see the sequential physical map above for the physical DynamoDB aliases.

### 1.1 User (core identity)

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `userId` | `string` (UUID) | yes | Unique identifier for the user. |
| `userHandle` | `string` (UUID or opaque ID) | yes | Opaque unique WebAuthn user handle used during passkey ceremonies. |
| `accountState` | `enum` (`shadow`, `full`) | yes | Lifecycle state for the account. `shadow` users can access the app pre-paywall; `full` users have been promoted later without creating a second user record. |
| `displayName` | `string` | yes | The user's display name. |
| `email` | `string` | yes | Unique email address used as the account identifier. Comparisons use a normalized lowercase form. |
| `emailVerifiedAt` | `Date` | no | Last successful magic-link verification timestamp. Null until the email has been verified. |
| `createdAt` | `Date` | yes | Account creation timestamp. |
| `updatedAt` | `Date` | yes | Last profile update timestamp. |
| `lastActiveAt` | `Date` | yes | Last conversation activity timestamp. |
| `promotedAt` | `Date` | no | Timestamp when the shadow account was promoted to `full`. Null while still shadow. |

### 1.2 UserLearningProfile

Nested within the user core item (`sk = A`). Captures language learning context used to personalize AI conversations.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `userId` | `string` (UUID) | yes | ID of the owning User. |
| `learningLevel` | `enum` (A1, A2, B1, B2, C1, C2) | yes | CEFR proficiency level. |
| `nativeLanguage` | `string` | yes | The user's native language, so the AI can tailor explanations. |
| `learningGoals` | `enum[]` (`travel`, `work`, `academic`, `immigration`, `hobby`) | no | Why the user is learning German. |
| `correctionStyle` | `enum` (`gentle`, `moderate`, `strict`) | yes | How aggressively the AI corrects mistakes. Default: `moderate`. |
| `topicsOfInterest` | `string[]` | no | Preferred conversation topics (e.g. "cooking", "technology"). |

### 1.3 UserUsageStats

Nested within the user core item (`sk = A`). Denormalized usage statistics updated after each conversation.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `userId` | `string` (UUID) | yes | ID of the owning User. |
| `totalConversationCount` | `int` | yes | Number of completed conversations. Default: 0. |
| `totalPracticeMinutes` | `int` | yes | Cumulative practice time in minutes. Default: 0. |
| `currentStreak` | `int` | yes | Consecutive days with at least one conversation. Default: 0. |
| `longestStreak` | `int` | yes | All-time best streak. Default: 0. |

### 1.4 UserPreferences

Nested within the user core item (`sk = A`). Non-learning settings.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `userId` | `string` (UUID) | yes | ID of the owning User. |
| `timezone` | `string` | yes | IANA timezone (e.g. `Europe/Berlin`). Used for streak calculation. |
| `uiLanguage` | `string` | yes | Language for non-AI UI text. Default: `en`. |

### 1.5 MagicLinkToken

Single-use token for emailed authentication and recovery links. Token value is hashed before storage. Tokens are issued for an existing shadow or full user and therefore carry the owning `userId` in addition to the email address.

Storage key: `pk = B#{tokenId}`, `sk = A`. TTL is applied on `AB`.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `tokenId` | `string` (UUID) | yes | Unique identifier. |
| `userId` | `string` (UUID) | yes | Owning user record targeted by this magic link. |
| `email` | `string` | yes | Email address the link was sent to. |
| `purpose` | `enum` (`sign_in`, `passkey_recovery`) | yes | Whether the token is for normal sign-in or for direct entry into passkey recovery/registration. |
| `tokenHash` | `string` | yes | SHA-256 hash of the token sent in the email. |
| `expiresAt` | `Date` | yes | Token expiry (15 minutes from creation). |
| `usedAt` | `Date` | no | Timestamp when token was consumed. Null if unused. |
| `createdAt` | `Date` | yes | Creation timestamp. |

### 1.6 UserPasskey

WebAuthn credential for passkey-based login. One-to-many with User (multiple devices).

Storage key: `pk = A#{userId}`, `sk = B#{credentialId}`.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `passkeyId` | `string` (UUID) | yes | Unique identifier. |
| `userId` | `string` (UUID) | yes | ID of the owning User. |
| `credentialId` | `string` | yes | WebAuthn credential ID (base64url-encoded). |
| `publicKey` | `string` | yes | COSE public key (base64url-encoded). |
| `counter` | `int` | yes | Signature counter for clone detection. |
| `deviceName` | `string` | no | User-provided label (e.g. "My Phone"). |
| `createdAt` | `Date` | yes | Registration timestamp. |
| `lastUsedAt` | `Date` | yes | Last successful authentication timestamp. |

### 1.7 RefreshToken

JWT refresh token stored hashed for rotation and revocation.

Storage key: `pk = A#{userId}`, `sk = C#{tokenId}`. TTL is applied on `AB`, while `AD` handles immediate invalidation.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `tokenId` | `string` (UUID) | yes | Unique identifier. |
| `userId` | `string` (UUID) | yes | ID of the owning User. |
| `tokenHash` | `string` | yes | SHA-256 hash of the refresh token. |
| `expiresAt` | `Date` | yes | Token expiry (7 days from creation). |
| `revokedAt` | `Date` | no | Timestamp when token was revoked. Null if active. |
| `createdAt` | `Date` | yes | Creation timestamp. |

## 2. Conversation

Represents a single interactive session between a user and the AI. This will be the primary object managed in memory.

- **Entity**: `Conversation`
- **Attributes**:
  - `conversationId`: `string` (UUID) - Unique identifier for the session.
  - `userId`: `string` (UUID) - The ID of the user who initiated the conversation.
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

