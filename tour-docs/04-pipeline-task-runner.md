# Topic 4: Pipeline, Task & Runner

## What This Topic Covers

This is the execution layer — how Pipecat boots up, runs a conversation, handles errors, and shuts down. Topics 2 and 3 explained what frames are and how a single processor handles them. This topic shows what happens when you wire processors together and run the whole thing.

We'll trace six real scenarios through the pipeline, from "the human says hello" to "the server crashes." Each scenario names the exact components involved so you can map them to the code.

## The Setup (for all examples)

A typical voice bot pipeline looks like this in user code:

```python
pipeline = Pipeline([transport.input(), stt, user_agg, llm, tts, transport.output(), assistant_agg])
task = PipelineTask(pipeline)
runner = PipelineRunner()
await runner.run(task)
```

But what actually runs is more than what you wrote. PipelineTask wraps your pipeline:

```
PipelineRunner
  └── PipelineTask
        └── TaskSource → RTVIProcessor → PipelineSource → [your 7 processors] → PipelineSink → TaskSink
```

- **TaskSource / TaskSink** — the task's own sentinels. They catch frames that escape your pipeline and handle lifecycle events (start, end, cancel, interruption).
- **RTVIProcessor** — auto-injected for client protocol support. Transparent if you don't use RTVI.
- **PipelineSource / PipelineSink** — your pipeline's own sentinels. They route frames in/out of your processor chain.

All four sentinels use **direct mode** (no queues — zero overhead). Your actual processors (STT, LLM, TTS, etc.) have the dual-queue architecture from Topic 3.

### What are TaskSource and TaskSink, exactly?

Think of your pipeline as a **factory assembly line** inside a building. Your processors (STT, LLM, TTS) are the **workers on the line** — each does a specific job and passes the product to the next. PipelineSource/PipelineSink are the **doors** at each end of the room.

**TaskSource and TaskSink are the security guards posted outside those doors.** They don't do any assembly work — they manage the building itself.

**TaskSource** (guard at the entrance):
- **Translates management orders into factory instructions.** When external code says "end the conversation" (`EndTaskFrame`), the guard translates it into something workers understand (`EndFrame`). Workers don't know about task-level commands — they only understand pipeline-level frames.
- **Handles emergencies that escape the building.** When a worker sends an error upstream and it flies out through the front door, this guard catches it and decides: log it? shut everything down?
- **Bypasses the queue in emergencies.** When an interruption arrives, this guard doesn't politely wait in the mail queue — they kick the door open and shove the `InterruptionFrame` directly onto the line. If they waited, the bot would keep talking while the "please stop" message sat in a mailbox.

**TaskSink** (guard at the exit):
- **Confirms orders were carried out.** When `StartFrame` exits the back door, this guard says "Boot complete!" and fires `on_pipeline_started`. When `EndFrame` exits, "Shutdown complete!" → `on_pipeline_finished`.
- **Signals the control room.** The `_process_push_queue()` loop waits on "has the pipeline started yet?" / "has the pipeline ended?" — TaskSink sets the `_pipeline_start_event` and `_pipeline_end_event` flags that unblock that loop.
- **Completes interruption handshakes.** When an `InterruptionFrame` exits the back door, TaskSink calls `frame.complete()` — signaling back to whoever triggered it: "All queues have been drained, you're clear."

**Why can't Pipeline handle this itself?** Because Pipeline doesn't know about lifecycle. It's just plumbing — it links processors and routes frames. It doesn't know what "starting" or "shutting down" means. TaskSource/TaskSink are the management layer that gives the plumbing awareness of application lifecycle.

---

## Example 1: "Hi, what can you do?"

**The happy path — a normal conversation turn from boot to response.**

### Phase 1: Boot

Before any audio flows, the pipeline must start up.

1. `runner.run(task)` calls `task.run()`, which calls `_process_push_queue()`.
2. `_process_push_queue()` creates a **StartFrame** carrying audio sample rates (16kHz in, 24kHz out), interruption config (`allow_interruptions=True`), and metrics flags.
3. StartFrame is injected **directly** into `pipeline.queue_frame()` — it does NOT go through `_push_queue`. Why? It must be the very first frame. If it went through the queue, a race condition could let user frames arrive first.
4. StartFrame flows through every processor in order:
   - **TaskSource** → passes downstream (direct mode, instant)
   - **RTVIProcessor** → passes downstream
   - **PipelineSource** → passes downstream (direct mode)
   - **TransportInput** → opens WebRTC connection, starts receiving audio
   - **STT** → connects to Deepgram/Google/etc., opens WebSocket
   - **UserAggregator** → initializes LLMContext
   - **LLM** → connects to OpenAI/Anthropic/etc.
   - **TTS** → connects to ElevenLabs/Cartesia/etc.
   - **TransportOutput** → opens speaker/WebRTC output
   - **AssistantAggregator** → ready
   - **PipelineSink** → passes downstream (direct mode)
   - **TaskSink** → **catches StartFrame**, fires `on_pipeline_started`, starts heartbeat and idle monitor tasks
5. `_process_push_queue()` was blocked on `_pipeline_start_event.wait()`. Now it unblocks and enters the main loop: `frame = await _push_queue.get()`.

**Every processor is now connected and ready. No audio frames entered the pipeline until this moment.** This prevents the situation where audio arrives at STT before STT has connected to its service.

### Phase 2: The human speaks

The user says "Hi, what can you do?"

6. **TransportInput** receives raw audio from WebRTC. It creates `InputAudioRawFrame` (a **SystemFrame** — high priority, never dropped) and pushes it downstream. The transport doesn't analyze the audio — it just moves bytes.
7. **STT** receives audio in its Input Queue (high priority — it jumps ahead of anything else). STT streams the audio to Deepgram. Deepgram returns partial results as `InterimTranscriptionFrame` (so the UI can show "Hi, what can..." appearing in real-time). When the user stops speaking, Deepgram returns the final result.
8. STT creates a `TranscriptionFrame("Hi, what can you do?")` — a **DataFrame** — and pushes it downstream.
9. **UserAggregator** receives the TranscriptionFrame. Its internal VAD has been analyzing the audio frames flowing through it (see Example 2 for details on where VAD lives). It appends `{"role": "user", "content": "Hi, what can you do?"}` to the shared `LLMContext.messages` list, then creates an `LLMRunFrame` (which tells the LLM to generate a response) and pushes it downstream.
10. **LLM** receives the LLMRunFrame. It sends `context.messages` to OpenAI. OpenAI streams back tokens. For each token, LLM creates an `LLMTextFrame("I")`, `LLMTextFrame("'m")`, `LLMTextFrame(" a")`, `LLMTextFrame(" voice")`, `LLMTextFrame(" assistant")`, etc. — each pushed downstream as a DataFrame.
11. **TTS** receives each LLMTextFrame. It accumulates tokens until it has a full sentence (sentence aggregation mode), then sends "I'm a voice assistant that can help you with various tasks." to ElevenLabs. ElevenLabs streams back audio chunks as `TTSAudioRawFrame` — each pushed downstream.
12. **TransportOutput** receives each TTSAudioRawFrame and plays it over WebRTC. The user hears the bot speaking.
13. **AssistantAggregator** has been watching for LLMTextFrames and the `LLMFullResponseEndFrame` that signals the LLM is done. It appends `{"role": "assistant", "content": "I'm a voice assistant that can help you with various tasks."}` to the same shared `LLMContext.messages`.

**The LLMContext now has two messages. The next user turn will include the full conversation history.**

### What the pipeline layers did

- **Pipeline** (PipelineSource/Sink): silently routed frames between your processors. You never interacted with them directly.
- **PipelineTask** (TaskSource/TaskSink): booted everything with StartFrame, started monitoring, and is now sitting in the `_push_queue.get()` loop waiting for external events.
- **PipelineRunner**: is just `await task.run()`, doing nothing until a signal arrives.

---

## Example 2: User interrupts the bot

**The human cuts off the bot mid-sentence.**

The bot is halfway through saying "I'm a voice assistant that can help you with various tasks." The TTS has already generated 10 audio chunks. 5 have been played, 5 are sitting in TransportOutput's Process Queue waiting to be sent.

The user says "Wait, actually—"

### Where does VAD live?

VAD (Voice Activity Detection) does **not** live in the transport. TransportInput just pushes raw audio bytes (`InputAudioRawFrame`) downstream — it doesn't know or care whether someone is speaking.

VAD lives inside the **LLMUserAggregator**. When you create the aggregator, you pass it a VAD analyzer:

```python
user_agg, assistant_agg = LLMContextAggregatorPair(
    context,
    user_params=LLMUserAggregatorParams(vad_analyzer=SileroVADAnalyzer()),
)
```

The aggregator receives every `InputAudioRawFrame` that flows through it (audio frames are SystemFrames, so they enter via the high-priority Input Queue). Its internal `VADController` runs the Silero model on each audio chunk. When voice confidence exceeds the threshold (default 0.7) for long enough (default 0.2s), the VAD state transitions from QUIET → STARTING → SPEAKING.

### The interruption flow

1. The **UserAggregator's VAD** detects speech in the audio frames flowing through it. It broadcasts `VADUserStartedSpeakingFrame` — a **SystemFrame** that jumps priority queues.

2. The aggregator's internal **turn start strategy** (e.g., `VADUserTurnStartStrategy`) receives the VAD signal and pushes an `InterruptionTaskFrame` **upstream**. It does this **regardless of whether the bot is currently speaking**. Why? The system is idempotent by design — if the bot was idle, every Process Queue is already empty, so draining them is a harmless no-op. No need to track bot state as a precondition.

3. The InterruptionTaskFrame travels upstream through every processor (via `push_frame(frame, UPSTREAM)`), passing through their Input Queues as a SystemFrame (high priority). It reaches PipelineSource, escapes the pipeline, and arrives at **TaskSource**.

4. **TaskSource** (`_source_push_frame()`) catches the InterruptionTaskFrame. Here's the critical part: it does NOT put it in `_push_queue`. Instead, it creates an `InterruptionFrame` and injects it **directly** into `pipeline.queue_frame()`. Why bypass the push queue? Because the push queue task might be blocked waiting for an EndFrame to traverse the pipeline. If the interruption waited in line, the bot would keep talking until the EndFrame finished — defeating the purpose.

5. The **InterruptionFrame** enters the pipeline as a **SystemFrame** (HIGH priority). At every processor it passes through:
   - It enters the Input Queue and jumps ahead of everything
   - The Input Task calls `_start_interruption()`
   - `_start_interruption()` **cancels the Process Task** and **drains the Process Queue**
   - All pending DataFrames are dropped: those 5 remaining TTSAudioRawFrames in TransportOutput? Gone. The LLMTextFrame tokens still in TTS's queue? Gone.
   - But `EndFrame`, `FunctionCallResultFrame`, etc. survive (they're `UninterruptibleFrame`)
   - A fresh Process Task is created — the processor is clean and ready

6. The InterruptionFrame reaches **TaskSink**, which calls `frame.complete()` — this sets the `asyncio.Event` inside the frame, unblocking whoever called `push_interruption_task_frame_and_wait()`.

7. **The bot stops talking mid-sentence.** The user hears silence. New audio from the user ("Wait, actually—") is now flowing through STT normally, starting a fresh conversation turn.

### Why this design matters

**Idempotent interruptions.** The system doesn't need to decide "is this an interruption?" before pushing the frame. It always pushes it. If the bot wasn't speaking, the queues are empty and nothing happens. If the bot was mid-sentence, the queues are full and everything gets drained. Same code path, same frame, different outcome based on pipeline state.

**~1ms latency.** The interruption reaches every processor almost instantly because SystemFrames jump the priority queue. Without the dual-queue architecture (Topic 3), you'd have to wait for each processor to finish processing its current frame before handling the interruption. With 10 audio chunks queued, that could be 200ms+ of continued bot speech after the user started talking.

**VAD is pluggable.** Because VAD lives in the aggregator (not the transport), you can swap `SileroVADAnalyzer` for any other analyzer, or use a standalone `VADProcessor` instead. The transport layer stays clean — it just moves bytes.

---

## Example 3: "What's the weather in Paris?"

**Function calling — and what happens if the user interrupts during the API call.**

### How tools get wired up (before the call starts)

Function calling requires **two separate registrations** that Pipecat connects at runtime:

**1. Tool definitions** — what the LLM *knows about* (lives in `LLMContext`):

```python
weather_function = FunctionSchema(
    name="get_weather",
    description="Get the current weather",
    properties={"city": {"type": "string"}},
    required=["city"],
)
tools = ToolsSchema(standard_tools=[weather_function])
context = LLMContext(messages=[], tools=tools)  # tools live here
```

**2. Handler callbacks** — what *runs* when the tool is called (lives in `llm._functions` dict):

```python
async def fetch_weather(params: FunctionCallParams):
    result = await call_weather_api(params.arguments["city"])
    await params.result_callback(result)  # this is how you return the result

llm.register_function("get_weather", fetch_weather)  # stored in llm._functions
```

These are separate by design. The `FunctionSchema` describes the function's signature for the LLM API (so OpenAI knows *what it can call*). The handler is the actual code that runs (so Pipecat knows *what to execute*). They're matched by name at runtime.

When the LLM sends an API request, the **adapter** (`OpenAILLMAdapter`) converts `context.tools` into the provider's format and sends it alongside `context.messages`. OpenAI sees the tool definitions and can decide to call one.

### The conversation turn

1. The user says "What's the weather in Paris?" The audio flows through TransportInput → STT → UserAggregator → LLM as in Example 1.

2. The **LLM** sends the context to OpenAI. OpenAI responds with a **function call** instead of text: `{"name": "get_weather", "arguments": {"city": "Paris"}}`. The LLM **broadcasts** (both upstream and downstream via `broadcast_frame()`):
   - `FunctionCallInProgressFrame` (a **ControlFrame + UninterruptibleFrame**) — tells downstream processors "I'm waiting for a tool result, don't expect text yet." The upstream copy reaches the context aggregator, which updates the LLM context with an IN_PROGRESS status.
   - `LLMFullResponseEndFrame` — signals the LLM response is done (even though it's a function call, not text).

3. The LLM looks up the handler you registered with `llm.register_function("get_weather", handler)` and **spawns it as an async task inside the LLM service** (via `create_task()`). The handler — your application code — makes an HTTP request to a weather API. This takes 2 seconds. During this time, the handler is running inside the LLM's process, not as an external entity.

4. **During those 2 seconds, the user says "Never mind."** An InterruptionFrame fires (same as Example 2). It drains every processor's Process Queue. But:
   - The `FunctionCallInProgressFrame` **survives** — it's `UninterruptibleFrame`
   - Any LLMTextFrames from a previous response would be dropped, but there aren't any here
   - The handler's async task is still running — the interruption drains processor queues but can't cancel the handler mid-await

5. The weather API returns to the handler. The handler calls `await params.result_callback({"temp": "18°C", "condition": "sunny"})`. This callback is a closure defined inside the LLM service — it calls `broadcast_frame(FunctionCallResultFrame(...))`, sending the result **both upstream and downstream** from the LLM's position in the pipeline.

6. The upstream `FunctionCallResultFrame` (**DataFrame + UninterruptibleFrame**) reaches the context aggregator, which updates the LLM context with the function result. Because it's `UninterruptibleFrame`, it survived the interruption. The LLM then re-invokes itself with the updated context (user question + function call + function result + "Never mind").

7. The full context now includes both the function result AND "Never mind." OpenAI sees the complete picture and responds: "No problem! For the record, it's 18°C and sunny in Paris."

### What the LLM context looks like at each stage

```python
# After step 1 — user speaks:
[{"role": "user", "content": "What's the weather in Paris?"}]

# After step 2 — OpenAI returns function call, aggregator adds tool messages:
[{"role": "user", "content": "What's the weather in Paris?"},
 {"role": "assistant", "tool_calls": [{"id": "call_123", "function": {"name": "get_weather", "arguments": "{\"city\": \"Paris\"}"}}]},
 {"role": "tool", "content": "IN_PROGRESS", "tool_call_id": "call_123"}]

# After step 4 — user interrupts, aggregator adds new user message:
[...,
 {"role": "tool", "content": "IN_PROGRESS", "tool_call_id": "call_123"},
 {"role": "user", "content": "Never mind."}]

# After step 5-6 — handler returns, aggregator updates tool message with result:
[{"role": "user", "content": "What's the weather in Paris?"},
 {"role": "assistant", "tool_calls": [{"id": "call_123", ...}]},
 {"role": "tool", "content": "{\"temp\": \"18°C\", \"condition\": \"sunny\"}", "tool_call_id": "call_123"},
 {"role": "user", "content": "Never mind."}]
# ↑ This updated context gets sent back to OpenAI → generates the final text response
```

This follows the OpenAI conversation format: each function call produces two messages — an `assistant` message with `tool_calls`, and a `tool` message with the result. The `tool_call_id` links them together.

### Why UninterruptibleFrame matters

Without it, the `FunctionCallResultFrame` would be dropped from processor queues when the user interrupted. The LLM context would have a function call without a corresponding result — OpenAI would error on the next request because this violates the API contract. The `UninterruptibleFrame` mixin ensures function call frames survive queue drains, keeping the conversation context consistent.

### Where does the function handler run?

A common misconception is that function handlers run "outside" the pipeline. In fact, the handler you register with `llm.register_function()` is stored inside the LLM service and **spawned as an async task** (`create_task()`) within the LLM service when a function call arrives. The result flows back into the pipeline via a `result_callback` closure that calls `broadcast_frame()` — sending the `FunctionCallResultFrame` both upstream (to context aggregators) and downstream (to other processors). The handler is your code, but it runs in the LLM's process context.

### Why does broadcast_frame send both directions?

The **upstream** copy is the one that drives the conversation forward — it reaches the context aggregator, updates the LLM context, and triggers re-invocation. But the **downstream** copy serves three concrete consumers that need to know "a function call is happening" without being part of the LLM context loop:

| Downstream consumer | What it does with function call frames |
|---|---|
| **RTVIProcessor** | Converts `FunctionCallInProgressFrame` → `RTVILLMFunctionCallInProgressMessage` for the client UI (so the browser can show "Calling get_weather..."). Converts the result → `RTVILLMFunctionCallStoppedMessage`. |
| **UserIdleProcessor** | Sets `_interrupted = True` when it sees `FunctionCallInProgressFrame` — **pausing idle timeout detection**. Without this, a 2-second API call would trigger the "user is idle" callback. Resets when the result arrives. |
| **STTMuteFilter** | Tracks function call state to intelligently manage STT muting during function execution. |

The LLM service doesn't know (or care) which of these are downstream — it just broadcasts, and whoever is listening reacts.

**Why not use events instead?** Events fire immediately, potentially before earlier frames have been processed. A broadcast frame travels **through the pipeline in order**, interleaved with other frames. This preserves the timing guarantees of the pipeline — the RTVIProcessor sees `FunctionCallInProgressFrame` at exactly the right moment relative to other frames, not "sometime around when it happened."

### How does the LLM get re-invoked?

After the handler calls `result_callback`, the `FunctionCallResultFrame` reaches the **context aggregator** (upstream). The aggregator:
1. Finds the `{role: "tool", content: "IN_PROGRESS", tool_call_id: "call_123"}` message in context
2. Replaces `"IN_PROGRESS"` with the actual result JSON
3. Pushes an updated `LLMContextFrame` **upstream** back to the LLM service

This triggers a fresh API call to OpenAI with the complete context — the user's question, the tool call, and now the tool result. OpenAI generates a natural language response, and `LLMTextFrame` tokens flow downstream to TTS as usual. From the pipeline's perspective, it looks just like a normal LLM response — the function call loop is invisible to processors downstream of the LLM.

---

## Example 4: The LLM service crashes

**Error handling — what happens when a service fails mid-response.**

1. The user asks "Tell me a long story about a dragon." The LLM starts streaming tokens. After "Once upon a time, in a land far—" the connection to OpenAI drops.

2. The **LLM processor** catches the exception in its `process_frame()`. It calls `await self.push_error("OpenAI connection lost", exception=e, fatal=False)`.

3. `push_error()` creates an `ErrorFrame` with `fatal=False` and pushes it **upstream** (errors always go upstream — toward the source, not the speaker). The ErrorFrame is a **SystemFrame**, so it jumps the priority queue at every processor it passes through.

4. The ErrorFrame travels upstream through TTS (which ignores it — it's not for TTS), through UserAggregator, through STT, through PipelineSource, out of the user pipeline, and reaches **TaskSource**.

5. **TaskSource** (`_source_push_frame()`) catches the ErrorFrame. Because `fatal=False`:
   - It fires `on_pipeline_error` event (your application code can listen to this)
   - It logs a warning
   - It does **NOT** shut down the pipeline

6. Your application code, listening to `on_pipeline_error`, could switch to a backup LLM, retry the request, or say "Sorry, let me try that again."

### What if fatal=True?

If the LLM had called `push_error(msg, fatal=True)` — or pushed a `FatalErrorFrame` — TaskSource would queue a `CancelFrame` into the pipeline. The CancelFrame traverses every processor, each one shuts down immediately, and the pipeline terminates. `PipelineRunner.run()` catches the `CancelledError` and returns.

**The distinction matters**: a transient network glitch should be `fatal=False` (let the app decide). A configuration error (wrong API key, unsupported model) should be `fatal=True` (no point continuing).

---

## Example 5: The user walks away

**Idle timeout — the pipeline detects silence and shuts down.**

1. The bot says "Is there anything else I can help with?" The user puts down their phone and walks away.

2. Time passes. No `UserSpeakingFrame` or `BotSpeakingFrame` flows through the pipeline.

3. Inside PipelineTask, the **IdleFrameObserver** is watching every frame that passes through the pipeline (via the TaskObserver proxy). It has an `asyncio.Event` called `_idle_event` that it sets whenever it sees speaking activity. For 5 minutes, it hasn't set it.

4. The **`_idle_monitor_handler()`** task was waiting: `await asyncio.wait_for(_idle_event.wait(), timeout=300)`. The timeout fires.

5. PipelineTask fires the `on_idle_timeout` event. If you registered a handler, it runs now — you could play "Are you still there?" or log the timeout.

6. Since `cancel_on_idle_timeout=True` (the default), PipelineTask calls `self.cancel()`.

7. `cancel()` queues a `CancelFrame` into `_push_queue`. The CancelFrame traverses the pipeline — each processor shuts down. The pipeline ends.

### How observers work (without blocking the pipeline)

The IdleFrameObserver doesn't sit inside the pipeline chain. Instead, it's attached via the **TaskObserver proxy**. Here's how:

- When any processor calls `push_frame()`, the TaskObserver is notified.
- TaskObserver maintains a separate `asyncio.Queue` + `asyncio.Task` for each observer (a "proxy").
- The frame notification is put into the observer's queue — this is non-blocking, so `push_frame()` returns instantly.
- The observer's own task drains its queue and processes events at its own pace.

This means a slow observer (e.g., one that writes to a database) can never block audio frames from flowing through the pipeline. It just processes events later.

---

## Example 6: "Goodbye" — graceful shutdown

**The conversation ends normally.**

1. The user says "Goodbye, thanks for your help!" The bot responds "You're welcome! Have a great day."

2. Your application code detects the end-of-conversation (maybe the LLM returned a `end_conversation` function call, or a timer expired). It calls:
   ```python
   await task.stop_when_done()
   ```

3. `stop_when_done()` queues an **EndFrame** into `_push_queue`.

4. `_process_push_queue()` dequeues the EndFrame and injects it into the pipeline head. The EndFrame is `UninterruptibleFrame` — even if the user speaks again, it won't be dropped.

5. EndFrame flows through every processor in order. Each processor's `super().process_frame()` handles it:
   - **TransportInput** → stops receiving audio, closes WebRTC input
   - **STT** → closes connection to Deepgram
   - **UserAggregator** → nothing to clean up
   - **LLM** → closes connection to OpenAI
   - **TTS** → closes connection to ElevenLabs
   - **TransportOutput** → flushes any remaining audio, closes WebRTC output
   - **AssistantAggregator** → nothing to clean up

6. EndFrame exits through PipelineSink → TaskSink. `_sink_push_frame()` fires `on_pipeline_finished` and sets `_pipeline_end_event`.

7. `_process_push_queue()` was waiting on `_pipeline_end_event`. It unblocks, exits the while loop, calls `_cleanup()` (cancels heartbeat, idle monitor, observer tasks).

8. `task.run()` returns. `PipelineRunner.run()` removes the task from its registry, runs optional `gc.collect()`, returns.

### EndFrame vs CancelFrame vs StopFrame

| | EndFrame | CancelFrame | StopFrame |
|---|---|---|---|
| **Triggered by** | `task.stop_when_done()` | `task.cancel()`, SIGINT, fatal error | Direct queue |
| **How it travels** | Through every processor in order | Through every processor (20s timeout) | Through every processor |
| **Processors do** | Graceful cleanup | Immediate shutdown | Stop processing but keep connections open |
| **Pipeline cleanup** | Yes — `_cleanup()` runs | Yes — `_cleanup()` runs | No — connections stay open |
| **Use case** | Normal end of conversation | Error recovery, Ctrl+C | Pausing (rare) |

---

## Quick Reference: The Three Layers

| Layer | Class | What it adds | You interact with it via |
|-------|-------|-------------|------------------------|
| **Pipeline** | `Pipeline` | Wires processors into a chain with Source/Sink sentinels | `Pipeline([proc1, proc2, ...])` |
| **Task** | `PipelineTask` | Double-wraps pipeline, manages lifecycle, heartbeats, idle timeout, observers | `task.queue_frame()`, `task.stop_when_done()`, `task.cancel()`, event handlers |
| **Runner** | `PipelineRunner` | Signal handling (SIGINT/SIGTERM), GC, entry point | `await runner.run(task)` |

### Pipeline Internals

A `Pipeline` is itself a `FrameProcessor`. It chains your processors into a doubly-linked list with **PipelineSource** at position `[0]` and **PipelineSink** at position `[-1]`. Both use direct mode (no queues).

When a frame hits a boundary:
- **Downstream frame hits PipelineSink** → escapes to whoever owns this pipeline (the parent pipeline or PipelineTask)
- **Upstream frame hits PipelineSource** → escapes upstream

This is what makes nesting work: a Pipeline inside another Pipeline behaves like any other processor.

### PipelineTask Internals

PipelineTask wraps your pipeline with TaskSource and TaskSink, creating the full chain:

```
_push_queue → TaskSource → [RTVI] → PipelineSource → [your processors] → PipelineSink → TaskSink
```

**TaskSource** catches upstream frames and translates them. Processors inside the pipeline don't know they're inside a task — they speak "pipeline language." When a processor wants to end the conversation, it pushes an `EndTaskFrame` upstream (task language). TaskSource catches it and translates it to an `EndFrame` (pipeline language) that workers understand. The full translation table:

| Processor pushes upstream (task language) | TaskSource translates to (pipeline language) | Meaning |
|---|---|---|
| `EndTaskFrame` | `EndFrame` | "End gracefully" |
| `CancelTaskFrame` | `CancelFrame` | "Abort now" |
| `InterruptionTaskFrame` | `InterruptionFrame` (injected directly, bypassing `_push_queue` to avoid deadlocks) | "User interrupted, drain queues" |

**TaskSink** catches downstream frames: fires lifecycle events (`on_pipeline_started`, `on_pipeline_finished`), sets gate events that unblock `_process_push_queue()`.

### One StartFrame/EndFrame per conversation

A `StartFrame` / `EndFrame` pair maps to **one conversation session** — one phone call, one WebRTC session, one chat. The lifecycle is:

1. Call connects → `StartFrame` flows through → all services open connections
2. Conversation happens → DataFrames flow back and forth (many user/bot turns)
3. Call ends → `EndFrame` flows through → all services close connections

Everything between Start and End is the conversation, no matter how many turns it takes.

**`_push_queue`** is how external code sends frames into the running pipeline. `task.queue_frame(frame)` puts a frame here; the `_process_push_queue` task consumes it and injects it into the pipeline head.

### ParallelPipeline

Runs multiple pipeline branches in parallel. Every frame is fanned out to all branches. Results are deduplicated (by `frame.id`) on the way back upstream. Lifecycle frames (`StartFrame`, `EndFrame`, `CancelFrame`) are synchronized: the pipeline waits for ALL branches to finish processing before pushing one copy downstream.

```python
ParallelPipeline(
    [STTService1()],   # Branch 1: e.g., Deepgram
    [STTService2()],   # Branch 2: e.g., Google STT
)
# Both receive the same audio; first transcription wins on upstream dedup
```

---

## Diagrams

- `04-pipeline-wiring.excalidraw` — Architecture diagram: PipelineRunner > PipelineTask > Pipeline nesting, escape hatches, monitoring, ParallelPipeline
- `04-interruption-sequence.excalidraw` — Sequence diagrams for Example 2 (interruption) and Example 3 (function calling + interruption during API wait), in the same file for side-by-side comparison

## Reference

| File | Line | What |
|------|------|------|
| `pipeline.py:21` | `PipelineSource` | Entry sentinel with upstream escape |
| `pipeline.py:55` | `PipelineSink` | Exit sentinel with downstream escape |
| `pipeline.py:91` | `Pipeline` | Processor chain + linking |
| `task.py:157` | `PipelineTask` | Main orchestrator |
| `task.py:628` | `queue_frame()` | External frame injection API |
| `task.py:823` | `_process_push_queue()` | Core pipeline engine loop |
| `task.py:866` | `_source_push_frame()` | Upstream escape hatch |
| `task.py:905` | `_sink_push_frame()` | Downstream escape hatch |
| `task.py:939` | `_heartbeat_push_handler()` | Heartbeat injection loop |
| `task.py:968` | `_idle_monitor_handler()` | Idle timeout detection |
| `runner.py:26` | `PipelineRunner` | Signal-aware entry point |
| `parallel_pipeline.py:25` | `ParallelPipeline` | Fan-out/fan-in with dedup |
