# Topic 7: Context & Turn Management

## What This Topic Covers

This is the conversation brain — how Pipecat tracks what's been said, decides when the user is done speaking, and triggers the LLM at the right moment. Without this layer, the pipeline would just be audio flowing through processors with no memory or conversation structure.

Two tightly coupled systems:
- **Context management**: accumulating user and assistant messages into `LLMContext`
- **Turn management**: detecting when the user starts/stops speaking (and when to interrupt the bot)

## LLMContext — The Conversation Memory

**File**: `src/pipecat/processors/aggregators/llm_context.py`

`LLMContext` is the shared state container for the entire conversation. It holds:

```python
class LLMContext:
    _messages: List[LLMContextMessage]            # Conversation history
    _tools: ToolsSchema | NotGiven                # Available function call tools
    _tool_choice: LLMContextToolChoice | NotGiven  # Tool selection strategy ("auto", "required", etc.)
```

Messages use a universal format (OpenAI-compatible `ChatCompletionMessageParam`), with an escape hatch for provider-specific formats via `LLMSpecificMessage`.

Both the user aggregator and assistant aggregator hold a reference to the **same LLMContext instance**. When either one appends a message, the other sees it immediately. This is how conversation history accumulates across turns.

---

## The Two Aggregators

**File**: `src/pipecat/processors/aggregators/llm_response_universal.py`

### LLMContextAggregatorPair — The Factory

A convenience class that creates both aggregators sharing the same context:

```python
context = LLMContext(messages=[system_prompt], tools=tools)

user_agg, assistant_agg = LLMContextAggregatorPair(
    context,
    user_params=LLMUserAggregatorParams(
        vad_analyzer=SileroVADAnalyzer(),
    ),
)

pipeline = Pipeline([
    transport.input(),
    stt,
    user_agg,          # accumulates user speech into context
    llm,
    tts,
    transport.output(),
    assistant_agg,      # accumulates bot responses into context
])
```

Both aggregators are FrameProcessors that sit in the pipeline. They intercept specific frames, accumulate text, and push context updates at the right moments.

### LLMUserAggregator — "What did the user say?"

**Position**: between STT and LLM in the pipeline.

**What it does**:
1. Receives `TranscriptionFrame` from STT — accumulates text in an internal buffer (does NOT push it downstream)
2. When the user's turn ends (detected by turn strategies) — concatenates all accumulated text into a single message, adds `{"role": "user", "content": "What's the weather in Paris?"}` to the shared `LLMContext`, and pushes an `LLMContextFrame` downstream to trigger the LLM

**Key frames consumed**:
- `TranscriptionFrame` — accumulated (consumed, not forwarded)
- `InterimTranscriptionFrame` — consumed silently
- `UserStartedSpeakingFrame` / `UserStoppedSpeakingFrame` — turn boundary signals
- `LLMRunFrame` — explicit "run the LLM now" trigger
- `LLMMessagesAppendFrame` — append messages to context programmatically
- `LLMSetToolsFrame` / `LLMSetToolChoiceFrame` — update tools at runtime

**Key frames produced**:
- `LLMContextFrame` — pushed downstream when user turn ends (this is what triggers the LLM)
- `UserStartedSpeakingFrame` / `UserStoppedSpeakingFrame` — broadcast to pipeline

**Events**:
- `on_user_turn_started` — user began speaking
- `on_user_turn_stopped` — user finished (includes the aggregated text)
- `on_user_turn_stop_timeout` — fallback if no stop strategy triggers within timeout
- `on_user_turn_idle` — user has been silent for configured duration

### LLMAssistantAggregator — "What did the bot say?"

**Position**: after transport.output() in the pipeline (it sees frames that already passed through TTS and output transport).

**What it does**:
1. Receives `LLMFullResponseStartFrame` — marks beginning of bot response
2. Receives `TextFrame` (including `LLMTextFrame`, `TTSTextFrame`) — accumulates text
3. Receives `LLMFullResponseEndFrame` — concatenates accumulated text, adds `{"role": "assistant", "content": "It's 18°C and sunny in Paris."}` to the shared `LLMContext`

**Function call handling** (recall Topic 4):
- `FunctionCallInProgressFrame` → adds `{"role": "assistant", "tool_calls": [...]}` and `{"role": "tool", "content": "IN_PROGRESS"}` to context
- `FunctionCallResultFrame` → replaces `"IN_PROGRESS"` with actual result, then pushes `LLMContextFrame` **upstream** to re-invoke the LLM

**Events**:
- `on_assistant_turn_started` — bot began responding
- `on_assistant_turn_stopped` — bot finished (includes the aggregated text)

### The aggregation cycle

```
User speaks → STT → TranscriptionFrame
                        ↓
              UserAggregator accumulates text
                        ↓
              Turn ends → push_aggregation()
                        ↓
              context.add_message({role: "user", content: "..."})
              push LLMContextFrame downstream
                        ↓
              LLM receives context, generates response
                        ↓
              LLMTextFrame tokens flow downstream
                        ↓
              TTS converts to audio → user hears bot
                        ↓
              AssistantAggregator accumulates text
                        ↓
              LLMFullResponseEndFrame → push_aggregation()
                        ↓
              context.add_message({role: "assistant", content: "..."})
                        ↓
              Context now has both messages. Ready for next turn.
```

---

## Turn Management — When Does the User Start and Stop?

Turn detection is handled by **strategies** — pluggable components that analyze frames and fire events when they detect turn boundaries.

### The strategy architecture

**Files**: `src/pipecat/turns/`

```python
@dataclass
class UserTurnStrategies:
    start: Optional[List[BaseUserTurnStartStrategy]] = None  # ANY can trigger
    stop: Optional[List[BaseUserTurnStopStrategy]] = None    # ANY can trigger
```

Multiple strategies can be active simultaneously. For start: **any one** can trigger the turn. For stop: same — first strategy to fire wins.

Strategies are configured via `LLMUserAggregatorParams` and managed by a `UserTurnController` inside the aggregator.

### Turn START strategies

| Strategy | Triggers on | Use case |
|----------|-------------|----------|
| `VADUserTurnStartStrategy` | `VADUserStartedSpeakingFrame` | Default — immediate response to voice activity |
| `TranscriptionUserTurnStartStrategy` | `TranscriptionFrame` or `InterimTranscriptionFrame` | Fallback when VAD misses soft speech |
| `MinWordsUserTurnStartStrategy` | N words detected in transcription | Prevents interruption on single words ("um") — requires `min_words` to interrupt bot |
| `ExternalUserTurnStartStrategy` | `UserStartedSpeakingFrame` from external processor | For STT/transport that handles its own VAD (Twilio, some WebRTC) |

**Default**: `[VADUserTurnStartStrategy(), TranscriptionUserTurnStartStrategy()]` — VAD is primary, transcription is fallback.

### Turn STOP strategies

| Strategy | How it decides | Use case |
|----------|---------------|----------|
| `SpeechTimeoutUserTurnStopStrategy` | Silence timeout after VAD stop + STT latency adjustment | Simple, predictable |
| `TurnAnalyzerUserTurnStopStrategy` | AI model predicts turn completion (`LocalSmartTurnAnalyzerV3`) | Smarter — detects "I want... um..." as incomplete |
| `ExternalUserTurnStopStrategy` | Waits for `UserStoppedSpeakingFrame` from external processor | For STT/transport that handles its own endpointing |

**Default**: `[TurnAnalyzerUserTurnStopStrategy(turn_analyzer=LocalSmartTurnAnalyzerV3())]` — uses an AI model to predict when the user is done.

### What happens when a turn starts

When a start strategy fires `trigger_user_turn_started()`:

1. The `UserTurnController` checks it's not already in a turn (prevents duplicates)
2. It calls the aggregator's `_on_user_turn_started()`:
   - Broadcasts `UserStartedSpeakingFrame` to the pipeline (if `enable_user_speaking_frames=True`)
   - Pushes `InterruptionFrame` to stop the bot mid-sentence (if `enable_interruptions=True`)
   - Fires `on_user_turn_started` event for application code

### What happens when a turn stops

When a stop strategy fires `trigger_user_turn_stopped()`:

1. The `UserTurnController` calls the aggregator's `_on_user_turn_stopped()`
2. Broadcasts `UserStoppedSpeakingFrame` to the pipeline
3. Calls `push_aggregation()`:
   - Concatenates all accumulated transcription text
   - Adds `{"role": "user", "content": "..."}` to `LLMContext`
   - Pushes `LLMContextFrame` downstream → LLM generates response
4. Fires `on_user_turn_stopped` event with the aggregated text

### The speech timeout trick

The `SpeechTimeoutUserTurnStopStrategy` accounts for STT latency to avoid double-counting silence that VAD already measured.

**The timeline**: Imagine the user says "What's the weather?" and stops talking:

```
t=0.0s  User says "weather?"
t=0.2s  User goes silent
t=0.4s  VAD transitions SPEAKING → STOPPING → QUIET (stop_secs=0.2s elapsed)
        → VADUserStoppedSpeakingFrame emitted
t=???   Final TranscriptionFrame arrives from Deepgram
```

At t=0.4s, the stop strategy receives `VADUserStoppedSpeakingFrame`. How long should it wait for the final transcription?

**Naive approach (too slow)**: Wait the full STT P99 latency (Deepgram's 0.35s) from t=0.4s → timer expires at t=0.75s. But Deepgram's latency is measured from when the user **actually** stopped (t=0.2s), not from when VAD confirmed it (t=0.4s). The transcription likely arrived around t=0.55s. By waiting until t=0.75s, you've added 200ms of unnecessary dead silence.

**The trick**: Subtract the time VAD already consumed:

```
remaining_wait = stt_p99_latency - vad_stop_secs = 0.35s - 0.2s = 0.15s
```

```
t=0.2s  User actually stops speaking
t=0.4s  VAD confirms silence (0.2s of stop_secs already elapsed)
t=0.55s Timer expires (0.4 + 0.15) — transcription has almost certainly arrived
```

The 0.2s that VAD spent confirming silence is time the STT provider was **also** processing. By subtracting it, you avoid waiting for time that already passed.

**The full formula**:

```
timeout = max(stt_p99_latency - vad_stop_secs, user_speech_timeout)
```

The `max()` ensures you never go below `user_speech_timeout` (default 0.6s) — a floor to give the user a chance to continue speaking (e.g., a pause between sentences).

**Concrete numbers**:

| Provider | P99 latency | vad_stop_secs | Adjusted wait | Floor (0.6s) | Actual timeout |
|----------|------------|---------------|---------------|-------------|----------------|
| Deepgram | 0.35s | 0.2s | 0.15s | 0.6s | **0.6s** (floor wins) |
| Azure | 1.80s | 0.2s | 1.60s | 0.6s | **1.6s** (latency wins) |
| Whisper | 1.00s | 0.2s | 0.80s | 0.6s | **0.8s** (latency wins) |

For fast providers like Deepgram, the floor dominates — you're mostly waiting to give the user a chance to keep talking. For slow providers like Azure, the adjusted latency dominates — you're waiting for the transcription to actually arrive.

The strategy gets the `stt_p99_latency` value from `STTMetadataFrame`, which the STT service broadcasts at pipeline start (see Topic 6).

---

## VAD — Voice Activity Detection

**Files**: `src/pipecat/audio/vad/`

VAD is the raw signal that feeds turn strategies. It analyzes audio frames and detects whether someone is speaking.

### State machine

```
QUIET ──(confidence > 0.7 for 0.2s)──→ STARTING ──(confirmed)──→ SPEAKING
  ↑                                        |                        |
  |                                   (drops early)         (confidence drops)
  |                                        |                        ↓
  └──────────────────────────────────── QUIET ←──(0.2s)──── STOPPING
                                                                    |
                                                              (rises again)
                                                                    ↓
                                                               SPEAKING
```

**Parameters** (`VADParams`):
- `confidence: 0.7` — minimum voice confidence threshold
- `start_secs: 0.2` — time voice must persist before confirming speech start
- `stop_secs: 0.2` — time silence must persist before confirming speech end
- `min_volume: 0.6` — minimum audio volume threshold

Lower `start_secs` = faster response but more false positives. Higher `stop_secs` = prevents flutter but delays turn end detection.

### VAD implementations

- **SileroVADAnalyzer** — ONNX model, runs locally, supports 8kHz/16kHz. Model state auto-resets every 5s to prevent memory growth. This is the standard choice.
- **AICVADAnalyzer** — integrates with AIC SDK's built-in VAD. Provider-managed, lazy-binding pattern.
- **WebRTCVADAnalyzer** — Daily's native WebRTC VAD (only available with Daily transport).

### VADController

Wraps a `VADAnalyzer` and emits events on state transitions:
- `on_speech_started` — transition to SPEAKING state
- `on_speech_stopped` — transition to QUIET state
- `on_speech_activity` — periodic signal while speaking

The controller is created internally by the `LLMUserAggregator` when you pass `vad_analyzer` in params. It processes every `InputAudioRawFrame` flowing through the aggregator.

---

## Putting It All Together

### Default configuration

If you just do:
```python
user_agg, assistant_agg = LLMContextAggregatorPair(
    context,
    user_params=LLMUserAggregatorParams(vad_analyzer=SileroVADAnalyzer()),
)
```

You get:
- **VAD**: Silero, running on every audio frame in the aggregator
- **Turn start**: VAD-based (primary) + transcription-based (fallback)
- **Turn stop**: AI turn analyzer (SmartTurn v3) — predicts when user is done
- **Interruptions**: enabled — bot stops talking when user starts
- **Turn stop timeout**: 5.0s fallback if no stop strategy triggers

### External control (telephony)

For Twilio/telephony where the STT provider handles endpointing:
```python
from pipecat.turns import ExternalUserTurnStrategies

user_agg, assistant_agg = LLMContextAggregatorPair(
    context,
    user_params=LLMUserAggregatorParams(
        user_turn_strategies=ExternalUserTurnStrategies(),
    ),
)
```

This disables Pipecat's own VAD and turn detection — relies on the STT/transport to emit `UserStartedSpeakingFrame` / `UserStoppedSpeakingFrame`.

### Custom strategy example

Push-to-talk (button press starts turn, release stops):
```python
class ButtonPressStartStrategy(BaseUserTurnStartStrategy):
    async def process_frame(self, frame):
        if isinstance(frame, ButtonPressedFrame):
            await self.trigger_user_turn_started()

class ButtonReleaseStopStrategy(BaseUserTurnStopStrategy):
    async def process_frame(self, frame):
        if isinstance(frame, ButtonReleasedFrame):
            await self.trigger_user_turn_stopped()

strategies = UserTurnStrategies(
    start=[ButtonPressStartStrategy(enable_interruptions=False)],
    stop=[ButtonReleaseStopStrategy()],
)
```

---

## Turn Tracking Observers

**File**: `src/pipecat/observers/turn_tracking_observer.py`

`TurnTrackingObserver` monitors conversation turns without modifying the pipeline:

- Tracks turn count, duration, and interruption status
- A "turn" starts when the user speaks and ends after the bot finishes responding
- Events: `on_turn_started(turn_number)`, `on_turn_ended(turn_number, duration, was_interrupted)`

**File**: `src/pipecat/utils/tracing/turn_trace_observer.py`

`TurnTraceObserver` creates OpenTelemetry spans for each turn — hierarchical tracing:

```
Conversation Span
  ├── Turn 1 Span (attributes: turn.number, turn.duration, turn.was_interrupted)
  │   ├── STT Service Span
  │   ├── LLM Service Span
  │   └── TTS Service Span
  ├── Turn 2 Span
  └── ...
```

---

## Reference

| File | What |
|------|------|
| `processors/aggregators/llm_context.py` | LLMContext — message and tool storage |
| `processors/aggregators/llm_response_universal.py` | LLMUserAggregator, LLMAssistantAggregator, LLMContextAggregatorPair |
| `turns/user_turn_controller.py` | UserTurnController — coordinates strategies |
| `turns/user_turn_processor.py` | UserTurnProcessor — FrameProcessor wrapper |
| `turns/user_turn_strategies.py` | UserTurnStrategies, ExternalUserTurnStrategies (defaults) |
| `turns/user_start/` | All turn start strategies (VAD, transcription, min words, external) |
| `turns/user_stop/` | All turn stop strategies (speech timeout, turn analyzer, external) |
| `audio/vad/vad_analyzer.py` | VADAnalyzer base, VADState, VADParams |
| `audio/vad/vad_controller.py` | VADController — wraps analyzer, emits events |
| `audio/vad/silero.py` | SileroVADAnalyzer — standard ONNX-based VAD |
| `observers/turn_tracking_observer.py` | TurnTrackingObserver — turn metrics |
| `utils/tracing/turn_trace_observer.py` | TurnTraceObserver — OpenTelemetry spans |
