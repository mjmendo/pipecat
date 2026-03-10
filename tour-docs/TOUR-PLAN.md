# Pipecat Codebase Tour Plan

A structured walkthrough of the Pipecat framework, building understanding from fundamentals to advanced topics.

## Tour Progress

### 1. Voice Bot Pipeline — End-to-End Overview ✅
- [x] High-level architecture: Mic → Transport In → STT → User Aggregator → LLM → TTS → Transport Out → Speaker
- [x] Frame types at each stage (AudioRawFrame, TranscriptionFrame, LLMMessagesFrame, LLMTextFrame, TTSAudioRawFrame)
- [x] Interruption flow (VAD → InterruptionFrame clears queues)
- [x] PipelineRunner / PipelineTask lifecycle
- [x] Diagram: `01-voice-bot-pipeline.excalidraw`

### 2. Frames & the Type System ✅
- [x] Frame hierarchy: DataFrame, ControlFrame, SystemFrame
- [x] Downstream vs Upstream frame direction
- [x] Key frame types: Audio, Text, Control, Lifecycle (exhaustive catalog)
- [x] UninterruptibleFrame mixin and why it matters
- [x] Input vs Output asymmetry (SystemFrame vs DataFrame)
- [x] Raw media mixins (AudioRawFrame, ImageRawFrame, DTMFFrame)
- [x] Special cases: LLMContextFrame anomaly, LLMThoughtTextFrame bypassing TTS
- [x] Deprecated frames and migration paths
- [x] Diagram: `02-frame-type-hierarchy.excalidraw`
- [ ] **Follow-up: practical examples** — add use-case-driven examples showing when/why key frames are used (e.g. "when does FunctionCallResultFrame matter?", "why is LLMThoughtTextFrame not a TextFrame?")

### 3. FrameProcessor Deep Dive ✅
- [x] Dual-queue architecture (Input PriorityQueue + Process FIFO)
- [x] `process_frame()` override pattern with practical example
- [x] `push_frame()` and direction handling
- [x] Interruption mechanics: queue draining, task cancellation (with scenarios)
- [x] Direct mode (used by Pipeline source/sink)
- [x] Linking model (doubly-linked list)
- [x] Event hooks and task management
- [x] Diagram: `03-frame-processor-internals.excalidraw`

### 4. Pipeline, Task & Runner ✅
- [x] Pipeline wiring: PipelineSource → processors → PipelineSink (escape hatches)
- [x] PipelineTask: double-wrapping, StartFrame synchronization, _push_queue, terminal frames
- [x] TaskSource/TaskSink escape hatches (TaskFrame → control frame translation)
- [x] Heartbeat system, idle timeout monitor, observer fan-out (TaskObserver proxy)
- [x] PipelineRunner: signal handling (SIGINT/SIGTERM), graceful shutdown
- [x] ParallelPipeline: fan-out/fan-in, deduplication, lifecycle frame synchronization
- [x] Scenarios: startup sequence, graceful shutdown, SIGINT, idle timeout
- [x] Diagram: `04-pipeline-wiring.excalidraw`

### 5. Transports ✅
- [x] Split architecture: BaseTransport factory → input()/output() as separate FrameProcessors
- [x] BaseInputTransport: audio queue, filtering, passthrough, lifecycle
- [x] BaseOutputTransport: MediaSender, audio chunking (40ms for interruptions), resampling, bot speaking detection
- [x] TransportParams configuration
- [x] Daily WebRTC transport (transcription, SIP, recording, participant mgmt)
- [x] LiveKit WebRTC transport (retry logic, data channels, DTMF)
- [x] WebSocket transport + serializer requirement
- [x] SmallWebRTC transport (pure Python, aiortc)
- [x] Local transport (PyAudio, mic/speaker)
- [x] Serializers: FrameSerializer base, telephony μ-law pattern (Twilio et al.), protobuf
- [x] Diagram: `05-transports.excalidraw` *(pending)*
- [ ] **Follow-up (Topic 9)**: Non-audio/video use cases — UI-driven bots (pizza ordering example), RTVI as the bidirectional data channel for structured messages (button clicks, form data, UI commands), how client events reach the pipeline

### 6. AI Services — STT, LLM, TTS ✅
- [x] AIService common base (settings delta, lifecycle hooks, generator processing)
- [x] STTService vs SegmentedSTTService (streaming vs batch, finalization patterns)
- [x] WebsocketSTTService (connection management, keepalive)
- [x] LLMService: streaming tokens, function calling internals (registry, parallel/sequential, timeout)
- [x] Adapter pattern (BaseLLMAdapter → provider format translation)
- [x] TTSService: text aggregation modes (SENTENCE vs TOKEN, lookahead trick)
- [x] Text transformation pipeline (filters → transforms → prepare, untransformed text to context)
- [x] Word-level timestamps (Cartesia pattern, AudioContextTTSService)
- [x] How to add a new service provider (6-step checklist)
- [x] Metrics: TTFB per service type, token usage (LLM), TTFS P99 latency (STT)

### 7. Context & Turn Management ✅
- [x] LLMContext — shared message/tool container, universal format with LLMSpecificMessage escape hatch
- [x] LLMContextAggregatorPair — factory creating user + assistant aggregators sharing same context
- [x] LLMUserAggregator — transcription accumulation, turn boundary detection, LLMContextFrame push
- [x] LLMAssistantAggregator — text accumulation between start/end markers, function call context tracking
- [x] The aggregation cycle (user turn → context update → LLM → assistant turn → context update)
- [x] Turn start strategies: VAD, transcription, min words, external (with comparison table)
- [x] Turn stop strategies: speech timeout (with STT latency trick), turn analyzer (SmartTurn), external
- [x] VAD state machine (QUIET → STARTING → SPEAKING → STOPPING), Silero/AIC/WebRTC implementations
- [x] Default vs external vs custom strategy configurations
- [x] Turn tracking observers (TurnTrackingObserver, TurnTraceObserver with OpenTelemetry spans)
- [x] **Deep dive: speech timeout trick** — timeline walkthrough explaining VAD stop_secs subtraction from STT P99, with concrete numbers per provider

### 8. Interruptions & Concurrency ✅
- [x] Full interruption lifecycle (VAD → strategy → InterruptionTaskFrame upstream → PipelineSource converts → InterruptionFrame downstream)
- [x] `push_interruption_task_frame_and_wait()` synchronization mechanism (`_wait_for_interruption` flag, bypass path, 5s timeout)
- [x] How each processor handles interruptions (base: cancel+recreate process task; LLM: cancel function calls; TTS: reset aggregation, reconnect WebSocket; Output transport: cancel media tasks)
- [x] Queue draining: `__reset_process_queue()` preserves `UninterruptibleFrame` (EndFrame, StopFrame), discards everything else
- [x] `InterruptionFrame.complete()` contract — must be called if frame is stopped from reaching sink
- [x] InterruptionFrame vs InterruptionTaskFrame vs CancelFrame vs EndFrame comparison
- [x] TaskManager: registry, wrapped coroutines (CancelledError re-raised, Exception swallowed), done_callback auto-cleanup, safe cancellation with timeout
- [x] `create_task()` / `cancel_task()` in FrameProcessor (name prefixing, 1s default timeout)
- [x] Error handling: `push_error()` → ErrorFrame upstream → PipelineSource `on_pipeline_error` event; fatal=True → CancelFrame shutdown
- [x] Practical scenarios: mid-sentence interruption, function call interruption, muted user, fatal error

### 9. Observers & RTVI ✅
- [x] Observer base interface: `on_process_frame()` (before dispatch) / `on_push_frame()` (after push), `FrameProcessed` / `FramePushed` data classes
- [x] TaskObserver proxy: per-observer queue + asyncio task, non-blocking fan-out, pipeline never blocked
- [x] Observer attachment: `PipelineTask(observers=[...])`, runtime `add_observer()` / `remove_observer()`
- [x] All built-in observers: TurnTrackingObserver, UserBotLatencyObserver, TurnTraceObserver (OTEL spans), IdleFrameObserver, RTVIObserver
- [x] Logger observers: DebugLogObserver, LLMLogObserver, MetricsLogObserver, TranscriptionLogObserver
- [x] Observers vs event handlers (pipeline-wide read-only vs per-processor hooks)
- [x] RTVIProcessor: client → server gateway, `client-ready` handshake, `send-text`, `client-message`, `llm-function-call-result`, `disconnect-bot`, `raw-audio`
- [x] RTVIObserver: pipeline frames → RTVI messages (speaking, transcription, LLM lifecycle, TTS lifecycle, function calls, metrics, audio levels)
- [x] RTVI wire format, full message type catalog (client→server and server→client)
- [x] Transport delivery: `OutputTransportMessageUrgentFrame` bypasses audio queue; telephony serializers filter RTVI messages
- [x] RTVIObserverParams: per-category toggles, `ignored_sources`, `function_call_report_level` (security), `bot_output_transforms`
- [x] **Non-audio use cases (follow-up from Topic 5)**: `client-message` for button clicks/form data, `server-message` for UI commands, `send-text` with `audio_response=false` for text-only

### 10. Metrics — Exhaustive Walkthrough ⬜
- [ ] All supported metrics in Pipecat (TTFB per service, processing time, etc.)
- [ ] How MetricsData flows through the pipeline
- [ ] Metrics observers and logging
- [ ] Unsupported but domain-relevant metrics (e.g. word error rate, conversation success, latency percentiles, user sentiment, interruption rate, silence ratio)
- [ ] Recommendations for custom metrics instrumentation

### 11. Advanced Topics ⬜
- [ ] ParallelPipeline and fan-out patterns
- [ ] Custom FrameProcessors (filters, aggregators, audio mixers)
- [ ] Serializers for telephony (Twilio, Plivo, etc.)
- [ ] Function calling end-to-end
- [ ] Vision and multimodal pipelines

---

## Diagrams

| # | Topic | File |
|---|-------|------|
| 1 | Voice Bot Pipeline E2E | `01-voice-bot-pipeline.excalidraw` |
| 2 | Frame Type Hierarchy | `02-frame-type-hierarchy.excalidraw` |
| 3 | FrameProcessor Internals | `03-frame-processor-internals.excalidraw` |
| 4 | Pipeline Wiring | `04-pipeline-wiring.excalidraw` |
| 4b | Interruption Sequence (Example 2) + Function Calling (Example 3) | `04-interruption-sequence.excalidraw` |
| 5 | Interruption Flow | *(pending)* |

## Fork Maintenance

See [`MAINTAINING-FORK.md`](./MAINTAINING-FORK.md) for how to sync this fork with upstream and update tour-docs content.

## Key Reference Files

| File | Purpose |
|------|---------|
| `examples/foundational/06-listen-and-respond.py` | Canonical voice bot example |
| `src/pipecat/frames/frames.py` | All frame definitions |
| `src/pipecat/processors/frame_processor.py` | Base processor |
| `src/pipecat/pipeline/pipeline.py` | Pipeline chaining |
| `src/pipecat/pipeline/task.py` | PipelineTask orchestration |
| `src/pipecat/pipeline/runner.py` | PipelineRunner entry point |
| `src/pipecat/services/stt_service.py` | STT base class |
| `src/pipecat/services/llm_service.py` | LLM base class |
| `src/pipecat/services/tts_service.py` | TTS base class |
| `src/pipecat/transports/base_transport.py` | Transport abstractions |
