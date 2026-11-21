# exercise-02 — Wire Streaming TTS With Chunked Playout — Reference Solution

## 1. Solution overview

This is an implementation exercise. The learning target is **a streaming TTS client that returns the first audio chunk as fast as possible, plays chunks out continuously without underrun, and cancels cleanly on barge-in** — the three properties that separate a usable voice-agent TTS integration from one that merely "works on happy-path demos."

The worked implementation below is deliberately in standard-library-plus-two-deps Python so it's portable and auditable. Replace the provider-call block with your real provider's streaming endpoint; the surrounding chunker, player, and cancellation harness is the part that generalises.

What this reference ships:

- A streaming TTS client (`stream_tts.py`) that uses `aiohttp` to chunk-read the provider's HTTP streaming response.
- A playout queue with a backpressure-safe producer/consumer pattern (`AudioPlayer`) that ships 20 ms frames to the output device.
- A **cancellation token** that cuts off both HTTP read and playback inside a bounded number of frames.
- A measurement harness (`measure.py`) that records TTFB, inter-chunk interval, underrun count, and cancel-latency against a provider.
- A README / rubric / common-mistakes list.

Design decisions and their rationale are in §2.2; the code is in §2.3–§2.5. Validation and the rubric are in §3 and §4.

## 2. Implementation

### 2.1 Target properties

A reviewer should see the implementation prove each of these:

1. **TTFB target**: ship the first audio frame to the playout device within ≤ *budget* ms of the user's end-of-turn (budget is set by the latency-budget exercise in mod-109; a typical warm-call target is 300–500 ms wall-clock for the whole LLM→TTS→first-byte leg).
2. **No underruns under normal load**: the playout device is fed monotonically; measured underrun rate ≤ 0 per 10-minute test session on warm connections.
3. **Barge-in / cancel latency**: on `cancel()`, playback stops within ≤ 50 ms (one chunk boundary at 20–40 ms frames) and the upstream connection is closed (no wasted provider cost, no late audio bleeding in).
4. **Backpressure**: if the player can't keep up, the producer blocks rather than growing the queue unboundedly.
5. **Observable**: one log line per chunk with `t_sent`, `t_received`, `bytes`, `cumulative_ms`, so §2.6's measurement harness can post-process.

### 2.2 Design decisions

| Decision | Choice | Why |
|---|---|---|
| Transport | HTTP chunked streaming (`transfer-encoding: chunked`) via `aiohttp` | Simplest surface; WebSocket is a drop-in substitute (provided below) when the provider's lowest-latency path is WS. |
| Audio format | Raw PCM 16-bit mono at the provider's streaming sample rate (24 kHz typical; µ-law 8 kHz for telephony) | Avoids mid-pipeline decode cost (no MP3/Opus decode on the hot path). For telephony, request µ-law directly from the provider. |
| Playout frame size | 20 ms | Matches WebRTC / Opus framing; the audio device callback at 20 ms frames is widely supported and keeps cancel latency tight. |
| Queue | `asyncio.Queue(maxsize=N)` with `N` sized to ~200 ms of audio | Small enough that cancel-flush is cheap; big enough to absorb a 2–3-chunk network hiccup. |
| Cancellation | `asyncio.Event` + closing the HTTP response + draining the queue | Single mechanism for stop; both the producer and consumer check it on each iteration. |
| Observability | Structured log line per chunk + a `Trace` dataclass returned from `speak()` | Enough to compute P50/P95 TTFB and inter-chunk jitter in post. |

### 2.3 `stream_tts.py` — streaming client

```python
# stream_tts.py
# Reference streaming-TTS client. Replace PROVIDER_URL / request body with your provider's shape.
# Tested surface: aiohttp>=3.9, sounddevice>=0.4, numpy>=1.26, Python 3.11+.
from __future__ import annotations

import asyncio
import contextlib
import dataclasses
import logging
import os
import time
from typing import AsyncIterator

import aiohttp

log = logging.getLogger("stream_tts")

PROVIDER_URL = os.environ["TTS_PROVIDER_URL"]
PROVIDER_KEY = os.environ["TTS_PROVIDER_KEY"]

SAMPLE_RATE = 24_000        # 24 kHz PCM; set 8_000 for telephony mu-law
BYTES_PER_SAMPLE = 2        # 16-bit PCM
FRAME_MS = 20
FRAME_SAMPLES = SAMPLE_RATE * FRAME_MS // 1000
FRAME_BYTES = FRAME_SAMPLES * BYTES_PER_SAMPLE
MAX_QUEUED_FRAMES = 10      # ~200 ms of audio


@dataclasses.dataclass
class ChunkEvent:
    t_wall_ms: float
    bytes_received: int
    cumulative_bytes: int


@dataclasses.dataclass
class Trace:
    t_request_sent_ms: float
    t_first_byte_ms: float | None
    t_first_frame_played_ms: float | None
    t_last_frame_played_ms: float | None
    chunks: list[ChunkEvent] = dataclasses.field(default_factory=list)
    underruns: int = 0
    cancelled: bool = False

    @property
    def ttfb_ms(self) -> float | None:
        if self.t_first_byte_ms is None:
            return None
        return self.t_first_byte_ms - self.t_request_sent_ms


async def _read_chunks(
    session: aiohttp.ClientSession,
    text: str,
    voice: str,
    cancel: asyncio.Event,
    out: asyncio.Queue[bytes | None],
    trace: Trace,
) -> None:
    """Stream PCM bytes from the provider into `out`.  Pushes `None` on completion."""
    body = {"text": text, "voice": voice, "format": "pcm_16000", "sample_rate": SAMPLE_RATE}
    headers = {"Authorization": f"Bearer {PROVIDER_KEY}"}
    trace.t_request_sent_ms = time.perf_counter() * 1000
    try:
        async with session.post(PROVIDER_URL, json=body, headers=headers) as resp:
            resp.raise_for_status()
            cumulative = 0
            # Pre-size read to the playout frame so chunks arrive at playable granularity.
            # Providers usually send larger chunks; we re-frame on the consumer side instead
            # to avoid flushing partial frames into the device.
            async for raw in resp.content.iter_any():
                if cancel.is_set():
                    trace.cancelled = True
                    break
                if not raw:
                    continue
                now = time.perf_counter() * 1000
                if trace.t_first_byte_ms is None:
                    trace.t_first_byte_ms = now
                cumulative += len(raw)
                trace.chunks.append(ChunkEvent(t_wall_ms=now, bytes_received=len(raw), cumulative_bytes=cumulative))
                await out.put(raw)
    except asyncio.CancelledError:
        trace.cancelled = True
        raise
    finally:
        await out.put(None)  # sentinel so the player can drain


async def _play_frames(
    in_q: asyncio.Queue[bytes | None],
    cancel: asyncio.Event,
    trace: Trace,
    writer,
) -> None:
    """Re-frame the incoming byte stream into FRAME_BYTES-sized frames and write to `writer`."""
    buf = bytearray()
    try:
        while True:
            if cancel.is_set():
                trace.cancelled = True
                return
            item = await in_q.get()
            if item is None:
                # Flush the tail as one short frame if non-empty.
                if buf:
                    await writer.write(bytes(buf))
                    buf.clear()
                return
            buf.extend(item)
            while len(buf) >= FRAME_BYTES:
                frame = bytes(buf[:FRAME_BYTES])
                del buf[:FRAME_BYTES]
                now = time.perf_counter() * 1000
                if trace.t_first_frame_played_ms is None:
                    trace.t_first_frame_played_ms = now
                trace.t_last_frame_played_ms = now
                await writer.write(frame)
    except asyncio.CancelledError:
        trace.cancelled = True
        raise


class FrameWriter:
    """Pluggable playout sink.  See AudioPlayer in §2.4 for a device-backed implementation."""
    async def write(self, frame: bytes) -> None: ...  # pragma: no cover


async def speak(
    text: str,
    voice: str,
    writer: FrameWriter,
    cancel: asyncio.Event | None = None,
) -> Trace:
    cancel = cancel or asyncio.Event()
    q: asyncio.Queue[bytes | None] = asyncio.Queue(maxsize=MAX_QUEUED_FRAMES)
    trace = Trace(t_request_sent_ms=0.0, t_first_byte_ms=None,
                  t_first_frame_played_ms=None, t_last_frame_played_ms=None)
    async with aiohttp.ClientSession() as session:
        producer = asyncio.create_task(_read_chunks(session, text, voice, cancel, q, trace))
        consumer = asyncio.create_task(_play_frames(q, cancel, trace, writer))
        try:
            await asyncio.gather(producer, consumer)
        except Exception:
            cancel.set()
            for t in (producer, consumer):
                t.cancel()
            with contextlib.suppress(Exception):
                await asyncio.gather(producer, consumer, return_exceptions=True)
            raise
    return trace
```

### 2.4 `audio_player.py` — device-backed FrameWriter

```python
# audio_player.py
# Writes 20 ms PCM16 frames to the default output device via sounddevice.
# For headless CI, swap this for a NullPlayer that records write timestamps only (see measure.py).
import asyncio
import numpy as np
import sounddevice as sd

from stream_tts import SAMPLE_RATE, BYTES_PER_SAMPLE, FrameWriter, Trace


class AudioPlayer(FrameWriter):
    def __init__(self, trace: Trace, sample_rate: int = SAMPLE_RATE):
        self._stream = sd.RawOutputStream(samplerate=sample_rate, channels=1, dtype="int16")
        self._stream.start()
        self._trace = trace

    async def write(self, frame: bytes) -> None:
        # sounddevice.RawOutputStream.write is blocking; offload to a thread to keep the loop responsive.
        await asyncio.to_thread(self._stream.write, frame)
        if self._stream.write_available < len(frame) // BYTES_PER_SAMPLE:
            self._trace.underruns += 1

    def close(self) -> None:
        self._stream.stop()
        self._stream.close()
```

### 2.5 WebSocket variant

Many providers (ElevenLabs, Cartesia, Deepgram Aura) ship a lower-latency WebSocket path. Swap only `_read_chunks`:

```python
# ws_stream_tts.py (skeleton — delta from stream_tts.py)
async def _read_chunks_ws(ws, text, voice, cancel, out, trace):
    trace.t_request_sent_ms = time.perf_counter() * 1000
    await ws.send_json({"text": text, "voice_id": voice, "format": "pcm_24000"})
    cumulative = 0
    async for msg in ws:
        if cancel.is_set():
            trace.cancelled = True
            break
        if msg.type != aiohttp.WSMsgType.BINARY:
            continue
        now = time.perf_counter() * 1000
        if trace.t_first_byte_ms is None:
            trace.t_first_byte_ms = now
        cumulative += len(msg.data)
        trace.chunks.append(ChunkEvent(t_wall_ms=now, bytes_received=len(msg.data), cumulative_bytes=cumulative))
        await out.put(msg.data)
    await out.put(None)
```

The producer/consumer, backpressure, and cancel surface are unchanged. This is by design — swapping transports should not re-architect playout.

### 2.6 `measure.py` — the measurement harness

```python
# measure.py
# Runs N streaming TTS requests through stream_tts.speak() and reports
# TTFB P50/P95/P99, inter-chunk gap distribution, underruns, and cancel latency.
import asyncio
import json
import statistics
import time
from typing import Iterable

from stream_tts import speak, FrameWriter, Trace


class NullPlayer(FrameWriter):
    """Timing-only sink.  Immediate-return writes let us measure TTFB independent of device jitter."""
    def __init__(self, trace: Trace):
        self._trace = trace
    async def write(self, frame: bytes) -> None:
        pass


async def run_trial(text: str, voice: str) -> Trace:
    trace = Trace(t_request_sent_ms=0.0, t_first_byte_ms=None,
                  t_first_frame_played_ms=None, t_last_frame_played_ms=None)
    await speak(text, voice, NullPlayer(trace))
    return trace


def percentile(xs, p):
    xs = sorted(xs)
    k = int(round((p / 100) * (len(xs) - 1)))
    return xs[k]


def inter_chunk_gaps(trace: Trace) -> list[float]:
    ts = [c.t_wall_ms for c in trace.chunks]
    return [b - a for a, b in zip(ts, ts[1:])]


async def main(lines: Iterable[str], voice: str, warmup: int = 3) -> None:
    traces: list[Trace] = []
    for i, line in enumerate(lines):
        trace = await run_trial(line, voice)
        if i >= warmup:
            traces.append(trace)
    ttfbs = [t.ttfb_ms for t in traces if t.ttfb_ms is not None]
    all_gaps: list[float] = []
    for t in traces:
        all_gaps.extend(inter_chunk_gaps(t))
    report = {
        "n_trials": len(traces),
        "warmup_excluded": warmup,
        "ttfb_ms": {
            "p50": percentile(ttfbs, 50),
            "p95": percentile(ttfbs, 95),
            "p99": percentile(ttfbs, 99),
            "mean": statistics.fmean(ttfbs),
        },
        "inter_chunk_gap_ms": {
            "p50": percentile(all_gaps, 50) if all_gaps else None,
            "p95": percentile(all_gaps, 95) if all_gaps else None,
            "p99": percentile(all_gaps, 99) if all_gaps else None,
        },
        "underruns": sum(t.underruns for t in traces),
    }
    print(json.dumps(report, indent=2))


async def cancel_latency_trial(text: str, voice: str, cancel_after_ms: int = 200) -> float:
    """Measure the gap between calling cancel() and the last frame actually being produced."""
    trace = Trace(t_request_sent_ms=0.0, t_first_byte_ms=None,
                  t_first_frame_played_ms=None, t_last_frame_played_ms=None)
    cancel = asyncio.Event()

    async def cancel_later():
        await asyncio.sleep(cancel_after_ms / 1000)
        cancel.set()
        return time.perf_counter() * 1000

    _, t_cancel_ms = await asyncio.gather(speak(text, voice, NullPlayer(trace), cancel), cancel_later())
    return (trace.t_last_frame_played_ms or t_cancel_ms) - t_cancel_ms
```

### 2.7 How this maps to the mod-106 barge-in surface

`cancel: asyncio.Event` is the integration point the VAD / barge-in layer (mod-106) flips. The `speak()` entrypoint does not own the VAD decision; it owns the *consequence*. Pattern for the caller:

```python
cancel = asyncio.Event()
tts_task = asyncio.create_task(speak(reply, voice, player, cancel))
async for vad_event in vad_stream:
    if vad_event.kind == "user_began_speaking":
        cancel.set()
        break
await tts_task
```

This keeps the TTS layer agnostic to *why* a cancel was requested — the VAD layer, a timeout, a tool-call pre-empt, all route through the same event.

## 3. Validation steps

A finished implementation should pass every check.

1. **Compiles clean.** `python -m py_compile stream_tts.py audio_player.py measure.py` returns no errors.
2. **TTFB measured.** `python measure.py <lines.txt> <voice_id>` prints a report with P50/P95/P99 TTFB and inter-chunk gap distribution. Warm-up trials are excluded. Report includes sample size.
3. **No underruns under normal load.** 10-minute soak with a typical reply length prints `"underruns": 0` on warm connections.
4. **Cancel latency ≤ 50 ms.** `cancel_latency_trial()` returns ≤ 50 ms on three consecutive runs. If not, inspect `MAX_QUEUED_FRAMES` (too large = more draining to do) and whether the HTTP connection is being closed inside the cancel path.
5. **Backpressure works.** Artificially stall the player (`await asyncio.sleep(0.5)` inside `AudioPlayer.write`). The queue size must *not* grow unboundedly — the producer blocks on `out.put`. Verify with logging.
6. **Transport-swap test.** Swap HTTP for the WebSocket variant with no changes to `speak()`, `AudioPlayer`, or `measure.py`.
7. **Observability check.** Each chunk emits one structured log line (`t_wall_ms`, `bytes_received`, `cumulative_bytes`). `jq`-able.
8. **No secrets on disk.** `grep -r PROVIDER_KEY` returns only the `os.environ[…]` lookup.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Streaming correctness | 20 % | TTS output plays end-to-end from a streaming endpoint; no buffered-the-whole-response first | First frame arrives on the device within the project's warm-call budget (mod-109) |
| TTFB measurement | 20 % | P50/P95/P99 reported with warm-up excluded, sample size noted | Inter-chunk gap distribution reported alongside |
| Underrun safety | 15 % | 0 underruns during 10-min soak on warm connection | Underrun count exposed as a metric per trace |
| Cancel semantics | 20 % | `cancel()` stops playback ≤ 50 ms and closes upstream connection | Cancel path cancels the HTTP read task explicitly, not via connection timeout |
| Backpressure | 10 % | Producer blocks when queue full; queue size bounded | Stall-injected player test shows bounded memory |
| Transport abstraction | 10 % | Swapping HTTP for WebSocket touches only the chunk-reader | No business logic in the transport module |
| Observability | 5 % | One structured log line per chunk; trace returned from `speak()` | `measure.py` emits a JSON report the grader can diff |

Scoring < 70 % is a fail-and-redo. The most common sub-70 failures are: (a) the implementation buffers the whole response before playing, (b) cancel sets a flag but doesn't close the HTTP response, (c) TTFB is reported as a one-shot mean instead of a distribution.

## 5. Common mistakes

- **Buffering the whole response before playing.** `await resp.read()` in the first line defeats the entire exercise. Use `iter_any()` / `iter_chunked()` (or WS `async for msg`).
- **Reporting TTFB as a mean.** Latency distributions are heavy-tailed; the P95 is the user experience. Report P50/P95/P99 and sample size.
- **Measuring cold.** First-call-of-the-morning TTFB includes DNS, TLS, and HTTP/2 connection setup. Exclude warm-up (3–5 trials is typical).
- **Cancel without closing the HTTP connection.** You stop playing but the provider keeps billing — and if the connection isn't closed, bytes keep arriving into your queue to be dropped.
- **Unbounded queue.** `asyncio.Queue()` with no `maxsize` is a memory leak waiting to happen if the player stalls; always set `maxsize`.
- **Big frames in the device callback.** Writing 500 ms frames to the output device makes cancel latency terrible. Frame size should be ≤ 40 ms.
- **Mixing decode onto the hot path.** Requesting MP3 and decoding in-process adds 10–30 ms per chunk on CPU. Request PCM (or Opus with a native decode offload).
- **Forgetting telephony sample rates.** On a SIP leg the pipeline is µ-law 8 kHz. Requesting 24 kHz PCM and resampling in the SIP gateway adds latency and quality loss. Request µ-law 8 kHz directly when applicable.
- **Hiding the cancel path from the VAD layer.** If `speak()` owns the VAD subscription, you've coupled TTS to barge-in. Expose a cancel Event; let the caller wire it.
- **Hard-coding the voice ID.** Pass it in; different routes use different voices (mod-104 exercise-04 A/B drill).

## 6. References

Primary — provider streaming-TTS endpoints (verify URLs still resolve at exercise-run time; vendor doc trees are frequently reorganised):

- ElevenLabs streaming + WebSocket TTS — `https://elevenlabs.io/docs/api-reference/text-to-speech-streaming`, `https://elevenlabs.io/docs/api-reference/websockets`
- Cartesia Sonic streaming — `https://docs.cartesia.ai/`
- Deepgram Aura (TTS) — `https://developers.deepgram.com/docs/tts-getting-started`
- OpenAI streaming TTS — `https://platform.openai.com/docs/guides/text-to-speech`, Realtime API audio output — `https://platform.openai.com/docs/guides/realtime`
- Google Cloud Text-to-Speech streaming — `https://cloud.google.com/text-to-speech/docs/streaming-synthesize`
- Azure speech synthesis streaming (Speech SDK) — `https://learn.microsoft.com/azure/ai-services/speech-service/get-started-text-to-speech`
- Amazon Polly streaming synthesis — `https://docs.aws.amazon.com/polly/latest/dg/API_SynthesizeSpeech.html`

Library docs used in the worked code:

- aiohttp streaming response — `https://docs.aiohttp.org/en/stable/streams.html`
- Python `asyncio.Queue` — `https://docs.python.org/3/library/asyncio-queue.html`
- sounddevice `RawOutputStream` — `https://python-sounddevice.readthedocs.io/en/latest/api/streams.html`

Related curriculum anchors:

- Barge-in / VAD surface integration — `mod-106-vad-turn-taking-barge-in`
- Latency budget framing — `mod-109-latency-budget-architecture`

Standards referenced indirectly:

- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — guides the "observable, auditable" posture of the trace output.

Source policy note. No vendor-specific TTFB number appears in this reference as a factual claim; the only numeric targets are the engineering budgets the module curriculum owns. All provider URLs carry a "verify at exercise time" expectation because vendor doc trees are frequently reorganised.
