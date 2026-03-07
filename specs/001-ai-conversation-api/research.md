# Research: AI Conversation API

This document outlines the research and decisions made to resolve the "NEEDS CLARIFICATION" items in the implementation plan.

## 1. Language and Framework

- **Decision**: NestJS (TypeScript).
- **Rationale**: NestJS is a powerful and progressive Node.js framework for building efficient, reliable, and scalable server-side applications. Its modular architecture, built on TypeScript, makes it highly suitable for this project. It has first-class support for WebSockets through its gateways feature, which is perfect for our real-time communication needs. The strong typing provided by TypeScript helps in maintaining a clean and robust codebase, and its dependency injection system simplifies managing services for STT, TTS, and LLM integrations.
- **Alternatives Considered**:
  - **FastAPI (Python)**: Initially considered, but the team has a preference for the TypeScript ecosystem. While Python has a strong AI/ML ecosystem, the core logic of this service is I/O-bound orchestration, which Node.js excels at.
  - **Express/Fastify (Node.js)**: While both are excellent and fast, NestJS provides a more structured, opinionated framework out-of-the-box, which helps in maintaining consistency and scalability as the project grows.

## 2. Unified Audio Solution: Gemini Native Audio

- **Decision**: Google Gemini Native Audio Model (All-in-One Solution)
- **Rationale**: After initial research, we discovered that Google's Gemini API now offers a **Native Audio Model** (`gemini-2.5-flash-native-audio-preview-09-2025`) that combines Speech-to-Text, Text-to-Speech, and Conversational AI in a single, integrated service. This represents a significant architectural simplification over using separate providers.

### Benefits of Gemini Native Audio:

1. **Single Provider Integration**: Only requires one API key and one SDK (`@google/genai`)
2. **Lower Latency**: No need to chain multiple services (STT → LLM → TTS), reducing round-trip time
3. **Better Context Understanding**: The model processes audio input and generates audio responses within the same context, leading to more natural conversations
4. **Cost Efficiency**: Consolidated API calls and simplified billing
5. **Architectural Simplicity**: One service implementation instead of three separate adapters
6. **Real-time Streaming**: Native WebSocket support for bidirectional audio streaming
7. **High Quality German Support**: Built-in German language support for both transcription and synthesis

### Technical Specifications:

**For Real-time Conversations:**
- Model: `gemini-2.5-flash-native-audio-preview-09-2025`
- Input: PCM audio at 16kHz, 16-bit, mono
- Output: Both text transcription and PCM audio at 24kHz, 16-bit, mono
- Connection: WebSocket for bidirectional streaming

**For High-Quality TTS:**
- Model: `gemini-2.5-flash-preview-tts`
- Voice: "Kore" (natural German voice)
- Format: PCM audio output, base64 encoded

This consolidation aligns perfectly with our constitutional principle of "Lean Maintainability" by minimizing dependencies while maintaining all required functionality.

## 3. Speech-to-Text Transcription via Gemini Live API

- **Decision**: Use Gemini Live API's built-in transcription feature
- **Rationale**: Gemini Live API provides native support for both input and output audio transcription through its `BidiGenerateContentSetup` configuration. This eliminates the need for a separate STT service and provides synchronous transcription alongside the audio stream.

### Transcription Configuration

The Gemini Live WebSocket API supports enabling transcription during session setup:

```json
{
  "setup": {
    "model": "models/gemini-2.5-flash-native-audio-preview-09-2025",
    "inputAudioTranscription": {},
    "outputAudioTranscription": {}
  }
}
```

### How It Works

1. **Input Audio Transcription (`inputAudioTranscription`)**: When enabled, Gemini Live transcribes the user's audio input as it's streamed. The backend receives `BidiGenerateContentTranscription` messages containing the transcribed text of what the user said.

2. **Output Audio Transcription (`outputAudioTranscription`)**: When enabled, Gemini Live provides transcriptions of the AI's audio responses. This allows the client to display what the AI is saying in text form alongside the audio playback.

### Server Message Structure

Transcriptions are delivered via `BidiGenerateContentServerMessage`:

```json
{
  "serverContent": {
    "inputTranscription": {
      "text": "Hallo, wie geht es dir?"
    }
  }
}
```

```json
{
  "serverContent": {
    "outputTranscription": {
      "text": "Mir geht es gut, danke! Wie kann ich dir heute helfen?"
    }
  }
}
```

### Benefits of Native Transcription

1. **Synchronous Delivery**: Transcriptions arrive in real-time alongside the audio stream
2. **No Additional API Calls**: Eliminates the need for a separate STT service like Google Cloud Speech-to-Text
3. **Language Alignment**: Transcription matches the configured audio language (German/English)
4. **Lower Latency**: No round-trip to an external service; transcription is part of the same WebSocket stream
5. **Cost Efficiency**: Bundled with the Gemini Live API usage, no separate billing

### Implementation Approach

1. Enable `inputAudioTranscription` and `outputAudioTranscription` in the session setup
2. Parse incoming `serverContent` messages for `inputTranscription` and `outputTranscription` fields
3. Emit `input_transcription` and `output_transcription` WebSocket events to connected clients
4. Persist transcriptions as `Message` entities in the `ConversationRepository`

### Transcription Storage

Transcriptions are saved to the conversation history as `Message` entities:
- **User transcriptions**: `sender: 'user'`, `textContent: <transcribed text>`
- **AI transcriptions**: `sender: 'ai'`, `textContent: <transcribed text>`

This allows clients to retrieve the full conversation transcript for review and ensures a complete audit trail of the conversation.
