# Data Model: AI Conversation API

This document defines the key data entities for the AI Conversation API feature, based on the feature specification. Since the initial implementation uses volatile in-memory storage, these models will be represented as TypeScript classes/interfaces in the NestJS backend.

## 1. User

Represents a learner using the application. The User entity is composed of a core identity record and related detail tables for learning profile, usage stats, and preferences.

### 1.1 User (core identity)

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `userId` | `string` (UUID) | yes | Unique identifier for the user. |
| `displayName` | `string` | yes | The user's display name. |
| `email` | `string` | yes | The user's email address (account identifier, not auth). |
| `createdAt` | `Date` | yes | Account creation timestamp. |
| `updatedAt` | `Date` | yes | Last profile update timestamp. |
| `lastActiveAt` | `Date` | yes | Last conversation activity timestamp. |

### 1.2 UserLearningProfile

One-to-one with User. Captures language learning context used to personalize AI conversations.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `userId` | `string` (UUID) | yes | FK → User. |
| `learningLevel` | `enum` (A1, A2, B1, B2, C1, C2) | yes | CEFR proficiency level. |
| `nativeLanguage` | `string` | yes | The user's native language, so the AI can tailor explanations. |
| `learningGoals` | `enum[]` (`travel`, `work`, `academic`, `immigration`, `hobby`) | no | Why the user is learning German. |
| `correctionStyle` | `enum` (`gentle`, `moderate`, `strict`) | yes | How aggressively the AI corrects mistakes. Default: `moderate`. |
| `topicsOfInterest` | `string[]` | no | Preferred conversation topics (e.g. "cooking", "technology"). |

### 1.3 UserUsageStats

One-to-one with User. Denormalized usage statistics updated after each conversation.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `userId` | `string` (UUID) | yes | FK → User. |
| `totalConversationCount` | `int` | yes | Number of completed conversations. Default: 0. |
| `totalPracticeMinutes` | `int` | yes | Cumulative practice time in minutes. Default: 0. |
| `currentStreak` | `int` | yes | Consecutive days with at least one conversation. Default: 0. |
| `longestStreak` | `int` | yes | All-time best streak. Default: 0. |

### 1.4 UserPreferences

One-to-one with User. Non-learning settings.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `userId` | `string` (UUID) | yes | FK → User. |
| `timezone` | `string` | yes | IANA timezone (e.g. `Europe/Berlin`). Used for streak calculation. |
| `uiLanguage` | `string` | yes | Language for non-AI UI text. Default: `en`. |

### 1.5 MagicLinkToken

Single-use token for passwordless email authentication. Token value is hashed before storage.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `tokenId` | `string` (UUID) | yes | Unique identifier. |
| `email` | `string` | yes | Email address the link was sent to. |
| `tokenHash` | `string` | yes | SHA-256 hash of the token sent in the email. |
| `expiresAt` | `Date` | yes | Token expiry (15 minutes from creation). |
| `usedAt` | `Date` | no | Timestamp when token was consumed. Null if unused. |
| `createdAt` | `Date` | yes | Creation timestamp. |

### 1.6 UserPasskey

WebAuthn credential for passkey-based login. One-to-many with User (multiple devices).

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `passkeyId` | `string` (UUID) | yes | Unique identifier. |
| `userId` | `string` (UUID) | yes | FK → User. |
| `credentialId` | `string` | yes | WebAuthn credential ID (base64url-encoded). |
| `publicKey` | `string` | yes | COSE public key (base64url-encoded). |
| `counter` | `int` | yes | Signature counter for clone detection. |
| `deviceName` | `string` | no | User-provided label (e.g. "My Phone"). |
| `createdAt` | `Date` | yes | Registration timestamp. |
| `lastUsedAt` | `Date` | yes | Last successful authentication timestamp. |

### 1.7 RefreshToken

JWT refresh token stored hashed for rotation and revocation.

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `tokenId` | `string` (UUID) | yes | Unique identifier. |
| `userId` | `string` (UUID) | yes | FK → User. |
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

