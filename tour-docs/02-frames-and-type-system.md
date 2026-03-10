# Topic 2: Frames & the Type System

## Overview

Every piece of data and every signal in Pipecat is a **Frame**. Frames are Python `@dataclass` objects that flow through the pipeline. The type system determines how each frame is queued, prioritized, and whether it survives interruptions.

Source: `src/pipecat/frames/frames.py` (~2283 lines, 100+ frame types)

## The Three Pillars

```
Frame (base)
├── SystemFrame    — priority queue, immediate, survives interruptions
├── DataFrame      — ordered FIFO queue, carries data, cancelled by interruptions
└── ControlFrame   — ordered FIFO queue, control signals, cancelled by interruptions
```

### Frame (base class)
Every frame carries: `id`, `name`, `pts` (presentation timestamp), `broadcast_sibling_id`, `metadata`, `transport_source`, `transport_destination`.

### SystemFrame
- Processed via a **priority queue** — they jump ahead of regular frames
- **Not affected by interruptions** — they are never dropped
- Used for: lifecycle events, speaking/VAD signals, input from transport, task signals, errors, metrics

### DataFrame
- Processed via an **ordered FIFO queue**
- **Cancelled by interruptions** — pending DataFrames are drained from the queue when an InterruptionFrame arrives
- Used for: text, audio output, images, LLM messages, function call results

### ControlFrame
- Same FIFO queue as DataFrame
- **Also cancelled by interruptions**
- Used for: LLM response lifecycle markers, TTS start/stop, service settings updates, pause/resume

## The UninterruptibleFrame Mixin

```python
@dataclass
class UninterruptibleFrame:
    pass
```

A **marker mixin** (not a Frame subclass). When combined with DataFrame or ControlFrame via multiple inheritance, the frame survives interruptions — it won't be dropped from queues.

Key uninterruptible frames:
- `EndFrame` (ControlFrame + UninterruptibleFrame) — graceful shutdown must always complete
- `StopFrame` (ControlFrame + UninterruptibleFrame) — stop must always reach all processors
- `FunctionCallResultFrame` (DataFrame + UninterruptibleFrame) — tool results must be delivered
- `FunctionCallInProgressFrame` (ControlFrame + UninterruptibleFrame)
- `ServiceUpdateSettingsFrame` (ControlFrame + UninterruptibleFrame) — settings changes must apply
- `LLMContextSummaryResultFrame` (ControlFrame + UninterruptibleFrame)

## Direction

Defined in `FrameDirection` enum (`frame_processor.py`):
- **DOWNSTREAM** (1) — input to output, left to right in the pipeline
- **UPSTREAM** (2) — output back to input, right to left

Most data flows downstream. Errors always go upstream. Task frames (`EndTaskFrame`, `InterruptionTaskFrame`, etc.) go upstream to the PipelineTask, which then sends the corresponding control frame downstream.

## Raw Media Mixins

Audio, image, and DTMF have **pure data mixins** (not Frame subclasses) that get combined with SystemFrame or DataFrame:

```
AudioRawFrame (mixin: audio bytes, sample_rate, num_channels)
├── InputAudioRawFrame  = SystemFrame + AudioRawFrame   (input, high priority)
│   └── UserAudioRawFrame                                (+ user_id)
└── OutputAudioRawFrame = DataFrame + AudioRawFrame      (output, interruptible)
    ├── TTSAudioRawFrame                                 (+ context_id)
    └── SpeechOutputAudioRawFrame
```

This means: **input from users is high-priority and uninterruptible** (SystemFrame), while **output to users is ordered and interruptible** (DataFrame). This asymmetry is intentional — you never want to drop incoming user audio, but you do want to stop bot audio when the user interrupts.

## Frame Categories

### SystemFrame Subtypes

#### Lifecycle
| Frame | Purpose |
|-------|---------|
| `StartFrame` | Boots the pipeline; carries sample rates, interruption config, metrics flags |
| `CancelFrame` | Immediate shutdown (+ reason) |
| `InterruptionFrame` | Clears all pending frames downstream; carries optional `asyncio.Event` |
| `ErrorFrame` | Error message + optional exception; `fatal=False` by default |
| `FatalErrorFrame` | Always `fatal=True` |

#### Speaking / VAD
| Frame | Purpose |
|-------|---------|
| `UserStartedSpeakingFrame` | Turn strategy signal: user began speaking |
| `UserStoppedSpeakingFrame` | Turn strategy signal: user finished speaking |
| `VADUserStartedSpeakingFrame` | Raw VAD detection (+ `start_secs`, `timestamp`) |
| `VADUserStoppedSpeakingFrame` | Raw VAD detection (+ `stop_secs`, `timestamp`) |
| `BotStartedSpeakingFrame` | Bot audio output began |
| `BotStoppedSpeakingFrame` | Bot audio output ended |
| `UserSpeakingFrame` | Emitted continuously while user is speaking |
| `BotSpeakingFrame` | Emitted continuously while bot is speaking |
| `UserMuteStartedFrame` / `UserMuteStoppedFrame` | Mute state changes |

#### Input (Transport)
| Frame | Purpose |
|-------|---------|
| `InputAudioRawFrame` | Raw audio from transport |
| `UserAudioRawFrame` | Audio + `user_id` (multi-participant) |
| `InputImageRawFrame` | Image from transport |
| `UserImageRawFrame` | Image + `user_id`, `text`, context flags |
| `InputTextRawFrame` | Text from transport |
| `InputDTMFFrame` | Telephone keypad input |

#### Task Frames (pushed upstream)
| Frame | Purpose |
|-------|---------|
| `EndTaskFrame` | Triggers graceful `EndFrame` downstream |
| `CancelTaskFrame` | Triggers immediate `CancelFrame` downstream |
| `StopTaskFrame` | Stops task but keeps processors running |
| `InterruptionTaskFrame` | Triggers `InterruptionFrame` downstream |

#### Other System
| Frame | Purpose |
|-------|---------|
| `MetricsFrame` | Carries `List[MetricsData]` |
| `STTMuteFrame` | Mute/unmute STT |
| `STTMetadataFrame` | STT latency hints (`ttfs_p99_latency`) |
| `SpeechControlParamsFrame` | Runtime VAD/turn param updates |
| `UserIdleTimeoutUpdateFrame` | Runtime idle timeout changes |

### DataFrame Subtypes

#### Text
| Frame | Purpose |
|-------|---------|
| `TextFrame` | Base text; `text`, `skip_tts`, `append_to_context` |
| `LLMTextFrame` | Token from LLM (sets `includes_inter_frame_spaces=True`) |
| `VisionTextFrame` | Token from vision model |
| `AggregatedTextFrame` | Buffered text (sentence aggregation) |
| `TTSTextFrame` | Text heading to TTS |
| `TranscriptionFrame` | STT result; `user_id`, `timestamp`, `language`, `finalized` |
| `InterimTranscriptionFrame` | Partial STT result |
| `TranslationFrame` | Translated text |
| `LLMThoughtTextFrame` | LLM reasoning text (deliberately NOT a TextFrame — bypasses TTS) |

#### Audio / Media Output
| Frame | Purpose |
|-------|---------|
| `OutputAudioRawFrame` | Base audio output |
| `TTSAudioRawFrame` | TTS audio chunk + `context_id` |
| `SpeechOutputAudioRawFrame` | Continuous speech stream |
| `OutputImageRawFrame` | Image output |
| `URLImageRawFrame` | Image + URL |
| `AssistantImageRawFrame` | Image from assistant |
| `SpriteFrame` | Animated sprite (list of images) |

#### LLM Data
| Frame | Purpose |
|-------|---------|
| `LLMRunFrame` | Trigger LLM with current context |
| `LLMMessagesAppendFrame` | Append messages to context |
| `LLMMessagesUpdateFrame` | Replace context messages |
| `LLMSetToolsFrame` | Set available tools |
| `LLMSetToolChoiceFrame` | Set tool choice policy |
| `LLMEnablePromptCachingFrame` | Toggle prompt caching |
| `LLMConfigureOutputFrame` | Configure output (e.g. skip TTS) |
| `FunctionCallResultFrame` * | Tool result (uninterruptible) |
| `TTSSpeakFrame` | Inject arbitrary text to speak |

### ControlFrame Subtypes

#### LLM Lifecycle
| Frame | Purpose |
|-------|---------|
| `LLMFullResponseStartFrame` | Start of streaming LLM response |
| `LLMFullResponseEndFrame` | End of streaming LLM response |
| `LLMThoughtStartFrame` / `LLMThoughtEndFrame` | Thought boundaries |
| `FunctionCallInProgressFrame` * | Tool call in flight (uninterruptible) |
| `LLMSummarizeContextFrame` | Request context summarization |
| `LLMContextSummaryRequestFrame` / `ResultFrame` * | Summary lifecycle |
| `LLMAssistantPushAggregationFrame` | Force aggregator push |

#### TTS / Service
| Frame | Purpose |
|-------|---------|
| `TTSStartedFrame` / `TTSStoppedFrame` | TTS audio burst boundaries |
| `EndFrame` * | Graceful shutdown (uninterruptible) |
| `StopFrame` * | Stop task (uninterruptible) |
| `HeartbeatFrame` | Pipeline health check |
| `ServiceUpdateSettingsFrame` * | Base for runtime settings updates |
| `LLMUpdateSettingsFrame` / `TTSUpdateSettingsFrame` / `STTUpdateSettingsFrame` | Per-service |

#### Filter / Mixer / Pause
| Frame | Purpose |
|-------|---------|
| `FilterUpdateSettingsFrame` / `FilterEnableFrame` | Audio filter control |
| `MixerUpdateSettingsFrame` / `MixerEnableFrame` | Audio mixer control |
| `VADParamsUpdateFrame` | VAD parameter changes |
| `FrameProcessorPauseFrame` / `ResumeFrame` | Ordered pause/resume |

## Special Cases

### LLMContextFrame
Inherits directly from `Frame` — not `DataFrame`, not `SystemFrame`. This is the only non-deprecated concrete frame that does this. It carries the full `LLMContext`.

### Input vs Output Asymmetry
Input frames are `SystemFrame` (priority, uninterruptible). Output frames are `DataFrame` (ordered, interruptible). This ensures user input is never dropped, while bot output can be cancelled on interruption.

### Deprecated Frames
Several frames are deprecated with `DeprecationWarning` in `__post_init__`:
- `StartInterruptionFrame` → use `InterruptionFrame`
- `BotInterruptionFrame` → use `InterruptionTaskFrame`
- `EmulateUser{Started,Stopped}SpeakingFrame`
- `LLMMessagesFrame` → use `LLMMessagesAppendFrame`
- `TransportMessageFrame` → use `OutputTransportMessageFrame`
- `InputTransportMessageUrgentFrame` → use `InputTransportMessageFrame`

## Diagram

See `02-frame-type-hierarchy.excalidraw` for the visual diagram.

## Reference

| File | Purpose |
|------|---------|
| `src/pipecat/frames/frames.py` | All frame definitions |
| `src/pipecat/processors/frame_processor.py` | `FrameDirection` enum, queue handling |
| `src/pipecat/metrics/metrics.py` | `MetricsData` types (carried by `MetricsFrame`) |
