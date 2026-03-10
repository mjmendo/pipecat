# Topic 6: AI Services — STT, LLM, TTS

## What This Topic Covers

AI services are FrameProcessors that wrap external APIs (Deepgram, OpenAI, ElevenLabs, etc.) and translate between frames and API calls. They sit in the pipeline like any other processor, but internally they manage WebSocket connections, streaming responses, token counting, and provider-specific quirks.

This topic covers the three core service types (STT, LLM, TTS), their shared base class, the adapter pattern, and how metrics flow through the system.

## AIService — The Common Base

**File**: `src/pipecat/services/ai_service.py`

All services extend `AIService`, which extends `FrameProcessor`. It provides:

- **Settings management**: A `ServiceSettings` object that's runtime-updatable via delta mode (merge changed fields, don't replace the whole object). Settings changes arrive as `*UpdateSettingsFrame` and are applied via `_update_settings(delta)`.
- **Model tracking**: Automatically syncs model names to metrics data via `set_core_metrics_data()`.
- **Lifecycle hooks**: `start(frame)`, `stop(frame)`, `cancel(frame)` — called on `StartFrame`, `EndFrame`, `CancelFrame` respectively.
- **Generator processing**: `process_generator(generator)` consumes an async generator of frames, pushing each one downstream and catching errors automatically.

```
FrameProcessor
  └── AIService
        ├── STTService
        ├── LLMService
        └── TTSService
```

---

## STTService — Speech to Text

**File**: `src/pipecat/services/stt_service.py`

### Two flavors

| | STTService | SegmentedSTTService |
|---|---|---|
| **Pattern** | Streaming / continuous | Batch / VAD-triggered |
| **When to use** | Providers with real-time streaming (Deepgram, Google Cloud) | Providers that process complete segments (local Whisper) |
| **Audio handling** | Sends each audio chunk immediately to provider | Buffers all audio, sends as WAV on `VADUserStoppedSpeakingFrame` |
| **Interim results** | Yes (`InterimTranscriptionFrame`) | No — only final transcriptions |
| **Typical latency** | Low (streaming) | Higher (waits for full utterance) |

### Frames consumed and produced

**Consumes**:
- `AudioRawFrame` — raw audio chunks from the transport (via high-priority SystemFrame path)
- `VADUserStartedSpeakingFrame` / `VADUserStoppedSpeakingFrame` — from UserAggregator's VAD, used for metrics timing and finalization
- `STTMuteFrame` — mute/unmute transcription
- `STTUpdateSettingsFrame` — runtime settings changes (model, language)
- `InterruptionFrame` — resets metrics state

**Produces**:
- `TranscriptionFrame` — final transcription (`"What's the weather in Paris?"`)
- `InterimTranscriptionFrame` — partial result (`"What's the wea..."`) — only STTService, not Segmented
- `STTMetadataFrame` — broadcast at pipeline start with `ttfs_p99_latency` (so downstream turn strategies know expected latency)
- Audio passthrough — if `audio_passthrough=True` (default), input audio frames are pushed downstream so other processors (VAD in UserAggregator) can see them

### The one abstract method

```python
@abstractmethod
async def run_stt(self, audio: bytes) -> AsyncGenerator[Frame, None]:
    """Send audio to the provider, yield transcription frames."""
```

For streaming providers like Deepgram, `run_stt()` sends audio over a WebSocket and yields `None` — actual transcriptions arrive asynchronously via WebSocket callbacks. For batch providers, it yields `TranscriptionFrame` directly.

### Finalization pattern

STT needs to know when a transcription is "final" (not just an interim result that will be revised). Two approaches:

- **Server-confirmed** (Deepgram): On `VADUserStoppedSpeakingFrame`, the service calls `connection.finalize()`. When Deepgram responds with `from_finalize=True`, the next transcription is marked `finalized=True`.
- **Timeout-based** (others): Wait `stt_ttfb_timeout` seconds (default 2.0s) after the last transcript, then report whatever we have.

### WebsocketSTTService

A mixin that adds WebSocket connection management + keepalive. Periodically sends silent audio to prevent idle connection closure. Deepgram STT extends this.

### Concrete example: Deepgram

```python
class DeepgramSTTService(WebsocketSTTService):
    async def run_stt(self, audio: bytes):
        await self._connection.send(audio)
        yield None  # Results arrive via WebSocket callbacks

    # WebSocket callback:
    async def _on_message(self, result):
        if result.is_final:
            await self.push_frame(TranscriptionFrame(text=result.transcript))
        else:
            await self.push_frame(InterimTranscriptionFrame(text=result.transcript))
```

---

## LLMService — Language Model

**File**: `src/pipecat/services/llm_service.py`

The most complex service. Handles streaming token generation, function calling (covered in Topic 4 from the pipeline perspective — here we see the service internals), and context management.

### Frames consumed and produced

**Consumes**:
- `LLMContextFrame` — "here's the conversation context, generate a response"
- `InterruptionFrame` — cancels registered function calls (if `cancel_on_interruption=True`)
- `LLMUpdateSettingsFrame` — runtime settings changes (temperature, model, etc.)
- `LLMContextSummaryRequestFrame` — request context summarization (for token management)

**Produces**:
- `LLMFullResponseStartFrame` / `LLMFullResponseEndFrame` — bookends around each response
- `LLMTextFrame` — one per streaming token chunk (`"I"`, `"'m"`, `" a"`, `" voice"`, ...)
- `FunctionCallInProgressFrame` — broadcast both directions (see Topic 4)
- `FunctionCallResultFrame` — broadcast both directions
- `FunctionCallCancelFrame` — broadcast both directions
- `FunctionCallsStartedFrame` — signals function execution has begun

### Streaming token pattern

The base `LLMService` defines the scaffolding; concrete implementations (like `BaseOpenAILLMService`) handle the actual API call:

```python
# In BaseOpenAILLMService._process_context():
await self.push_frame(LLMFullResponseStartFrame())

async for chunk in api_stream:
    if chunk.choices[0].delta.content:
        await self._push_llm_text(chunk.choices[0].delta.content)  # → LLMTextFrame
    if chunk.choices[0].delta.tool_calls:
        # Accumulate function name + arguments across chunks
        function_name += tool_call.function.name
        arguments += tool_call.function.arguments

# After stream completes:
if function_calls:
    await self.run_function_calls(function_calls)

await self.push_frame(LLMFullResponseEndFrame())
```

Each `LLMTextFrame` flows immediately downstream to TTS — the LLM doesn't wait for the full response before sending tokens.

### Function calling internals

We covered the pipeline-level view in Topic 4. Here's the service-level detail:

**Registry**: `self._functions: Dict[Optional[str], FunctionCallRegistryItem]`
- Key is function name (or `None` for a catch-all handler)
- Value holds the handler, `cancel_on_interruption` flag, and metadata

**Execution modes**:
- **Parallel** (`run_in_parallel=True`): All function calls dispatched as concurrent tasks
- **Sequential** (`run_in_parallel=False`): Queued and executed one at a time via `_sequential_runner_handler`

**Per-call timeout**: Each function call gets a timeout task (default 10s). If the handler doesn't call `result_callback` in time, the timeout calls it with `None`. This prevents hung function calls from blocking the pipeline forever.

**Interruption handling**: When `InterruptionFrame` arrives, the service iterates all registered functions and cancels any with `cancel_on_interruption=True`.

### The adapter pattern

**File**: `src/pipecat/adapters/base_llm_adapter.py`

The LLM service uses a universal `LLMContext` that's provider-agnostic. The **adapter** converts this to the provider's format:

```python
class BaseLLMAdapter:
    def get_llm_invocation_params(self, context: LLMContext) -> TLLMInvocationParams:
        """Convert universal context → provider-specific params (messages, tools, tool_choice)"""

    def to_provider_tools_format(self, tools_schema: ToolsSchema) -> List[Any]:
        """Convert standardized ToolsSchema → provider's tool format"""
```

Each LLM service declares its adapter class:

```python
class BaseOpenAILLMService(LLMService):
    adapter_class = OpenAILLMAdapter  # default

class AnthropicLLMService(LLMService):
    adapter_class = AnthropicLLMAdapter
```

This means you can switch LLM providers without changing how you define tools or manage context — the adapter handles the translation.

### The one abstract method

```python
@abstractmethod
async def run_inference(self, context, max_tokens=None) -> Optional[str]:
    """Non-streaming, one-shot LLM call. Used for context summarization."""
```

The main streaming flow is handled by concrete implementations overriding `process_frame()` directly (not via an abstract method).

---

## TTSService — Text to Speech

**File**: `src/pipecat/services/tts_service.py`

Receives text from the LLM and produces audio. The key design decision here is **when to send text to the provider** — immediately per token, or after accumulating a full sentence.

### Text aggregation modes

```python
class TextAggregationMode(str, Enum):
    SENTENCE = "sentence"  # Default: buffer until sentence boundary
    TOKEN = "token"        # Stream tokens immediately
```

**SENTENCE mode** (default): Accumulates `LLMTextFrame` tokens until it detects a sentence boundary (`.?!` followed by a non-whitespace character, validated by NLTK). Then sends the complete sentence to the TTS provider.

- Pro: Better prosody (the provider sees complete sentences, can plan intonation)
- Con: ~200-300ms latency waiting for the sentence to complete

**TOKEN mode**: Sends each token immediately to the provider as it arrives from the LLM.

- Pro: Minimal latency (user hears audio almost as fast as the LLM generates tokens)
- Con: May reduce speech quality (provider can't plan ahead)

### Sentence detection — the lookahead trick

The `SimpleTextAggregator` doesn't just split on `.?!`. It uses lookahead to avoid false positives:

1. Buffer characters as they arrive
2. When `.?!` detected, enable lookahead mode
3. Wait for the next non-whitespace character to **confirm** the sentence ended
4. Validate with NLTK (`match_endofsentence()`) to handle edge cases like `"$29."` or `"Dr."`

At `LLMFullResponseEndFrame` (LLM done generating), any remaining buffered text is flushed as a final sentence.

### Frames consumed and produced

**Consumes**:
- `LLMTextFrame` / `TextFrame` — text to speak (goes through aggregation)
- `TTSSpeakFrame` — direct "speak this text" instruction (bypasses aggregation)
- `LLMFullResponseStartFrame` / `LLMFullResponseEndFrame` — marks response boundaries (triggers flush on end)
- `InterruptionFrame` — clears buffered text, resets state
- `TTSUpdateSettingsFrame` — runtime settings changes (voice, model, language)

**Produces**:
- `TTSAudioRawFrame` — PCM audio chunks (flows to output transport)
- `TTSStartedFrame` / `TTSStoppedFrame` — lifecycle markers per synthesis request
- `TTSTextFrame` — the text being spoken (for context synchronization — so the assistant aggregator knows what was actually said)

### Text transformation pipeline

Before text reaches the TTS provider, it goes through:

1. **Aggregation** (SENTENCE or TOKEN mode)
2. **Text filters** (`_text_filters`) — post-aggregation filtering
3. **Text transforms** (`_text_transforms`) — registered mutation functions (e.g., add SSML tags, Cartesia emotion markup)
4. **Final preparation** (`_prepare_text_for_tts()`) — e.g., add trailing space

Important: the **original untransformed text** is pushed to the assistant context, not the transformed version. This prevents TTS-specific markup (SSML, emotion tags) from polluting the conversation history.

### Interruption handling

When `InterruptionFrame` arrives:
- `_processing_text = False` — stop sending text
- Text aggregator buffer cleared
- All text filters reset
- LLM response state reset
- Word timestamp tracking reset (if applicable)

The TTS stops mid-sentence. The output transport (Topic 5) handles clearing any queued audio chunks.

### The one abstract method

```python
@abstractmethod
async def run_tts(self, text: str, context_id: str) -> AsyncGenerator[Frame, None]:
    """Send text to provider, yield TTSStartedFrame, TTSAudioRawFrame chunks, TTSStoppedFrame."""
```

### Word-level timestamps (advanced)

Some providers (Cartesia) return word-level timing info alongside audio. When `supports_word_timestamps=True`:
- `TTSTextFrame` is generated **dynamically** as words are spoken (synced to audio playback)
- Only spoken words are added to the assistant context (if the bot is interrupted mid-sentence, the context only includes words actually spoken)
- The `_words_task_handler()` manages a queue of `(word, timestamp)` pairs and emits `TTSTextFrame` at the right moment

### WebsocketTTSService and AudioContextTTSService

Two progressive specializations:

- **WebsocketTTSService** — adds WebSocket connection management + keepalive (same pattern as WebsocketSTTService)
- **AudioContextTTSService** — manages multiple concurrent TTS requests. Each `context_id` gets its own audio queue, and they're played in request order. This allows overlapping synthesis (start generating sentence 2 while sentence 1 is still playing).

---

## Metrics Tracking

All three service types track metrics, but the specifics differ:

### STT metrics

- **TTFB (Time-to-First-Byte)**: Time from `VADUserStoppedSpeakingFrame` to final `TranscriptionFrame`. This measures "how long after the user stops speaking do we have their words?"
- **TTFS P99 latency**: Pre-measured per-provider latency (e.g., Deepgram=0.35s, Whisper=1.0s). Broadcast via `STTMetadataFrame` at startup so turn strategies can optimize timing.

### LLM metrics

- **TTFB**: Time from sending the API request to first streaming chunk
- **Token usage**: `prompt_tokens`, `completion_tokens`, `total_tokens`, `cache_read_input_tokens` (prompt caching), `reasoning_tokens` (o1-class models)
- **Processing time**: Total wall-clock time for the API call

### TTS metrics

- **TTFB**: Time from sending text to first audio chunk received
- **Text aggregation time**: How long text sat in the aggregation buffer (only meaningful in SENTENCE mode)
- **TTS usage**: Text length per synthesis request

---

## How a New Service Provider Fits In

Adding a new provider (say, a new STT service) means:

1. **Extend the right base class** (`STTService`, `LLMService`, or `TTSService`)
2. **Implement the abstract method** (`run_stt`, `run_inference`, or `run_tts`)
3. **Define settings** — extend `ServiceSettings` with provider-specific fields
4. **Handle lifecycle** — connect to the provider on `start()`, disconnect on `stop()`
5. **Map language codes** — override `language_to_service_language()` if the provider uses non-standard codes
6. **Push errors** — call `push_error(msg, fatal=False)` on API failures (non-fatal lets the app recover)

The base class handles frame routing, metrics timing, settings delta application, and pipeline integration. The concrete service only deals with the provider's API.

---

## Reference

| File | What |
|------|------|
| `services/ai_service.py` | AIService base — settings, lifecycle, error handling |
| `services/stt_service.py` | STTService, SegmentedSTTService, WebsocketSTTService |
| `services/llm_service.py` | LLMService — function calling, context management |
| `services/tts_service.py` | TTSService — text aggregation, word timestamps |
| `adapters/base_llm_adapter.py` | BaseLLMAdapter — provider format translation |
| `services/openai/base_llm.py` | BaseOpenAILLMService — concrete LLM example |
| `services/deepgram/stt.py` | DeepgramSTTService — concrete STT example |
| `services/cartesia/tts.py` | CartesiaTTSService — concrete TTS with word timestamps |
| `utils/text/simple_text_aggregator.py` | Sentence/token aggregation logic |
| `services/stt_latency.py` | Pre-measured P99 latency per STT provider |
