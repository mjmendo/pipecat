# Topic 1: Voice Bot Pipeline — End-to-End Overview

## What Is Pipecat?

Pipecat is an open-source Python framework for building real-time voice and multimodal conversational AI agents. It orchestrates audio/video, AI services, transports, and conversation pipelines using a frame-based architecture.

## The Core Concept: Frames + Pipelines

Everything flows as **Frame** objects through a chain of **FrameProcessors**:

```
[Transport Input] → [STT] → [User Aggregator] → [LLM] → [TTS] → [Transport Output] → [Assistant Aggregator]
```

- **Frames** = data units (audio chunks, text, images) and control signals
- **Processors** = units that receive, transform, and push frames
- **Pipelines** = chains of processors
- **Transports** = I/O layer (WebRTC via Daily/LiveKit, WebSocket, local mic/speaker)

## Step-by-Step Flow

### 1. Audio In — Transport Input
`Transport Input` (WebRTC/WebSocket) receives raw PCM audio from the user's microphone and pushes `AudioRawFrame` downstream.

### 2. Speech-to-Text — STTService
`STTService` consumes `AudioRawFrame`, sends audio to the provider (e.g. Deepgram), and pushes `TranscriptionFrame` with the recognized text.

### 3. Turn Aggregation — User Aggregator
`User Aggregator` collects transcription frames until the user finishes speaking (detected by VAD). Then it releases an `LLMMessagesFrame` containing the full conversation context.

### 4. Language Model — LLMService
`LLMService` receives `LLMMessagesFrame`, streams the response from the provider (e.g. OpenAI), and pushes `LLMTextFrame` tokens one by one.

### 5. Text-to-Speech — TTSService
`TTSService` buffers tokens until a sentence boundary, then calls the TTS provider (e.g. ElevenLabs) and pushes `TTSAudioRawFrame` chunks.

### 6. Audio Out — Transport Output
`Transport Output` sends the audio chunks to the user's speaker via WebRTC/WebSocket.

### 7. Context Update — Assistant Aggregator
`Assistant Aggregator` (placed after transport output so it doesn't block audio delivery) records the bot's response back into the shared `LLMContext` for the next turn.

## Frame Types at Each Stage

| Stage | Frame Type | Description |
|-------|-----------|-------------|
| Transport In → STT | `AudioRawFrame` | Raw PCM audio bytes |
| STT → User Aggregator | `TranscriptionFrame` | Recognized text |
| User Aggregator → LLM | `LLMMessagesFrame` | Full conversation context |
| LLM → TTS | `LLMTextFrame` | Streamed response tokens |
| TTS → Transport Out | `TTSAudioRawFrame` | Synthesized audio chunks |

## Interruptions

When the user speaks while the bot is talking, VAD triggers an `InterruptionFrame` that flows downstream through every processor, clearing all queued frames — the bot stops mid-sentence and listens.

The interruption path:
1. VAD detects speech → `VADUserStartedSpeakingFrame`
2. Turn start strategy pushes `InterruptionTaskFrame` upstream
3. `PipelineTask` converts it to `InterruptionFrame` and sends it downstream
4. Each processor's `_start_interruption()` fires: cancels process task, drains queue
5. TTS cancels ongoing audio generation, pushes `BotStoppedSpeakingFrame`
6. Pipeline is clean and ready for the new user utterance

## Execution Layer

- **PipelineRunner** — entry point, handles SIGINT/SIGTERM for graceful shutdown
- **PipelineTask** — sends `StartFrame` to boot the pipeline, manages heartbeats, idle timeouts, observers

## Additional Details

### Shared LLMContext
The User Aggregator and Assistant Aggregator are created together as an `LLMContextAggregatorPair`. They share a single `LLMContext` object, so both sides of the conversation accumulate into the same message history.

### VAD (Voice Activity Detection)
The transport input typically runs a VAD analyzer that emits `VADUserStartedSpeakingFrame` / `VADUserStoppedSpeakingFrame`. These propagate downstream and are what the User Aggregator uses to know when the user's turn is complete.

### User Turn Strategies
VAD alone doesn't trigger interruptions. A *turn start strategy* (e.g. `VADUserTurnStartStrategy`) interprets the VAD signal and decides whether to push `UserStartedSpeakingFrame` / `UserStoppedSpeakingFrame`. This separation lets you swap strategies (e.g. push-to-talk vs. voice-activity) without changing the pipeline.

### StartFrame Bootstrapping
Nothing flows until `PipelineTask` sends a `StartFrame` downstream. Each processor's `start()` hook fires when it receives this frame, which is when services open connections to their providers.

### Metrics
STT tracks TTFB (time from user stops speaking to first transcription), LLM tracks TTFB (time from context sent to first token), TTS tracks TTFB similarly. These are pushed as `MetricsData` and can be observed/logged.

### Graceful Shutdown
`EndFrame` (graceful, waits for in-flight work) vs `CancelFrame` (immediate) are both `UninterruptibleFrame` subclasses, meaning they survive interruptions and always reach every processor.

## Diagram

See `01-voice-bot-pipeline.excalidraw` for the visual diagram.

## Reference Files

| File | Purpose |
|------|---------|
| `examples/foundational/06-listen-and-respond.py` | Canonical voice bot example |
| `src/pipecat/pipeline/pipeline.py` | Pipeline chaining |
| `src/pipecat/pipeline/task.py` | PipelineTask orchestration |
| `src/pipecat/pipeline/runner.py` | PipelineRunner entry point |
| `src/pipecat/services/stt_service.py` | STT base class |
| `src/pipecat/services/llm_service.py` | LLM base class |
| `src/pipecat/services/tts_service.py` | TTS base class |
| `src/pipecat/transports/base_transport.py` | Transport abstractions |
| `src/pipecat/processors/frame_processor.py` | Base processor |
