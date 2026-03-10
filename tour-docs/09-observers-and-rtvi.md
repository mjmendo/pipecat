# Topic 9: Observers & RTVI

## What This Topic Covers

Observers are the read-only monitoring layer — they see every frame flowing between every processor in the pipeline without modifying anything. RTVI (Real-Time Voice Interface) is the protocol that bridges browser/mobile clients and the Pipecat pipeline, built on top of the observer pattern.

This topic also covers the non-audio use cases we flagged in Topic 5: how button clicks, text input, and UI commands reach the pipeline and how the pipeline sends structured messages back to the client.

---

## The Observer Pattern

### Base interface

**File**: `src/pipecat/observers/base_observer.py`

```python
class BaseObserver(BaseObject):
    async def on_process_frame(self, data: FrameProcessed):
        pass  # called BEFORE a processor dispatches a frame

    async def on_push_frame(self, data: FramePushed):
        pass  # called when a processor pushes a frame to the next processor
```

Two data classes carry context:

```python
@dataclass
class FrameProcessed:
    processor: FrameProcessor   # the processor about to handle the frame
    frame: Frame
    direction: FrameDirection
    timestamp: int              # pipeline clock (nanoseconds)

@dataclass
class FramePushed:
    source: FrameProcessor      # processor pushing
    destination: FrameProcessor # processor receiving
    frame: Frame
    direction: FrameDirection
    timestamp: int
```

**Key distinction**: `on_process_frame` fires when a processor is *about* to handle a frame. `on_push_frame` fires when a processor is *done* and handing the frame to the next processor. Most built-in observers only implement `on_push_frame`.

### Attaching observers to PipelineTask

Observers are passed to `PipelineTask`:

```python
task = PipelineTask(
    pipeline,
    observers=[MetricsLogObserver(), TranscriptionLogObserver(), my_custom_observer],
)
```

You can also add/remove at runtime:

```python
task.add_observer(my_observer)
await task.remove_observer(my_observer)
```

### The TaskObserver proxy

**File**: `src/pipecat/pipeline/task_observer.py`

Every `FrameProcessor` receives a single `_observer` reference. This is a `TaskObserver` — a proxy that fans out to all registered observers without blocking the pipeline.

Each registered observer gets its own `asyncio.Queue` + `asyncio.Task`:

```
FrameProcessor calls self._observer.on_push_frame(data)
    ↓
TaskObserver enqueues data into each observer's queue (instant, non-blocking)
    ↓
Each observer's background task dequeues and calls observer.on_push_frame(data)
```

This means:
- The pipeline is never blocked by a slow observer
- Slow observers only grow their own queue — they don't affect each other
- Frame ordering within a single observer is preserved

### Where callbacks are invoked

Two call sites in `FrameProcessor`:

1. **`process_frame()`** — at the top, before any frame dispatch:
   ```python
   await self._observer.on_process_frame(FrameProcessed(processor=self, frame=frame, ...))
   ```

2. **`__internal_push_frame()`** — on every `push_frame()` call:
   ```python
   await self._observer.on_push_frame(FramePushed(source=self, destination=self._next, ...))
   ```

Both return in microseconds (just queue insertion).

### Observers vs Event Handlers

| | Observers | Event Handlers |
|---|---|---|
| **Scope** | Pipeline-wide — see ALL frames between ALL processors | Per-processor — see only one processor's events |
| **Registration** | `PipelineTask(observers=[...])` | `@processor.event_handler("on_before_push_frame")` |
| **Execution** | Separate asyncio task per observer (never blocks pipeline) | Inline (sync) or background task (async) |
| **Modification** | Read-only | Can trigger side effects on the specific processor |
| **Use case** | Cross-cutting: logging, tracing, analytics, RTVI | Fine-grained hooks: react to a specific LLM's errors |

Observers also inherit `BaseObject`, so they can expose their own named events. `TurnTrackingObserver` uses this to emit `on_turn_started` / `on_turn_ended`.

---

## Built-In Observers

### Auto-added by PipelineTask

These are automatically instantiated based on feature flags:

| Observer | Added when | What it does |
|----------|-----------|--------------|
| `TurnTrackingObserver` | `enable_turn_tracking=True` (default) | Tracks turn count, duration, interruption status. Fires `on_turn_started` / `on_turn_ended` events. |
| `UserBotLatencyObserver` | Tracing enabled | Measures time from user-stopped-speaking to bot-started-speaking. Fires `on_latency_measured`. |
| `TurnTraceObserver` | Turn tracking + tracing | Creates OpenTelemetry spans per conversation and per turn. Subscribes to TurnTrackingObserver and UserBotLatencyObserver events. |
| `RTVIObserver` | `enable_rtvi=True` (default) | Converts pipeline frames to RTVI protocol messages for the client. |
| `IdleFrameObserver` | `idle_timeout_secs` set (default 300s) | Watches for activity frames (BotSpeaking, UserSpeaking). Signals the idle monitor. |

### Logger observers (user-added)

| Observer | File | What it logs |
|----------|------|-------------|
| `DebugLogObserver` | `observers/loggers/debug_log_observer.py` | Configurable frame types with field values. Accepts type filters and source/destination filters. |
| `LLMLogObserver` | `observers/loggers/llm_log_observer.py` | Frames to/from LLMService: context, text chunks, function calls. |
| `MetricsLogObserver` | `observers/loggers/metrics_log_observer.py` | MetricsFrame data: TTFB, processing time, token usage, TTS usage. Deduplicates across processor hops. |
| `TranscriptionLogObserver` | `observers/loggers/transcription_log_observer.py` | TranscriptionFrame and InterimTranscriptionFrame from STT services. |

### TurnTrackingObserver details

**File**: `src/pipecat/observers/turn_tracking_observer.py`

Watches `on_push_frame` for: `UserStartedSpeakingFrame`, `BotStartedSpeakingFrame`, `BotStoppedSpeakingFrame`.

A "turn" starts when the user speaks and ends 2.5s after the bot stops speaking (timer-based, to handle multi-sentence responses where the bot briefly pauses between sentences).

Uses a `deque` of frame IDs (max 100) to skip duplicate pushes of the same frame through multiple processor hops.

Exposes events:
- `on_turn_started(observer, turn_count)`
- `on_turn_ended(observer, turn_count, duration_secs, was_interrupted)`

### TurnTraceObserver details

**File**: `src/pipecat/utils/tracing/turn_trace_observer.py`

Creates OpenTelemetry span hierarchy:

```
Conversation Span
  ├── Turn 1 Span (attributes: turn.number, turn.duration, turn.was_interrupted, turn.latency)
  │   ├── STT Service Span
  │   ├── LLM Service Span
  │   └── TTS Service Span
  ├── Turn 2 Span
  └── ...
```

Subscribes to `TurnTrackingObserver.on_turn_started/ended` and `UserBotLatencyObserver.on_latency_measured` via the `@event_handler` decorator.

Exposes `get_current_turn_context()` so AI services can attach child spans to the current turn.

---

## RTVI — Real-Time Voice Interface Protocol

### Overview

RTVI is a bidirectional protocol (version `1.2.0`) that enables client applications (browsers, mobile apps) to interact with the Pipecat pipeline. It uses JSON messages over the transport's data channel (WebRTC data channel, WebSocket text messages).

Two Pipecat classes implement it:
- **RTVIProcessor** — a `FrameProcessor` at the head of the pipeline. Handles client → server messages.
- **RTVIObserver** — a `BaseObserver` watching all frame pushes. Handles server → client messages.

**File**: `src/pipecat/processors/frameworks/rtvi.py`

### Wire format

All messages share this structure:

```json
{
  "label": "rtvi-ai",
  "type": "<message-type>",
  "id": "<message-id>",
  "data": { ... }
}
```

### RTVIProcessor — Client → Server

`RTVIProcessor` is automatically prepended to the pipeline by `PipelineTask` (when `enable_rtvi=True`, the default).

It intercepts `InputTransportMessageFrame` (raw JSON from the transport), validates the `"rtvi-ai"` label, and dispatches to handlers.

**Client message types and what they produce**:

| Client message | Pipeline effect |
|----------------|----------------|
| `client-ready` | Starts audio input, triggers `on_client_ready` event, sends `bot-ready` response |
| `send-text` | `LLMMessagesAppendFrame` (optionally interrupts bot, toggles TTS via `LLMConfigureOutputFrame`) |
| `client-message` | `RTVIClientMessageFrame` downstream + `on_client_message` event |
| `llm-function-call-result` | `FunctionCallResultFrame` downstream |
| `disconnect-bot` | `EndTaskFrame` upstream (shuts down pipeline) |
| `raw-audio` / `raw-audio-batch` | `InputAudioRawFrame` to input transport |

**The handshake**:

1. Pipeline starts → `RTVIProcessor` receives `StartFrame`
2. Client sends `client-ready` (with version and client library info)
3. `RTVIProcessor` starts audio input on the transport
4. `PipelineTask`'s auto-registered `on_client_ready` handler calls `set_bot_ready()`
5. `RTVIProcessor` sends `bot-ready` to client (with protocol version and pipeline info)

### RTVIObserver — Server → Client

Watches every `push_frame()` in the pipeline and converts events to RTVI messages:

| Pipeline frame | RTVI message | Toggle |
|---------------|-------------|--------|
| `UserStartedSpeakingFrame` | `user-started-speaking` | `user_speaking_enabled` |
| `UserStoppedSpeakingFrame` | `user-stopped-speaking` | `user_speaking_enabled` |
| `TranscriptionFrame` | `user-transcription` (final=true) | `user_transcription_enabled` |
| `InterimTranscriptionFrame` | `user-transcription` (final=false) | `user_transcription_enabled` |
| `LLMContextFrame` | `user-llm-text` (last user message) | `user_llm_enabled` |
| `BotStartedSpeakingFrame` | `bot-started-speaking` | `bot_speaking_enabled` |
| `BotStoppedSpeakingFrame` | `bot-stopped-speaking` | `bot_speaking_enabled` |
| `LLMFullResponseStartFrame` | `bot-llm-started` | `bot_llm_enabled` |
| `LLMFullResponseEndFrame` | `bot-llm-stopped` | `bot_llm_enabled` |
| `LLMTextFrame` | `bot-llm-text` | `bot_llm_enabled` |
| `TTSStartedFrame` | `bot-tts-started` | `bot_tts_enabled` |
| `TTSStoppedFrame` | `bot-tts-stopped` | `bot_tts_enabled` |
| `TTSTextFrame` | `bot-tts-text` + `bot-output` (spoken=true) | `bot_tts_enabled` / `bot_output_enabled` |
| `FunctionCallsStartedFrame` | `llm-function-call-started` | `function_call_report_level` |
| `FunctionCallInProgressFrame` | `llm-function-call-in-progress` | `function_call_report_level` |
| `FunctionCallResultFrame` | `llm-function-call-stopped` | `function_call_report_level` |
| `MetricsFrame` | `metrics` | `metrics_enabled` |
| `InputAudioRawFrame` | `user-audio-level` | `user_audio_level_enabled` |
| `TTSAudioRawFrame` | `bot-audio-level` | `bot_audio_level_enabled` |
| `RTVIServerMessageFrame` | `server-message` | always |
| `RTVIServerResponseFrame` | `server-response` or `error-response` | always |

**Deduplication**: The observer maintains a `_frames_seen` set of frame IDs. Each frame is processed only once (important since broadcast frames appear at multiple processor boundaries).

**Configuration** via `RTVIObserverParams`:
- Toggle each message category on/off
- `ignored_sources`: skip frames from specific processors (e.g., a parallel evaluator LLM)
- `function_call_report_level`: per-function control over what's exposed (DISABLED / NONE / NAME / FULL) — important for security
- `audio_level_period_secs`: throttle period for audio level messages (default 0.15s)
- `bot_output_transforms`: transform text before sending

### Transport delivery

RTVI messages go through the transport's data channel:

```
RTVIObserver → RTVIProcessor.push_transport_message()
  → OutputTransportMessageUrgentFrame  (bypasses audio queue — delivered immediately)
    → Transport.send_message()
      → Daily: app message to participants
      → WebSocket: serialized JSON
      → LiveKit: data channel
```

**Serializer filtering**: Telephony serializers (`TwilioFrameSerializer`, etc.) have `ignore_rtvi_messages=True` by default — RTVI messages don't leak to phone lines.

---

## Non-Audio Use Cases — How UI Events Reach the Pipeline

(Follow-up from Topic 5)

### The question: "If the user clicks a button, how does that reach Pipecat?"

The answer is `client-message` — a generic bidirectional channel in the RTVI protocol.

**Client → Server (button click)**:

```javascript
// Client SDK
rtviClient.sendMessage({
  type: "pizza-topping-selected",
  data: { topping: "pepperoni" }
});
```

This sends:
```json
{"label": "rtvi-ai", "type": "client-message", "id": "abc123", "data": {"t": "pizza-topping-selected", "d": {"topping": "pepperoni"}}}
```

**RTVIProcessor** receives it, pushes `RTVIClientMessageFrame` downstream, and fires `on_client_message`.

**Pipeline handler** (a custom processor or event handler):

```python
@rtvi.event_handler("on_client_message")
async def handle_client_message(rtvi, msg: RTVIClientMessage):
    if msg.type == "pizza-topping-selected":
        topping = msg.data["topping"]
        # Update order state, push LLMMessagesAppendFrame, etc.
        await rtvi.send_server_response(msg, {"status": "ok", "topping": topping})
```

**Server → Client (UI command)**:

Any processor can push an `RTVIServerMessageFrame`:

```python
await self.push_frame(RTVIServerMessageFrame(data={
    "type": "show-order-summary",
    "items": [{"name": "Pepperoni Pizza", "price": 12.99}]
}))
```

The `RTVIObserver` sees it and sends `{"type": "server-message", "data": {...}}` to the client.

### Why this works for non-audio bots

The pipeline is frame-agnostic. RTVI provides:
- **Structured input**: `client-message` for button clicks, form submissions, option selections
- **Structured output**: `server-message` for UI commands, state updates, display data
- **Text input**: `send-text` for typed messages (with optional `audio_response=false` to skip TTS)
- **Function calls as UI triggers**: LLM function calls can return UI instructions that the client renders

The transport's data channel carries both audio and structured messages in parallel. The client SDK handles routing.

---

## Reference

| File | What |
|------|------|
| `observers/base_observer.py` | `BaseObserver`, `FrameProcessed`, `FramePushed` |
| `pipeline/task_observer.py` | `TaskObserver` — proxy that fans out to all observers via per-observer queues |
| `pipeline/task.py` | Observer auto-wiring, `add_observer()`, `remove_observer()` |
| `observers/turn_tracking_observer.py` | Turn count/duration/interruption tracking, `on_turn_started`/`on_turn_ended` events |
| `observers/user_bot_latency_observer.py` | User-to-bot latency measurement, `on_latency_measured` event |
| `utils/tracing/turn_trace_observer.py` | OpenTelemetry span hierarchy per conversation/turn |
| `observers/loggers/debug_log_observer.py` | Configurable frame logging with type/source filters |
| `observers/loggers/llm_log_observer.py` | LLM-specific frame logging |
| `observers/loggers/metrics_log_observer.py` | Metrics frame logging with deduplication |
| `observers/loggers/transcription_log_observer.py` | STT transcription logging |
| `processors/frameworks/rtvi.py` | `RTVIProcessor`, `RTVIObserver`, full RTVI protocol implementation |
