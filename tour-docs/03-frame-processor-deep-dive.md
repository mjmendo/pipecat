# Topic 3: FrameProcessor Deep Dive

Source: `src/pipecat/processors/frame_processor.py`

## What Is It?

Every box in a Pipecat pipeline is a `FrameProcessor`. STT, LLM, TTS, transports, aggregators — they all extend this base class. Understanding it means understanding how every processor works internally.

## The Mental Model

Think of each processor as a **worker with two inboxes**:

1. **Urgent inbox** (Input Queue) — for system frames like interruptions, VAD signals, errors. Always checked first.
2. **Regular inbox** (Process Queue) — for data and control frames like audio, text, LLM tokens. Processed in order.

When the boss says "drop everything" (interruption), the regular inbox gets dumped, but critical mail (UninterruptibleFrame) is kept.

## How Frames Flow Through a Single Processor

```
prev processor calls queue_frame(frame, direction)
         │
         ▼
   ┌─ Input Queue (PriorityQueue) ─┐
   │  SystemFrame → HIGH priority   │
   │  Everything else → LOW         │
   └────────────┬───────────────────┘
                │
         Input Task (always running)
           │            │
     SystemFrame?    No → enqueue in Process Queue
           │                    │
     process_frame()     Process Task dequeues
     called immediately  → process_frame(frame, dir)  ← YOUR override
                                │
                         push_frame(frame, dir)
                                │
                    next.queue_frame()  [downstream]
                 or prev.queue_frame()  [upstream]
```

## The Two Methods You Care About

### `process_frame(frame, direction)` — what you override

This is where your logic goes. You receive a frame, decide what to do, and push results.

**Example: a simple profanity filter**
```python
class ProfanityFilter(FrameProcessor):
    async def process_frame(self, frame, direction):
        await super().process_frame(frame, direction)  # handles StartFrame, interruptions, etc.

        if isinstance(frame, TranscriptionFrame):
            frame.text = self._censor(frame.text)

        # Always push the frame onward (in the direction it came from)
        await self.push_frame(frame, direction)
```

**Why call `super().process_frame()`?** The base class handles `StartFrame` (boots the processor), `InterruptionFrame` (drains queues), `CancelFrame` (shuts down), and pause/resume. If you skip it, your processor won't respond to any of those.

**Why push in the same direction?** By default, all frames should continue in the direction they arrived. If you receive a downstream frame, push it downstream. The exception is errors — those go upstream via `push_error()`.

### `push_frame(frame, direction)` — how you send results

Sends a frame to the next processor (downstream) or previous processor (upstream). Under the hood it calls `next.queue_frame()` or `prev.queue_frame()`.

## Practical Scenarios

### Scenario 1: User interrupts the bot mid-sentence

The TTS has 10 audio chunks queued in the Process Queue. An `InterruptionFrame` arrives:

1. It enters the Input Queue as a **SystemFrame** (HIGH priority), jumping ahead of everything
2. The Input Task processes it immediately — calls `_start_interruption()`
3. `_start_interruption()` **cancels** the Process Task (which was about to dequeue the next audio chunk)
4. The Process Queue is **drained** — all 10 audio chunks are dropped
5. But if there was an `EndFrame` in the queue? It's `UninterruptibleFrame` — it's kept
6. A fresh Process Task is created, ready for the new user turn

**Why this matters:** Without this mechanism, the bot would keep speaking old audio while trying to process new user input. The dual-queue design lets system frames bypass and cancel data frames instantly.

### Scenario 2: STT receives audio while LLM is thinking

Audio frames (`InputAudioRawFrame`) are **SystemFrames**. Even if the STT's Process Queue is busy handling something, incoming audio is never blocked — it enters the high-priority Input Queue and gets routed appropriately.

### Scenario 3: A function call result must survive interruptions

The user says "what's the weather?" The LLM calls a tool. While waiting for the API response, the user says something else (interruption). The `FunctionCallResultFrame` is `UninterruptibleFrame` — when the Process Queue is drained, this frame survives. The tool result is delivered even though everything else was cleared.

### Scenario 4: Pipeline boundaries need zero overhead

`PipelineSource` and `PipelineSink` use `enable_direct_mode=True`. In direct mode, `queue_frame()` skips both queues entirely and calls `process_frame()` synchronously. This avoids adding async overhead at every pipeline boundary (which would add up in nested pipelines).

## The Linking Model

Processors form a **doubly-linked list**:

```python
processor.link(next_processor)
# Sets: processor._next = next_processor
#        next_processor._prev = processor
```

`push_frame(frame, DOWNSTREAM)` calls `self._next.queue_frame(frame)`
`push_frame(frame, UPSTREAM)` calls `self._prev.queue_frame(frame)`

The `Pipeline` class calls `link()` on consecutive processors to wire the chain.

## Event Hooks

Every processor fires these events (all synchronous — must be fast):

| Event | When | Use case |
|-------|------|----------|
| `on_before_process_frame` | Before `process_frame()` | Logging, debugging |
| `on_after_process_frame` | After `process_frame()` | Metrics, tracing |
| `on_before_push_frame` | Before `push_frame()` | Intercepting output |
| `on_after_push_frame` | After `push_frame()` | Confirming delivery |
| `on_error` | When `push_error()` is called | Error handling |

## Task Management

Always use `self.create_task(coro, name)` instead of raw `asyncio.create_task()`. The `TaskManager` tracks all tasks and cleans them up when the processor shuts down. Use `await self.cancel_task(task, timeout)` for cancellation — the default 1s timeout prevents hangs from libraries that swallow `CancelledError`.

## Key Takeaways

1. **Override `process_frame()`**, always call `super()`, always push frames onward
2. **SystemFrames jump the queue** — that's why interruptions feel instant
3. **Interruptions drain the Process Queue** but keep UninterruptibleFrames
4. **Direct mode** skips queues entirely — used at pipeline boundaries for performance
5. **Errors go upstream** via `push_error()` — so the PipelineTask can handle them

## Diagram

See `03-frame-processor-internals.excalidraw` for the visual diagram.

## Reference

| File | Line | What |
|------|------|------|
| `frame_processor.py:86` | `FrameProcessorQueue` | Priority queue implementation |
| `frame_processor.py:142` | `FrameProcessor` | Main class |
| `frame_processor.py:633` | `queue_frame()` | Entry point for incoming frames |
| `frame_processor.py:690` | `process_frame()` | Override point |
| `frame_processor.py:777` | `push_frame()` | Send frames to neighbors |
| `frame_processor.py:933` | `_start_interruption()` | Interruption handling |
| `frame_processor.py:1044` | `__reset_process_queue()` | Queue draining (keeps UninterruptibleFrame) |
| `frame_processor.py:1085` | `__input_frame_task_handler()` | Input task loop |
| `frame_processor.py:1113` | `__process_frame_task_handler()` | Process task loop |
