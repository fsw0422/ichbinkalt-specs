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

### Key Entities *(include if feature involves data)*

- **User**: Represents a learner using the application. Key attributes: `user_id`, `name`, `learning_level`.
- **Conversation**: Represents a single interactive session between a user and the AI. Key attributes: `conversation_id`, `user_id`, `start_time`, `end_time`.
- **Message**: Represents a single turn in a conversation. Key attributes: `message_id`, `conversation_id`, `sender` (user or AI), `text_content`, `audio_data` (optional), `timestamp`. Transcriptions from both user input audio and AI output audio are stored as Message entities.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 95% of user audio utterances should be transcribed with acceptable accuracy using Gemini's automatic STT.
- **SC-002**: Latency from the end of a user's speech to receiving the first transcription event must be less than 500ms.
- **SC-003**: The p95 latency for the AI's text response to be delivered to the client must be under 3 seconds after the user's turn ends.
- **SC-004**: The backend must support at least 100 concurrent active conversations while maintaining the defined performance success criteria.
- **SC-005**: The WebSocket connection must reliably maintain state for at least 5 minutes of user inactivity.
