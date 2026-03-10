# Topic 5: Transports

## What This Topic Covers

Transports are the I/O boundary of the pipeline — they move audio and video between the real world (microphones, speakers, WebRTC, telephony) and the frame-based pipeline. Everything inside the pipeline is frames; transports convert between frames and whatever the external medium uses (raw PCM, WebRTC tracks, WebSocket messages with μ-law encoding, etc.).

This topic covers the base transport architecture, the five concrete transport families, and the serializer system that bridges WebSocket transports to telephony providers.

## The Split Architecture

A transport is **not** a single FrameProcessor. It's a factory that produces **two separate processors** — one for input, one for output — that sit in different positions in the pipeline:

```python
transport = DailyTransport(...)
pipeline = Pipeline([
    transport.input(),     # ← position 0: receives audio/video from WebRTC
    stt,
    user_agg,
    llm,
    tts,
    transport.output(),    # ← position 5: sends audio/video to WebRTC
])
```

Why split? Because input and output do completely different things and sit at opposite ends of the pipeline. Input receives raw audio from the user's microphone and pushes `InputAudioRawFrame` downstream. Output receives `TTSAudioRawFrame` (or any `OutputAudioRawFrame`) from TTS and writes it to the user's speaker. They share configuration (sample rates, codecs) but operate independently.

### Class hierarchy

```
BaseObject
└── BaseTransport            ← NOT a FrameProcessor. Just a factory.
    ├── input() → FrameProcessor   (returns a BaseInputTransport)
    └── output() → FrameProcessor  (returns a BaseOutputTransport)

FrameProcessor
├── BaseInputTransport       ← IS a FrameProcessor. Lives in pipeline.
└── BaseOutputTransport      ← IS a FrameProcessor. Lives in pipeline.
```

`BaseTransport` extends `BaseObject`, not `FrameProcessor`. It never touches frames directly — it just holds shared config and lazily creates the input/output processors on first access (cached after that).

---

## BaseInputTransport

**File**: `src/pipecat/transports/base_input.py`

Receives media from the external world and pushes frames downstream into the pipeline.

### What it does

1. **Audio ingestion**: Maintains an `_audio_in_queue` (asyncio.Queue) and an `_audio_task` that pulls from it. Subclasses call `push_audio_frame(InputAudioRawFrame)` to feed audio in — the base class handles queuing and filtering.

2. **Optional audio filtering**: If `audio_in_filter` is set in params, the audio task runs each frame through the filter before pushing downstream. This is for preprocessing like noise suppression.

3. **Audio passthrough**: If `audio_in_passthrough=True` (default), input audio frames are pushed downstream so other processors (STT, VAD in UserAggregator) can see them. If false, audio is consumed but not forwarded.

4. **Video ingestion**: `push_video_frame(InputImageRawFrame)` pushes video directly downstream (no queue — video is less latency-sensitive than audio for this purpose).

5. **Lifecycle**: Handles `StartFrame` (initialize sample rate, create audio task), `EndFrame` (cancel audio task, clean up), `StopFrame` (pause without full shutdown), `CancelFrame` (immediate stop).

### Key state

- `_sample_rate` — initialized from `TransportParams.audio_in_sample_rate` or from `StartFrame.audio_in_sample_rate`
- `_bot_speaking` / `_user_speaking` — tracks speaking state for interruption logic
- `_paused` — set by `StopFrame`, cleared by next `StartFrame`

### What subclasses override

- `start_audio_in_streaming()` — hook to begin receiving audio from the transport SDK (e.g., subscribe to a WebRTC track)

### Deprecated VAD in transport

The base input transport has VAD/turn analysis support (`vad_analyzer`, `turn_analyzer` params), but this is **deprecated**. VAD now lives in `LLMUserAggregator` with `user_turn_strategies` (see Topic 7). You'll still see the code in the base class, but new pipelines shouldn't use it.

---

## BaseOutputTransport

**File**: `src/pipecat/transports/base_output.py`

Receives frames from the pipeline and writes media to the external world. This is significantly more complex than input because it handles buffering, chunking, resampling, timing, and mixing.

### The MediaSender inner class

Output transport doesn't process audio directly. It delegates to **MediaSender** instances — one per output destination. The default destination is `None` (always created). If you configure `audio_out_destinations=["speaker1", "speaker2"]`, you get additional MediaSenders.

Each MediaSender runs **three concurrent tasks**:

| Task | What it does |
|------|-------------|
| **Audio task** | Pulls from `_audio_queue`, calls `write_audio_frame()`, tracks bot speaking state |
| **Video task** | Streams live video or cycles through sprite images at the configured framerate |
| **Clock task** | Delivers frames with presentation timestamps (PTS) at the right moment |

### Audio chunking — why it matters for interruptions

When TTS generates a long audio response, it arrives as large `TTSAudioRawFrame` objects. If you queued those directly, an interruption would have to wait for the current large frame to finish playing before the queue could be drained.

Instead, the output transport **chunks** incoming audio into small pieces (default: 4 × 10ms = 40ms chunks):

```
audio_bytes_10ms = (sample_rate / 100) × num_channels × 2 bytes
chunk_size = audio_bytes_10ms × audio_out_10ms_chunks

Example: 16kHz mono, 4 chunks = 160 × 1 × 2 × 4 = 1280 bytes (40ms of audio)
```

The chunking logic buffers incoming audio, slices it into `chunk_size` pieces, and queues each piece separately. Leftover bytes stay in the buffer for the next frame. This means an interruption only has to wait at most 40ms before the queue is cleared.

### Resampling

TTS services often output at a different sample rate than the transport expects (e.g., ElevenLabs outputs 24kHz, but WebRTC might need 16kHz). Each MediaSender has a **stream resampler** that converts incoming audio to the transport's output sample rate before chunking.

### Bot speaking detection

The audio task tracks whether the bot is currently producing audio:

- When the first audio chunk is written → pushes `BotStartedSpeakingFrame` upstream
- When no audio arrives for `BOT_VAD_STOP_SECS` (0.35s) → pushes `BotStoppedSpeakingFrame` upstream
- While speaking, periodically pushes `BotSpeakingFrame` upstream

These frames flow upstream to processors like UserAggregator, which uses them for interruption logic ("don't interrupt if the bot isn't speaking").

### Interruption handling

When `InterruptionFrame` arrives (and `allow_interruptions=True`):

1. Cancel all three tasks (audio, video, clock)
2. Clear all queues and buffers
3. Recreate all three tasks from scratch
4. Push `BotStoppedSpeakingFrame` upstream

The bot stops talking immediately. Fresh tasks are ready for new audio.

### Video modes

Two modes controlled by `video_out_is_live`:

- **Live mode** (`True`): Streams video frames from the queue with wall-clock synchronization. Maintains framerate by calculating sleep time between frames. Resyncs if drift exceeds 5 frames.
- **Cyclic mode** (`False`): Cycles through a set of images (from `SpriteFrame`) at the configured framerate. Useful for animated avatars.

### What subclasses override

- `write_audio_frame(frame) → bool` — actually send audio bytes to the transport (WebRTC track, PyAudio stream, etc.)
- `write_video_frame(frame) → bool` — actually send video to the transport
- `send_message(frame)` — handle `OutputTransportMessageFrame`
- `register_audio_destination(dest)` / `register_video_destination(dest)` — multi-destination routing

### DTMF support

Output transport has built-in DTMF (dial tone) support:
- If the transport supports native DTMF (override `_supports_native_dtmf() → True`), it sends via the transport SDK
- Otherwise, it generates DTMF audio tones from frequency data and writes them as regular audio

---

## TransportParams

**File**: `src/pipecat/transports/base_transport.py`

Pydantic `BaseModel` shared by both input and output processors. Key parameters:

| Category | Parameter | Default | Purpose |
|----------|-----------|---------|---------|
| **Audio In** | `audio_in_enabled` | `False` | Enable audio input |
| | `audio_in_sample_rate` | `None` | Hz (falls back to StartFrame) |
| | `audio_in_channels` | `1` | Mono/stereo |
| | `audio_in_filter` | `None` | Audio preprocessing filter |
| | `audio_in_passthrough` | `True` | Forward audio downstream |
| **Audio Out** | `audio_out_enabled` | `False` | Enable audio output |
| | `audio_out_sample_rate` | `None` | Hz (falls back to StartFrame) |
| | `audio_out_channels` | `1` | Mono/stereo |
| | `audio_out_10ms_chunks` | `4` | Chunk size = N × 10ms |
| | `audio_out_mixer` | `None` | Audio mixer instance |
| | `audio_out_end_silence_secs` | `2` | Silence to send after EndFrame |
| **Video Out** | `video_out_enabled` | `False` | Enable video output |
| | `video_out_is_live` | `False` | Live streaming vs cyclic |
| | `video_out_width` | `1024` | Output width (px) |
| | `video_out_height` | `768` | Output height (px) |
| | `video_out_framerate` | `30` | FPS |

---

## Concrete Transports

### Daily WebRTC

**Files**: `src/pipecat/transports/daily/`

The most feature-rich transport. Wraps the [Daily](https://daily.co) real-time communication SDK.

**What it adds beyond the base**:
- **DailyTransportClient** — manages the Daily SDK connection (`CallClient`), room joining, participant lifecycle, track subscription. Runs blocking SDK calls in a thread executor.
- **Built-in transcription** — can enable Deepgram transcription directly in the Daily room (no separate STT service needed)
- **Recording** — start/stop call recording
- **SIP dial-in/dial-out** — telephony integration for PSTN calls via `DailySIPTransferFrame`
- **Custom audio tracks** — uses `VirtualSpeakerDevice` for output audio routing
- **Participant management** — tracks who's in the room, per-participant audio tracks (`audio_in_user_tracks=True`)
- **WebRTC VAD** — optional `WebRTCVADAnalyzer` using Daily's native VAD instead of Silero

**Daily-specific params** (`DailyParams`): `api_url`, `api_key`, `transcription_enabled`, `transcription_settings`, `dialin_settings`, `audio_in_user_tracks`

### LiveKit WebRTC

**Files**: `src/pipecat/transports/livekit/`

Wraps the [LiveKit](https://livekit.io) real-time communication SDK. Similar to Daily in concept but with a different SDK.

**What it adds beyond the base**:
- **LiveKitTransportClient** — manages `livekit.rtc.Room`, creates `AudioSource` and `LocalAudioTrack` for output
- **Retry logic** — `@retry` decorator with exponential backoff for connection failures
- **DTMF support** — RFC 4733 DTMF tones via audio (maps digits to tone codes)
- **Data channel** — send/receive binary data between participants
- **Participant events** — connected, disconnected, track subscription

**LiveKit-specific params** (`LiveKitParams`): inherits from `TransportParams` with no additional fields.

### WebSocket Transport

**Files**: `src/pipecat/transports/websocket/`

Generic WebSocket transport. Unlike WebRTC transports (which handle media natively), WebSocket transports rely on **serializers** to convert between frames and wire format.

**The serializer is required.** Without it, the transport doesn't know how to interpret incoming bytes or format outgoing frames.

```python
transport = WebsocketServerTransport(
    params=WebsocketServerParams(
        serializer=TwilioFrameSerializer(stream_sid="..."),
        add_wav_header=False,
    )
)
```

**Data flow**:
```
Client ──[WebSocket msg]──→ deserialize() → Frame → Pipeline → Frame → serialize() → [WebSocket msg]──→ Client
```

**WebSocket-specific params**: `serializer`, `add_wav_header` (adds WAV headers to audio for browser playback), `session_timeout`

### SmallWebRTC

**Files**: `src/pipecat/transports/smallwebrtc/`

Pure Python WebRTC using `aiortc` — no native SDK dependency.

**Key component**: `RawAudioTrack` — a custom `AudioStreamTrack` that generates 10ms audio chunks from a queue with precise timing. Yields silence if the queue is empty.

**Use cases**: development/testing, platforms where native WebRTC SDKs aren't available, browser signaling compatibility.

### Local Audio

**Files**: `src/pipecat/transports/local/audio.py`

Uses `pyaudio` for system microphone/speaker access. No network — direct device I/O.

- **Input**: PyAudio stream callback pushes audio frames to the base class's audio queue
- **Output**: Writes audio to PyAudio stream via `ThreadPoolExecutor`
- **Params**: `input_device_index`, `output_device_index` (PyAudio device IDs)

No serialization needed — audio is already raw PCM.

---

## Serializers

**Files**: `src/pipecat/serializers/`

Serializers convert frames to/from wire format for WebSocket transports. They're needed because WebSocket is a generic byte/text transport — it doesn't have built-in media handling like WebRTC.

### FrameSerializer base class

```python
class FrameSerializer(BaseObject):
    async def setup(frame: StartFrame):
        """Initialize with pipeline config (sample rates, etc.)"""

    @abstractmethod
    async def serialize(frame: Frame) -> str | bytes | None:
        """Frame → wire format. Return None to skip."""

    @abstractmethod
    async def deserialize(data: str | bytes) -> Frame | None:
        """Wire format → Frame. Return None to skip."""
```

### Telephony serializers

Six telephony providers, all following the same pattern: **Twilio**, **Plivo**, **Vonage**, **Telnyx**, **Exotel**, **Genesys**.

They all solve the same problem: telephony uses **μ-law encoding at 8kHz**, while the pipeline uses **linear PCM at 16kHz+**.

**Serialize (pipeline → phone)**:
1. Receive `OutputAudioRawFrame` (16kHz PCM)
2. Resample: 16kHz → 8kHz
3. Encode: PCM → μ-law
4. Format: base64-encode → wrap in provider-specific JSON

**Deserialize (phone → pipeline)**:
1. Receive provider-specific JSON/bytes from WebSocket
2. Decode: base64 → μ-law bytes
3. Decode: μ-law → PCM
4. Resample: 8kHz → 16kHz
5. Create `InputAudioRawFrame`

**DTMF**: Telephony serializers also convert DTMF events from provider format into `InputDTMFFrame`.

**Call termination**: If `auto_hang_up=True` (default) and `EndFrame`/`CancelFrame` arrives, the serializer calls the provider's API to hang up the phone call. Each provider has a different API endpoint and auth mechanism.

### Protobuf serializer

Binary serialization using Protocol Buffers. More bandwidth-efficient than JSON-based telephony serializers. Useful for custom WebSocket protocols where performance matters.

---

## How It All Fits Together

```
                          ┌─────────────────────────────────┐
                          │         BaseTransport            │
                          │  (factory, NOT a FrameProcessor) │
                          │                                  │
                          │  input() ──→ BaseInputTransport  │
                          │  output() ──→ BaseOutputTransport│
                          └─────────────────────────────────┘

  BaseInputTransport                              BaseOutputTransport
  ┌──────────────────┐                            ┌───────────────────────────┐
  │ push_audio_frame()│                           │  MediaSender (per dest)    │
  │       ↓          │                            │  ┌─────────────────────┐  │
  │  _audio_in_queue │                            │  │ Audio: buffer→chunk │  │
  │       ↓          │                            │  │  →resample→queue    │  │
  │  _audio_task     │                            │  │  →write_audio_frame │  │
  │       ↓          │                            │  ├─────────────────────┤  │
  │  [filter?]       │                            │  │ Video: resize→queue │  │
  │       ↓          │                            │  │  →write_video_frame │  │
  │  push downstream │                            │  ├─────────────────────┤  │
  │  (InputAudioRaw) │                            │  │ Clock: PTS→wait    │  │
  └──────────────────┘                            │  │  →push at right t   │  │
                                                  │  └─────────────────────┘  │
                                                  └───────────────────────────┘

  WebSocket adds:
  ┌─────────────────────────────────────────────┐
  │  WebSocket msg → serializer.deserialize()    │
  │       → Frame → pipeline → Frame             │
  │       → serializer.serialize() → WebSocket   │
  └─────────────────────────────────────────────┘
```

---

## Transport Comparison

| Transport | Protocol | Serializer | Audio Encoding | Unique Features |
|-----------|----------|------------|---------------|-----------------|
| **Daily** | WebRTC | Native | Codec-negotiated | Transcription, recording, SIP, participant mgmt |
| **LiveKit** | WebRTC | Native | Codec-negotiated | Retry logic, data channels, DTMF |
| **WebSocket** | WebSocket | **Required** | Via serializer | Generic, telephony-compatible |
| **SmallWebRTC** | WebRTC | Native | PCM (aiortc) | Pure Python, no native SDK |
| **Local** | System I/O | None | Raw PCM | PyAudio, dev/testing |

---

## Reference

| File | What |
|------|------|
| `transports/base_transport.py` | BaseTransport factory, TransportParams |
| `transports/base_input.py` | BaseInputTransport — audio/video ingestion |
| `transports/base_output.py` | BaseOutputTransport — buffering, chunking, MediaSender |
| `transports/daily/transport.py` | Daily WebRTC transport |
| `transports/livekit/transport.py` | LiveKit WebRTC transport |
| `transports/websocket/server.py` | WebSocket server transport |
| `transports/smallwebrtc/transport.py` | Pure Python WebRTC transport |
| `transports/local/audio.py` | Local microphone/speaker transport |
| `serializers/__init__.py` | FrameSerializer base class |
| `serializers/twilio.py` | Twilio μ-law serializer (representative of all telephony serializers) |
| `serializers/protobuf.py` | Protobuf binary serializer |
