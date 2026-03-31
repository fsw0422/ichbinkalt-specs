# Data Model: AI Conversation API

This document defines the key data entities for the AI Conversation API feature, based on the feature specification. Since the initial implementation uses volatile in-memory storage, these models will be represented as TypeScript classes/interfaces in the NestJS backend.

## 1. User

Represents a learner using the application. This model is included for conceptual clarity, but will not be persisted in the initial version.

- **Entity**: `User`
- **Attributes**:
  - `userId`: `string` (UUID) - Unique identifier for the user.
  - `name?`: `string` - The user's display name.
  - `learningLevel?`: `string` - The user's self-assessed German proficiency (e.g., A1, B2).

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

