# Topic 8: Interruptions & Concurrency

## What This Topic Covers

This is the mechanism that makes voice conversations feel natural — when the user starts talking, the bot stops mid-sentence. It sounds simple, but the implementation involves a carefully orchestrated chain: VAD detection → turn strategy → upstream TaskFrame → downstream InterruptionFrame → queue draining across every processor → synchronization event.

We also cover the `TaskManager` system that underpins all async work in Pipecat, and the `push_error()` mechanism for propagating failures upstream.

---

## The Full Interruption Lifecycle

### Step 1: VAD detects speech

Audio flows through `BaseInputTransport._audio_task_handler()`. The VAD analyzer processes every audio chunk. When confidence exceeds the threshold for `start_secs` duration, VAD transitions to SPEAKING and broadcasts `VADUserStartedSpeakingFrame`.

### Step 2: Turn start strategy fires

The `LLMUserAggregator` holds a `UserTurnController`, which routes frames to all registered start strategies. The default `VADUserTurnStartStrategy` reacts to `VADUserStartedSpeakingFrame`:

```python
# vad_user_turn_start_strategy.py
async def process_frame(self, frame):
    if isinstance(frame, VADUserStartedSpeakingFrame):
        await self.trigger_user_turn_started()
```

`trigger_user_turn_started()` emits an event with `UserTurnStartedParams(enable_interruptions=True)`.

### Step 3: Aggregator requests interruption

The `LLMUserAggregator._on_user_turn_started()` handler fires:

```python
async def _on_user_turn_started(self, controller, strategy, params):
    if params.enable_user_speaking_frames:
        await self.broadcast_frame(UserStartedSpeakingFrame())
    if params.enable_interruptions and self._allow_interruptions:
        await self.push_interruption_task_frame_and_wait()  # BLOCKS HERE
    await self._call_event_handler("on_user_turn_started", strategy)
```

### Step 4: `push_interruption_task_frame_and_wait()`

This is the synchronization mechanism. Defined in `FrameProcessor`:

```python
async def push_interruption_task_frame_and_wait(self, *, timeout=5.0):
    self._wait_for_interruption = True
    event = asyncio.Event()
    await self.push_frame(InterruptionTaskFrame(event=event), FrameDirection.UPSTREAM)

    while not event.is_set():
        try:
            await asyncio.wait_for(event.wait(), timeout=timeout)
        except asyncio.TimeoutError:
            logger.warning("InterruptionFrame has not completed after 5s...")
            event.set()  # force unblock to avoid deadlock

    self._wait_for_interruption = False
```

The `_wait_for_interruption` flag is critical. While this method is suspended awaiting `event.wait()`, the `InterruptionFrame` will come back downstream through this same processor. Without the flag, the base class would try to cancel the process task (its default interruption behavior), causing a deadlock. The flag tells `_start_interruption()` to drain the queue instead of canceling.

There's also a bypass path: when `_wait_for_interruption` is set, an incoming `InterruptionFrame` skips the input queue entirely and is processed immediately — preventing another deadlock scenario where the input task is blocked.

### Step 5: TaskFrame → InterruptionFrame conversion

The `InterruptionTaskFrame` travels upstream to `PipelineSource`, which delegates to `PipelineTask._source_push_frame()`:

```python
elif isinstance(frame, InterruptionTaskFrame):
    # Bypass the push queue — directly inject into the pipeline.
    await self._pipeline.queue_frame(InterruptionFrame(event=frame.event))
```

The bypass is intentional: the push queue might be blocked waiting for an `EndFrame` to traverse. Direct injection avoids this.

The `InterruptionFrame` now flows **downstream** through the entire pipeline, carrying the same `asyncio.Event`.

### Step 6: Every processor handles the interruption

The `InterruptionFrame` is a `SystemFrame`, so it gets **high priority** in every processor's input queue (processed before any buffered audio, text, or data frames).

In every `FrameProcessor`:

```python
elif isinstance(frame, InterruptionFrame):
    await self._start_interruption()
    await self.stop_all_metrics()
```

`_start_interruption()` has three behaviors:

| Condition | Action |
|-----------|--------|
| `_wait_for_interruption` is True | Drain process queue only (keep `UninterruptibleFrame` items). Don't cancel the task. |
| Currently processing an `UninterruptibleFrame` | Drain process queue only. Don't cancel the task — it must finish. |
| Normal case | **Cancel** the process task (1s timeout), then **recreate** it fresh. |

The cancel+recreate is how in-flight work (LLM streaming, TTS synthesis) gets stopped mid-stream.

### Step 7: Queue draining

When `_start_interruption()` drains the process queue, it preserves `UninterruptibleFrame` instances (like `EndFrame`, `StopFrame`) and discards everything else:

```python
def __reset_process_queue(self):
    new_queue = asyncio.Queue()
    while not self.__process_queue.empty():
        item = self.__process_queue.get_nowait()
        frame = item[0]
        if isinstance(frame, UninterruptibleFrame):
            new_queue.put_nowait(item)  # keep
        self.__process_queue.task_done()
    # put back only UninterruptibleFrames
    while not new_queue.empty():
        self.__process_queue.put_nowait(new_queue.get_nowait())
```

This ensures lifecycle frames survive interruptions while all buffered audio/text/data is discarded.

### Step 8: Service-specific handling

**LLM Service** — cancels function calls with `cancel_on_interruption=True` (the default):

```python
async def _handle_interruptions(self, _):
    for function_name, entry in self._functions.items():
        if entry.cancel_on_interruption:
            await self._cancel_function_call(function_name)
```

**TTS Service** — resets all text aggregation state:

```python
async def _handle_interruption(self, frame, direction):
    self._processing_text = False
    await self._text_aggregator.handle_interruption()
    for filter in self._text_filters:
        await filter.handle_interruption()
    self._llm_response_started = False
    self._streamed_text = ""
```

For `InterruptibleTTSService` (WebSocket-based TTS without word timestamps), it goes further — disconnects and reconnects the WebSocket to cancel in-flight audio.

For `AudioContextTTSService`, it stops the audio context task and resets the active context.

**Output Transport** — cancels all media sending tasks:

```python
async def handle_interruptions(self, _):
    if not self._transport._allow_interruptions:
        return
    await self._cancel_audio_task()
    await self._cancel_clock_task()
    await self._cancel_video_task()
    self._create_video_task()
    self._create_clock_task()
    self._create_audio_task()
    await self._bot_stopped_speaking()
```

All buffered audio chunks in the audio queue are abandoned. A `BotStoppedSpeakingFrame` is emitted if the bot was actively speaking.

### Step 9: Sink completes the event

When `InterruptionFrame` reaches `PipelineSink`:

```python
elif isinstance(frame, InterruptionFrame):
    frame.complete()
```

`complete()` sets `frame.event`, which unblocks `push_interruption_task_frame_and_wait()` back in the aggregator. The aggregator resumes, fires `on_user_turn_started`, and begins accumulating the user's new speech.

### The `complete()` contract

If any processor consumes an `InterruptionFrame` without pushing it downstream (preventing it from reaching the sink), it **must** call `frame.complete()` manually. Otherwise, `push_interruption_task_frame_and_wait()` hangs until the 5-second timeout.

Real-world example — the mute handler in `LLMUserAggregator`:

```python
if should_mute_frame and isinstance(frame, InterruptionFrame):
    frame.complete()  # frame won't reach sink, so we must signal completion
```

---

## The Full Flow (Summary)

```
[Audio Input] → VAD detects speech
    ↓
VADUserStartedSpeakingFrame (broadcast)
    ↓
LLMUserAggregator._on_user_turn_started()
    ↓
push_interruption_task_frame_and_wait()  ←── BLOCKS, sets _wait_for_interruption
    ↓ (upstream)
InterruptionTaskFrame(event) → PipelineSource
    ↓
PipelineSource converts → InterruptionFrame(event) injected DOWNSTREAM
    ↓
Every FrameProcessor:
  • HIGH_PRIORITY in input queue (processed before buffered data)
  • _start_interruption(): cancel process task + drain queue
    ↓
LLMService: cancel function calls
TTSService: reset text aggregation, reconnect WebSocket
OutputTransport: cancel audio/video tasks, emit BotStoppedSpeaking
    ↓
PipelineSink: frame.complete() → sets event
    ↓
push_interruption_task_frame_and_wait() UNBLOCKS
    ↓
Aggregator continues: on_user_turn_started fires, new turn begins
```

---

## TaskManager — Async Task Lifecycle

**File**: `src/pipecat/utils/asyncio/task_manager.py`

All async work in Pipecat goes through `TaskManager`. A single instance is created by `PipelineTask` and shared with every `FrameProcessor` in the pipeline via `FrameProcessorSetup`.

### Registry

```python
class TaskManager:
    _tasks: Dict[str, TaskData]  # keyed by task name
```

Tasks are auto-removed via a `done_callback` when they finish (normal completion, cancellation, or exception). No explicit cleanup needed.

### `create_task(coroutine, name)` — wrapping and error handling

Every coroutine is wrapped in a `run_coroutine()` that:
- Catches `CancelledError` → logs, then **re-raises** (so asyncio cancellation propagates correctly)
- Catches `Exception` → logs with traceback, but does **not** re-raise (prevents one task crash from killing the event loop)

The task is added to the registry and gets a `done_callback` for auto-removal.

### `cancel_task(task, timeout)` — safe cancellation

```python
async def cancel_task(self, task, timeout=None):
    task.cancel()
    try:
        if timeout:
            await asyncio.wait_for(task, timeout=timeout)
        else:
            await task
    except asyncio.TimeoutError:
        logger.warning("timed out waiting for task to cancel")
    except asyncio.CancelledError:
        pass  # expected
```

The timeout guard is essential: some third-party libraries (SDK clients, etc.) swallow `CancelledError`, which would cause `await task` to hang forever.

### FrameProcessor integration

`FrameProcessor.create_task()` auto-prefixes task names with the processor identity (`"ProcessorName::coroutine_name"`), making debug output clear. `cancel_task()` defaults to a 1-second timeout for safety.

Each processor has two internal managed tasks:

| Task | Purpose |
|------|---------|
| `__input_frame_task` | Drains the priority input queue. Routes `SystemFrame` immediately, defers non-system frames to process queue. |
| `__process_frame_task` | Drains the process queue (FIFO). This is the task that gets cancelled on interruption. |

### Dangling task detection

At pipeline shutdown, `PipelineTask._print_dangling_tasks()` checks if any tasks are still registered. This catches leaked tasks that weren't properly cancelled — a common bug when writing custom processors.

---

## Error Handling — `push_error()`

### The mechanism

```python
async def push_error(self, error_msg, exception=None, fatal=False):
    error_frame = ErrorFrame(error=error_msg, fatal=fatal, exception=exception, processor=self)
    await self.push_error_frame(error_frame)

async def push_error_frame(self, error):
    await self._call_event_handler("on_error", error)  # local handler (synchronous)
    logger.error(error_message)
    await self.push_frame(error, FrameDirection.UPSTREAM)
```

Errors always propagate **upstream** toward `PipelineSource`.

### Where errors are auto-caught

The base `FrameProcessor` catches exceptions in three places and calls `push_error()`:
- `__process_frame()` — any exception during `process_frame()`
- `_start_interruption()` — any exception during interruption handling
- `__internal_push_frame()` — any exception during frame routing

AI services also catch errors from generators via `process_generator()` — if a streaming response yields an `ErrorFrame`, it's routed upstream instead of downstream.

### Terminal handling in PipelineTask

`_source_push_frame()` receives `ErrorFrame` at the pipeline's upstream boundary:

```python
elif isinstance(frame, ErrorFrame):
    await self._call_event_handler("on_pipeline_error", frame)
    if frame.fatal:
        await self.queue_frame(CancelFrame())  # triggers full pipeline shutdown
    else:
        logger.warning(f"Something went wrong: {frame}")
```

- **Non-fatal** (default): warning logged, pipeline continues. Application code can listen to `on_pipeline_error` to react (e.g., switch to a backup service).
- **Fatal**: a `CancelFrame` is queued downstream, which triggers orderly shutdown of all processors. Each processor receiving `CancelFrame` sets `_cancelling = True`, which makes `queue_frame()` a no-op — preventing any further frames from entering.

---

## InterruptionFrame vs TaskFrame vs CancelFrame

| Frame | Direction | Purpose |
|-------|-----------|---------|
| `InterruptionTaskFrame` | Upstream (to PipelineSource) | Request: "please interrupt the pipeline" — carries an `asyncio.Event` |
| `InterruptionFrame` | Downstream (through pipeline) | Command: "interrupt now" — drains queues, cancels tasks, carries the same event |
| `CancelFrame` | Downstream (through pipeline) | Command: "shut down the pipeline" — sets `_cancelling`, stops all frame processing |
| `EndFrame` | Downstream | Graceful shutdown — `UninterruptibleFrame`, survives queue drains |
| `StopFrame` | Downstream | Immediate stop — `UninterruptibleFrame`, survives queue drains |

`InterruptionTaskFrame` is a `TaskFrame` — it never reaches user processors. `PipelineSource` intercepts and converts it. This two-frame pattern (request upstream → command downstream) ensures the interruption traverses the entire pipeline from source to sink.

---

## UninterruptibleFrame — Surviving Interruptions

Some frames must never be discarded, even during interruptions:

```python
class UninterruptibleFrame(DataFrame):
    """Mixin that prevents this frame from being removed during queue draining."""
```

`EndFrame` and `StopFrame` both extend `UninterruptibleFrame`. During `__reset_process_queue()`, these are preserved while all other queued frames are discarded. This guarantees graceful shutdown can complete even if interruptions are happening.

---

## Practical Scenarios

### Scenario 1: User interrupts bot mid-sentence

```
Bot is saying: "The weather in Paris is currently 18 degrees and—"
User starts talking: "Actually, what about London?"

1. VAD detects user speech → VADUserStartedSpeakingFrame
2. VADUserTurnStartStrategy fires → trigger_user_turn_started()
3. Aggregator calls push_interruption_task_frame_and_wait()
4. InterruptionTaskFrame → PipelineSource → InterruptionFrame downstream
5. LLM: cancels any in-flight function calls
6. TTS: stops text aggregation, clears buffer ("and sunny" never synthesized)
7. Output transport: cancels audio task (discards queued "degrees and—" audio chunks)
8. Sink: frame.complete() → aggregator unblocks
9. Aggregator accumulates "Actually, what about London?"
10. Turn ends → LLMContextFrame → LLM generates new response about London
```

### Scenario 2: Interruption during function call

```
Bot is calling get_weather("Paris") with cancel_on_interruption=True

1. User speaks → interruption chain fires
2. LLM service receives InterruptionFrame
3. _handle_interruptions() iterates registered functions
4. get_weather has cancel_on_interruption=True → cancelled
5. FunctionCallCancelFrame broadcast both directions
6. Pipeline continues with user's new input
```

If `cancel_on_interruption=False` (e.g., a database write), the function call completes even through the interruption.

### Scenario 3: Muted user — interruption suppressed

```
User is muted (mute strategy active)

1. VAD detects speech → VADUserStartedSpeakingFrame
2. Strategy fires, but aggregator's mute check triggers
3. InterruptionFrame created but should_mute_frame=True
4. frame.complete() called manually (frame won't reach sink)
5. Bot continues speaking uninterrupted
```

### Scenario 4: Fatal error during LLM call

```
1. LLM API returns 500 error
2. LLM service calls push_error("API error", fatal=True)
3. ErrorFrame travels upstream to PipelineSource
4. on_pipeline_error event fires (app code can log/alert)
5. CancelFrame queued downstream
6. Every processor: _cancelling=True, queue_frame becomes no-op
7. Pipeline shuts down
```

---

## Reference

| File | What |
|------|------|
| `frames/frames.py` | `InterruptionFrame.complete()`, `InterruptionTaskFrame`, `UninterruptibleFrame`, `ErrorFrame`, `CancelFrame` |
| `processors/frame_processor.py` | `push_interruption_task_frame_and_wait()`, `_start_interruption()`, `__reset_process_queue()`, `create_task()`, `cancel_task()`, `push_error()` |
| `pipeline/task.py` | `_source_push_frame()` (TaskFrame→InterruptionFrame conversion, ErrorFrame handling), `_sink_push_frame()` (InterruptionFrame.complete()) |
| `pipeline/pipeline.py` | `PipelineSource`, `PipelineSink` — direct mode processors delegating to task callbacks |
| `utils/asyncio/task_manager.py` | `TaskManager` — task registry, wrapped coroutines, safe cancellation |
| `turns/user_start/vad_user_turn_start_strategy.py` | VAD-triggered turn start |
| `turns/user_turn_controller.py` | Coordinates start/stop strategies, routes frames |
| `processors/aggregators/llm_response_universal.py` | `_on_user_turn_started()` calls `push_interruption_task_frame_and_wait()`, mute path calls `frame.complete()` |
| `services/llm_service.py` | Cancels function calls on interruption |
| `services/tts_service.py` | `_handle_interruption()` resets TTS state; `InterruptibleTTSService` reconnects WebSocket |
| `transports/base_output.py` | `MediaSender.handle_interruptions()` cancels media tasks |
