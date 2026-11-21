# exercise-03 — Ring Buffer and Chunking Layer for Streaming — Reference Solution

<!-- Source policy: the lesson file lives in the paired `voice-agent-engineer-learning` repo and is not reachable from this sandbox. The deliverables below are reconstructed from the exercise title plus the official references cited in §6. Any specific numeric claim is anchored to the spec or library-doc it was taken from in the row where it appears. -->

## 1. Solution overview

This is an **implementation exercise**. The learner builds the component every real-time voice-agent pipeline needs between the capture / network side (which hands you audio in whatever packet size it feels like) and the consumer side (which wants fixed-size chunks — 10, 20, 30 ms — aligned to the frame size of the next stage: VAD, codec, ASR). The deliverable is a small, well-tested ring-buffer + chunker that:

1. Accepts variable-sized PCM writes (producer: capture, decoder output, Opus decoder, Twilio Media Stream, etc.).
2. Emits fixed-size chunks to a consumer (ASR, VAD, Opus encoder).
3. Is safe to call concurrently from one producer thread and one consumer thread.
4. Has a clear, declared policy for **overflow** (producer faster than consumer) and **underflow** (consumer asks for a chunk before enough samples have arrived).

The reference artefact is a single file, `ring_buffer.py`, plus a unit-test file, `test_ring_buffer.py`, that covers the five correctness properties below. Both are below.

## 2. Worked implementation

### 2.1 Design decisions and declared policy

| Decision | Choice | Why |
|---|---|---|
| Storage | Pre-allocated `numpy.ndarray` of `int16` | Matches the on-the-wire PCM dtype; no per-call allocation. |
| Capacity | Power-of-two frames (default 2 s at 16 kHz = 32 768 samples → round up to 65 536) | Index wrap is a mask, not a modulo. |
| Concurrency | `threading.Lock` around read / write pointer updates | One producer thread + one consumer thread is the real-world case; `Lock` is sufficient. For cross-process, swap for `multiprocessing.Queue`. |
| Overflow policy | **Drop oldest** (overwrite head) and *return a drop count* from `write` | Voice-agent pipelines prefer to keep the recent audio and surface the loss; blocking the capture thread causes jitter. |
| Underflow policy | **Return `None`** from `read_chunk` when < `chunk_size` samples available | Consumer handles timing; the buffer does not generate silence (that is a VAD / jitter-buffer decision, not a ring-buffer decision — see mod-106 / mod-107). |
| Chunk size | Caller supplies; default 320 samples (20 ms at 16 kHz) | 20 ms is the Opus / WebRTC frame cadence (RFC 6716 §2.1.4). |
| Monotonic sample index | Yes, emitted alongside the chunk | Downstream alignment / drift detection needs it. |

### 2.2 `ring_buffer.py`

```python
# ring_buffer.py
# Reference solution for mod-102 exercise-03.
# Single-producer / single-consumer PCM ring buffer with fixed-size chunking.

from __future__ import annotations

from dataclasses import dataclass
from threading import Lock
from typing import Optional

import numpy as np


@dataclass(frozen=True)
class Chunk:
    """Fixed-size PCM chunk emitted by RingBuffer.read_chunk."""
    samples: np.ndarray        # int16, shape (chunk_size,)
    start_index: int           # monotonically increasing sample index of first sample


class RingBuffer:
    """Single-producer / single-consumer int16 PCM ring buffer with fixed-size chunking.

    Overflow policy: drop oldest. `write` returns the number of samples dropped so the
    caller can log / count underruns. The buffer never blocks the producer.

    Underflow policy: `read_chunk` returns None if fewer than `chunk_size` samples are
    available. The consumer decides whether to spin, back off, or inject silence —
    that decision belongs in the jitter buffer / VAD, not here.
    """

    def __init__(self, capacity_samples: int = 65_536) -> None:
        # Round up to the next power of two so wraparound is a bitmask.
        cap = 1
        while cap < capacity_samples:
            cap <<= 1
        self._cap = cap
        self._mask = cap - 1
        self._buf = np.zeros(cap, dtype=np.int16)
        self._write_idx = 0        # absolute, monotonically increasing
        self._read_idx = 0         # absolute, monotonically increasing
        self._lock = Lock()

    @property
    def capacity(self) -> int:
        return self._cap

    def _available_locked(self) -> int:
        return self._write_idx - self._read_idx

    def write(self, samples: np.ndarray) -> int:
        """Append samples. Returns the number of oldest samples dropped due to overflow.

        `samples` must be a 1-D np.int16 array. If you have float32 in [-1, 1], convert
        before calling — the ring buffer refuses to guess your scaling.
        """
        if samples.dtype != np.int16:
            raise TypeError(f"expected int16 PCM, got {samples.dtype}")
        if samples.ndim != 1:
            raise ValueError(f"expected 1-D, got shape {samples.shape}")

        n = samples.shape[0]
        if n == 0:
            return 0
        if n > self._cap:
            # Pathological: incoming block is bigger than the whole buffer.
            # Keep only the most recent `cap` samples; count the rest as dropped.
            dropped = n - self._cap
            samples = samples[-self._cap:]
            n = self._cap
        else:
            dropped = 0

        with self._lock:
            # Compute post-write occupancy and drop oldest if it would exceed capacity.
            future_occupancy = self._available_locked() + n
            if future_occupancy > self._cap:
                extra = future_occupancy - self._cap
                self._read_idx += extra   # advance read ptr; oldest `extra` samples are lost
                dropped += extra

            # Two-part copy because of wraparound.
            start = self._write_idx & self._mask
            end = start + n
            if end <= self._cap:
                self._buf[start:end] = samples
            else:
                head = self._cap - start
                self._buf[start:] = samples[:head]
                self._buf[: n - head] = samples[head:]
            self._write_idx += n

        return dropped

    def read_chunk(self, chunk_size: int) -> Optional[Chunk]:
        """Return one chunk of exactly `chunk_size` samples, or None if underflowed.

        Advances the read cursor only when a chunk was produced.
        """
        if chunk_size <= 0 or chunk_size > self._cap:
            raise ValueError(f"invalid chunk_size {chunk_size}")

        with self._lock:
            if self._available_locked() < chunk_size:
                return None

            start = self._read_idx & self._mask
            end = start + chunk_size
            if end <= self._cap:
                out = self._buf[start:end].copy()
            else:
                head = self._cap - start
                out = np.concatenate(
                    (self._buf[start:], self._buf[: chunk_size - head])
                )
            chunk = Chunk(samples=out, start_index=self._read_idx)
            self._read_idx += chunk_size
            return chunk

    def available(self) -> int:
        """Current occupancy in samples (approx — may change under concurrent writes)."""
        with self._lock:
            return self._available_locked()
```

### 2.3 `test_ring_buffer.py`

```python
# test_ring_buffer.py
import numpy as np
import threading
import time

from ring_buffer import RingBuffer


def _tone(n: int, start: int = 0) -> np.ndarray:
    # Monotonic ramp so we can assert ordering and loss precisely.
    return np.arange(start, start + n, dtype=np.int16)


def test_basic_write_then_read_exact_chunk():
    rb = RingBuffer(capacity_samples=1024)
    rb.write(_tone(320))
    c = rb.read_chunk(320)
    assert c is not None
    assert c.start_index == 0
    assert np.array_equal(c.samples, _tone(320))
    assert rb.read_chunk(320) is None  # underflow after draining


def test_underflow_returns_none_without_advancing():
    rb = RingBuffer(capacity_samples=1024)
    rb.write(_tone(100))
    assert rb.read_chunk(320) is None
    # The 100 samples are still available after the failed read.
    rb.write(_tone(220, start=100))
    c = rb.read_chunk(320)
    assert c is not None
    assert np.array_equal(c.samples, _tone(320))


def test_wraparound_integrity():
    rb = RingBuffer(capacity_samples=1024)   # pow2 → 1024
    rb.write(_tone(800))
    rb.read_chunk(640)                        # read ptr moved past half
    # Next write straddles the wrap point.
    rb.write(_tone(500, start=800))
    c = rb.read_chunk(320)
    assert c is not None
    assert c.start_index == 640
    assert np.array_equal(c.samples, _tone(320, start=640))


def test_overflow_drops_oldest_and_reports_count():
    rb = RingBuffer(capacity_samples=1024)   # pow2 → 1024
    dropped = rb.write(_tone(900))
    assert dropped == 0
    dropped = rb.write(_tone(300, start=900))  # 1200 total, cap=1024 → drop 176
    assert dropped == 176
    # The sample at absolute index 176 is now the head.
    c = rb.read_chunk(320)
    assert c is not None
    assert c.start_index == 176
    assert np.array_equal(c.samples, _tone(320, start=176))


def test_oversized_single_write_is_truncated_from_the_front():
    rb = RingBuffer(capacity_samples=1024)
    # 3 000 samples into a 1 024-cap buffer: last 1 024 survive; 1 976 dropped.
    dropped = rb.write(_tone(3_000))
    assert dropped == 1_976
    c = rb.read_chunk(1_024)
    assert c is not None
    assert c.start_index == 1_976
    assert np.array_equal(c.samples, _tone(1_024, start=1_976))


def test_single_producer_single_consumer_concurrency():
    rb = RingBuffer(capacity_samples=4096)
    total_in = 50_000
    produced_sum = 0
    consumed = []

    def produce():
        nonlocal produced_sum
        rng = np.random.default_rng(seed=0)
        remaining = total_in
        while remaining:
            n = int(min(remaining, rng.integers(50, 1200)))
            block = _tone(n, start=total_in - remaining)
            rb.write(block)
            produced_sum += int(block.sum())
            remaining -= n
            time.sleep(0.0001)

    def consume():
        while True:
            c = rb.read_chunk(160)
            if c is None:
                time.sleep(0.0005)
                continue
            consumed.append(c)
            if c.start_index + 160 >= total_in:
                break

    tp = threading.Thread(target=produce)
    tc = threading.Thread(target=consume)
    tp.start(); tc.start()
    tp.join(timeout=10); tc.join(timeout=10)

    # Monotonic start_index.
    for i in range(1, len(consumed)):
        assert consumed[i].start_index == consumed[i - 1].start_index + 160

    # No drops because capacity is comfortably larger than producer-consumer skew.
    assert rb.available() < 160
```

### 2.4 How this plugs into a voice-agent pipeline

```text
Twilio Media Stream (μ-law, 160 samples / 20 ms frame)
    │  (decode μ-law → int16, upsample 8 k → 16 k)
    ▼
[ RingBuffer, cap=32 k samples ]           <-- this exercise
    │  read_chunk(320)  // 20 ms at 16 kHz
    ▼
VAD + ASR streaming client (mod-103 / mod-106)
```

The ring buffer is deliberately dumb. It knows nothing about VAD, nothing about Opus, nothing about network jitter — it just absorbs variable write sizes and emits fixed read sizes. All smartness about "should I play silence? should I resync?" lives in the layer above.

## 3. Validation steps

A finished submission should pass all of the following:

1. **Unit tests green.** `pytest test_ring_buffer.py` passes on all six tests.
2. **Overflow policy declared.** The implementation surfaces a drop count or raises — the policy is named, not left as "undefined behaviour".
3. **Underflow policy declared.** `read_chunk` returns a sentinel (`None`, zero-length, etc.) and does not advance the read pointer; the policy is named.
4. **No per-call allocation on the hot path.** The ring buffer pre-allocates its backing `np.ndarray`; `write` and `read_chunk` do not grow Python lists in the fast case. Chunk output is a `.copy()` so the consumer can't be corrupted by the next writer.
5. **Monotonic sample index.** Every emitted chunk carries a `start_index` that is contiguous (`prev.start_index + chunk_size == next.start_index`) when no overflow has occurred, and is monotonically increasing when overflow does occur.
6. **Wraparound tested.** At least one test writes across the wrap boundary and reads a chunk that straddles it.
7. **Concurrency tested.** At least one test runs a producer and a consumer on separate threads; no assertion fires.
8. **Typed PCM.** `write` rejects dtypes other than `int16` (or declares the dtype explicitly) — the buffer does not silently coerce float32 to int16.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Correctness under single-thread use | 25 % | Basic write / read / wraparound tests pass | Property-based test (Hypothesis) covers random sequences |
| Overflow / underflow policy | 20 % | Policy declared and visible in a dropped-count or sentinel return | Drop metric is surfaced (callback or counter) rather than only returned |
| Concurrency safety | 20 % | One-producer / one-consumer test passes without data corruption | Second test with two consumers (should be rejected explicitly or shown to be safe) |
| No per-call allocation | 10 % | Backing array pre-allocated; no list growth | Chunk returned as a view with a copy flag, documenting the cost / safety trade |
| Monotonic index + chunk metadata | 10 % | Each chunk carries `start_index` | Also carries wall-clock timestamp for jitter metrics |
| Integration note | 15 % | A 1-paragraph note describing where this sits in the mod-102 → mod-103 / mod-106 flow | Diagram + a worked bytes-per-second / frames-per-second number |

Submissions below 70 % typically fail on either overflow policy (silently dropping, no counter) or on the wraparound test.

## 5. Common mistakes

- **Using a Python `list` or `collections.deque` with `maxlen`.** It "works" but allocates every call and makes the numpy / Opus / ASR consumer pay for a copy. The deliverable is specifically a ring buffer over a fixed `np.ndarray`.
- **Blocking on overflow.** Blocking the capture thread causes the audio device to miss frames and jitter spreads through the whole pipeline. The lesson's preferred policy is drop-oldest + report.
- **Zero-fill on underflow.** Injecting silence inside the ring buffer makes the consumer think it received real samples and poisons the VAD / jitter buffer. Return `None`; let the consumer decide.
- **Non-power-of-two capacity.** Non-pow2 works, but the wraparound code uses modulo, which is slower and more error-prone than a bitmask. Round up to the next power of two.
- **Mutable chunk output.** If `read_chunk` returns a view of the internal buffer, the next writer will mutate the consumer's data. Always `.copy()` on emit (or document the lifetime constraint).
- **Floating-point PCM.** The buffer is dtype-opinionated for a reason — speech pipelines operate in int16 PCM on the wire and in float32 inside models; conflating the two is the single biggest source of "the audio sounds half-volume" bugs.
- **Globally mutable sample index.** `start_index` must be local to the buffer instance (it's a per-stream monotonic). Making it a module-level global causes subtle cross-stream corruption.
- **No test for the wrap boundary.** A buffer that doesn't test the boundary case is a buffer with an unknown bug in the boundary case.

## 6. References

Primary — official project documentation and standards:

- **Python `threading` module** — https://docs.python.org/3/library/threading.html (Lock semantics for single-producer / single-consumer).
- **NumPy array semantics** — https://numpy.org/doc/stable/reference/arrays.ndarray.html (strided layout, views vs. copies).
- **RFC 6716 §2.1.4** — Opus frame sizes (2.5 / 5 / 10 / 20 / 40 / 60 ms), which is where the 20 ms default chunk size comes from. https://datatracker.ietf.org/doc/html/rfc6716#section-2.1.4 .
- **WebRTC Opus Payload Format (RFC 7587)** — 20 ms PTime default. https://datatracker.ietf.org/doc/html/rfc7587 .

Reference implementations (**practitioner examples**, not standards):

- **Pipecat** `pipecat-ai` audio frame handling — https://github.com/pipecat-ai/pipecat (framework-level chunking + jitter handling for voice agents; informative, not authoritative).
- **LiveKit Agents** — https://github.com/livekit/agents (audio-stream and frame-queue primitives).
- **VeriSwarm practitioner examples** — treat as implementation reference only, per source policy.

Local curriculum anchors:

- `mod-106-vad-turn-taking-barge-in` — the first consumer of the chunk stream (per-frame VAD decisions).
- `mod-107-telephony-webrtc-transport` — the typical producer side (Twilio Media Streams / WebRTC frames).
- `mod-109-latency-budget-architecture` — where the ring-buffer depth (cap = N ms) is traded against end-to-end latency.

Source policy note. OWASP ML Top 10, MITRE ATLAS, and NIST AI RMF are not load-bearing here; this is a data-structures / concurrency exercise.
