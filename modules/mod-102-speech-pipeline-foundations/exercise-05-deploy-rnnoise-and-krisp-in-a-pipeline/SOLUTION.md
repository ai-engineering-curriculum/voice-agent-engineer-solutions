# exercise-05 — Deploy RNNoise (and Evaluate Krisp) in a Pipeline — Reference Solution

<!-- Source policy: the lesson file lives in the paired `voice-agent-engineer-learning` repo and is not reachable from this sandbox. The deliverables below are reconstructed from the exercise title plus the official references cited in §6. Any specific numeric claim is anchored to the upstream project or paper it was taken from where it appears. -->

## 1. Solution overview

This exercise has two halves and the learner should visibly do both:

1. **Deploy RNNoise** — insert the open-source, Mozilla / Xiph RNNoise denoiser into a streaming voice-agent pipeline, and demonstrate it with a working example. RNNoise is open source (BSD), ships as a small C library (`librnnoise`) with Python bindings available, and is realtime by construction — it runs on tiny frames with negligible CPU cost. This is the "do it" half.
2. **Evaluate Krisp** — Krisp is a *closed-source commercial* noise-cancellation SDK with native + WebRTC integrations. The learner cannot deploy it without a vendor key and (in production) a commercial agreement; they can and should propose an **evaluation protocol** that would let them decide whether to add Krisp to a pipeline that already has RNNoise. This is the "decide" half.

The reference artefacts are:

- A Python script that wraps RNNoise for 10 ms frames at 48 kHz and returns the denoised stream.
- A set of objective metrics (PESQ, STOI, SI-SDR) used to score before / after on a held-out noisy-speech test set.
- A written evaluation protocol for Krisp with the pass / fail decision criteria.
- The reviewer rubric and the common mistakes.

## 2. Worked implementation

### 2.1 Where the denoiser sits in the pipeline

```text
Capture (48 kHz Opus or 8 kHz μ-law)
    │
    ▼
Decode + resample to 48 kHz int16          <-- RNNoise's declared operating rate
    │
    ▼
Ring buffer (mod-102 exercise-03)
    │   emits 480-sample frames (10 ms @ 48 kHz)
    ▼
[ RNNoise denoiser ]                       <-- this exercise
    │   emits 480-sample denoised frames + per-frame VAD prob
    ▼
Downsample to 16 kHz  →  VAD / ASR (mod-103 / mod-106)
```

Two engineering facts that pin this placement:

- **RNNoise is 48 kHz-native.** The upstream paper and the `rnnoise.h` header define a frame size of **480 samples at 48 kHz = 10 ms** (`RNNOISE_FRAME_SIZE 480`). You do not feed it 16 kHz audio; you resample to 48 kHz first, denoise, then resample to whatever the next stage wants.
- **RNNoise emits a VAD probability as a by-product.** `rnnoise_process_frame` returns a float in `[0, 1]`. You can log it, but do **not** replace your main VAD with it — the model was trained on specific noise / speech distributions and is less reliable than a dedicated VAD (see mod-106).

### 2.2 Reference script (`denoise_pipeline.py`)

```python
# denoise_pipeline.py
# Reference solution for mod-102 exercise-05.
# Wraps RNNoise for a 48 kHz / 10 ms streaming pipeline.
# Requires: pip install "rnnoise-wrapper" numpy soundfile librosa

from __future__ import annotations

import numpy as np
import soundfile as sf
import librosa

# RNNoise bindings exist in several Python packages; this solution uses a
# ctypes-level wrapper to make the FFI contract explicit. Replace with the
# Python binding of your choice (e.g., `rnnoise-wrapper`, `py-rnnoise`).
from rnnoise_wrapper import RNNoise

TARGET_SR = 48_000
FRAME_SAMPLES = 480            # 10 ms at 48 kHz, per rnnoise.h


def load_as_48k_int16(path: str) -> np.ndarray:
    y, sr = sf.read(path, dtype="int16", always_2d=False)
    if y.ndim > 1:
        y = y.mean(axis=1).astype(np.int16)
    if sr != TARGET_SR:
        yf = y.astype(np.float32) / 32768.0
        yf = librosa.resample(yf, orig_sr=sr, target_sr=TARGET_SR)
        y = np.clip(yf * 32768.0, -32768, 32767).astype(np.int16)
    return y


def chunked(a: np.ndarray, n: int):
    for i in range(0, len(a) - n + 1, n):
        yield a[i : i + n]


def denoise_stream(y: np.ndarray) -> tuple[np.ndarray, np.ndarray]:
    """Streaming denoiser: returns (clean_int16, vad_probs_per_frame).

    Pads the tail with zeros so the last partial frame still gets processed.
    """
    denoiser = RNNoise()
    out: list[np.ndarray] = []
    vads: list[float] = []

    tail = len(y) % FRAME_SAMPLES
    if tail:
        y = np.concatenate([y, np.zeros(FRAME_SAMPLES - tail, dtype=np.int16)])

    for frame in chunked(y, FRAME_SAMPLES):
        vad_prob, clean = denoiser.process_frame(frame)
        out.append(clean)
        vads.append(vad_prob)

    return np.concatenate(out), np.array(vads, dtype=np.float32)


if __name__ == "__main__":
    import sys
    in_path, out_path = sys.argv[1], sys.argv[2]
    noisy = load_as_48k_int16(in_path)
    clean, vads = denoise_stream(noisy)
    sf.write(out_path, clean, TARGET_SR, subtype="PCM_16")
    print(f"frames={len(vads)}  mean_vad_prob={vads.mean():.3f}")
```

### 2.3 Objective evaluation (before vs after)

For each test clip, compute three objective metrics on the clean reference, the noisy input, and the denoised output:

| Metric | What it measures | What to report |
|---|---|---|
| **PESQ** (ITU-T P.862 for narrowband / P.862.2 for wideband) | Perceived speech quality, MOS-like scale (~1.0 – ~4.5) | `delta_PESQ = PESQ(denoised, clean) - PESQ(noisy, clean)`. Positive means denoising helped. |
| **STOI** / **ESTOI** | Short-time objective intelligibility, 0 – 1 | `delta_STOI` same way. Intelligibility sometimes drops even when PESQ rises; report both. |
| **SI-SDR** (dB) | Scale-invariant signal-to-distortion ratio | `delta_SI_SDR`. Positive = cleaner signal. |

Minimum test set: 20 clips (10 s each) across 4 noise types × 3 SNRs. The learner should name a public source (e.g., VoiceBank+DEMAND, MS-SNSD) and not fabricate clips.

```python
# eval_denoise.py (sketch)
from pesq import pesq            # pip install pesq
from pystoi import stoi          # pip install pystoi
import numpy as np

def si_sdr(reference, estimate):
    reference = reference - reference.mean()
    estimate = estimate - estimate.mean()
    s_target = np.dot(estimate, reference) / np.dot(reference, reference) * reference
    e_noise = estimate - s_target
    return 10 * np.log10(np.sum(s_target ** 2) / np.sum(e_noise ** 2))

def score(clean_16k, noisy_16k, denoised_16k):
    return dict(
        pesq_noisy=pesq(16_000, clean_16k, noisy_16k, "wb"),
        pesq_denoised=pesq(16_000, clean_16k, denoised_16k, "wb"),
        stoi_noisy=stoi(clean_16k, noisy_16k, 16_000, extended=False),
        stoi_denoised=stoi(clean_16k, denoised_16k, 16_000, extended=False),
        sdr_noisy=si_sdr(clean_16k, noisy_16k),
        sdr_denoised=si_sdr(clean_16k, denoised_16k),
    )
```

Report a 2-column table across the test set: median + 10th/90th percentiles of each `delta`. Don't report means — the distribution is heavy-tailed on hard noise types.

### 2.4 Krisp evaluation protocol (black-box, no deploy)

Krisp is closed-source and distributed as an SDK with per-seat or per-minute commercial terms. The deliverable here is a **protocol**, not a deployment.

A defensible evaluation protocol:

1. **Define the decision criteria** before running anything. Example: "we will add Krisp only if, at equal CPU / latency budget, it beats RNNoise by `> 0.3` PESQ (wideband) median on our test set and does not degrade ASR WER on the same set by more than 1 absolute point."
2. **Hold out the eval set.** The same 20 clips × 4 noise types × 3 SNRs used for RNNoise, plus 5 real captures from actual customer calls (consent-reviewed; see mod-108).
3. **Compare at the same insertion point.** RNNoise and Krisp both sit in the pre-ASR post-decode stage, operating on decoded 48 kHz PCM. Do not compare RNNoise-before-codec against Krisp-after-codec — that's a confounded test.
4. **Measure four axes**: perceptual quality (PESQ / STOI / SI-SDR), downstream ASR WER on the full clip, CPU cost per frame (steady-state), and wall-clock latency added per frame. Any one of these worse than RNNoise is a reason to pause.
5. **Trial under the licence.** Krisp provides a developer trial with the SDK; the eval runs under that licence. Do not reverse-engineer or redistribute the binary — this is both a licence violation and a mod-108 / NIST AI RMF "govern" red flag (third-party component with unclear provenance).
6. **Decide and document.** Produce a one-page memo with the numbers, the decision, and the specific versioned SDK you tested. Re-run quarterly; closed-source denoiser quality drifts silently as the vendor ships new weights.

### 2.5 Deployment note — RNNoise in realtime

RNNoise is cheap enough that it can sit inline in the hot path (10 ms frames, tens of microseconds per frame on a modern x86 core). Two gotchas:

- **It does not adapt to new speakers / rooms on its own.** The model is small and pretrained; if your product sees unusual acoustics (call-centre booths, wind on a lavalier, in-car cabin noise) it will under-suppress. Evaluate per acoustic condition, not just average.
- **It does not do echo cancellation.** RNNoise suppresses stationary + modulated noise; it does not remove the agent's own TTS leaking from the speaker into the mic. For that you need an AEC (WebRTC AEC3, Speex AEC). AEC and denoise are different components in the same pipeline, not substitutes.

## 3. Validation steps

A finished submission should pass all of the following:

1. **RNNoise runs end-to-end.** The reference script converts a noisy WAV to a denoised WAV at 48 kHz without crashing and without producing clicks at frame boundaries.
2. **Frame rate is correct.** The implementation processes 10 ms / 480-sample frames at 48 kHz (not 16 kHz, not 20 ms) — matching `rnnoise.h`.
3. **Before / after scored on a documented test set.** The chosen test set is named (VoiceBank+DEMAND, MS-SNSD, or a declared local capture with consent) and not fabricated.
4. **At least three metrics reported.** PESQ (wideband or narrowband, declared), STOI / ESTOI, SI-SDR. Means and percentiles reported, not just one number.
5. **ASR WER co-measured.** Denoising that improves PESQ but degrades ASR WER is a net loss for a voice agent; the submission runs a baseline ASR (Whisper-tiny or similar) over noisy and denoised clips and reports both.
6. **Krisp protocol is a protocol, not a deployment.** The learner describes an evaluation plan; they do not claim to have deployed Krisp without a licence.
7. **Decision criteria stated before results.** Preregistration-style: "we'd adopt Krisp if ΔPESQ > 0.3 AND ΔWER ≥ 0" named in the protocol, not retrofitted after seeing numbers.
8. **AEC vs denoise distinguished.** Submission states that RNNoise is not an echo canceller and names where the AEC sits in the pipeline.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| RNNoise correctly deployed | 20 % | 48 kHz / 10 ms frames; realtime code path; no boundary artefacts | Also implemented via native `librnnoise` + `ctypes` with explicit buffer ownership |
| Metrics coverage | 20 % | PESQ + STOI + SI-SDR reported with distribution, not just means | Also reports ΔMOS-LQO via ITU-T P.863 |
| Test set documentation | 10 % | Named public set or documented consent-reviewed local captures | ≥ 4 noise types × ≥ 3 SNRs with per-cell reporting |
| ASR WER co-measurement | 15 % | Baseline ASR WER on noisy vs denoised reported | Covers at least two ASR systems (Whisper + Deepgram) |
| Krisp evaluation protocol | 20 % | Decision criteria pre-declared; black-box evaluation; licence respected | Protocol includes per-leg (PSTN μ-law vs WebRTC Opus) stratification |
| AEC callout | 10 % | AEC vs denoise distinction made | AEC chain shown in the diagram with its own evaluation plan |
| Common-pitfalls callout | 5 % | At least 3 of §5 mistakes mentioned | Links the AEC + denoise + VAD interaction into the latency budget (mod-109) |

Submissions under 70 % typically fail on either the frame rate (feeding 16 kHz audio into RNNoise) or on the Krisp half (claiming to have deployed Krisp without explaining the licence story).

## 5. Common mistakes

- **Feeding 16 kHz audio into RNNoise.** RNNoise is 48 kHz-native per `rnnoise.h`; feeding 16 kHz produces garbage that sometimes sounds like "it worked a little" because the output envelope is roughly right. Resample first.
- **Wrong frame size.** `RNNOISE_FRAME_SIZE 480` is 10 ms at 48 kHz, not 20 ms, not 480 at 16 kHz. Mis-sizing the frame breaks the internal state.
- **Treating RNNoise's VAD probability as the system VAD.** It is a side output, trained on the model's own clean / noise distribution. It is not a drop-in replacement for Silero / WebRTC VAD (mod-106).
- **Confusing denoise with AEC.** RNNoise removes noise; it does not cancel the agent's own TTS loopback. Adding RNNoise with no AEC in a speakerphone product is the classic "the agent interrupts itself" bug.
- **Reporting only means.** Noise suppression metrics are heavy-tailed. Report medians + 10th/90th percentiles stratified by noise type.
- **Running Krisp without a licence.** The SDK requires a developer / commercial key. "I compiled an older open binary" is a licence violation and a mod-108 compliance red flag; don't.
- **Skipping the ASR co-measure.** Many denoisers improve subjective quality while removing cues the ASR relied on (stop consonants, fricatives). A denoise step that drops ASR accuracy is a regression for a voice agent, even if a human rater prefers the audio.
- **Not holding out a real-call set.** Public test sets (MS-SNSD, VoiceBank+DEMAND) are useful but do not include your product's actual acoustics. Keep a small, consent-reviewed real-call set as the ground truth.
- **Picking the denoiser before the eval.** The decision criterion should be declared before the numbers are seen. Picking the winner first and backfilling the metric is `p < 0.05`-hacking the stack.

## 6. References

Primary — project documentation and standards:

- **RNNoise** — upstream project (Xiph / Mozilla / Jean-Marc Valin). Code: https://github.com/xiph/rnnoise . Demo page: https://jmvalin.ca/demo/rnnoise/ . 48 kHz / 10 ms frame constants: `rnnoise.h` (`RNNOISE_FRAME_SIZE 480`).
- **J.-M. Valin, "A Hybrid DSP/Deep Learning Approach to Real-Time Full-Band Speech Enhancement,"** 2018 IEEE 20th International Workshop on Multimedia Signal Processing. arXiv preprint: https://arxiv.org/abs/1709.08243 . The paper behind RNNoise.
- **ITU-T Recommendation P.862 / P.862.2** — PESQ (narrowband / wideband). https://www.itu.int/rec/T-REC-P.862/ .
- **ITU-T Recommendation P.863** — POLQA, the successor to PESQ. https://www.itu.int/rec/T-REC-P.863/ .
- **Short-Time Objective Intelligibility (STOI) / ESTOI** — Taal et al., "A short-time objective intelligibility measure for time-frequency weighted noisy speech." Reference implementation: https://github.com/mpariente/pystoi (practitioner).
- **SI-SDR** — Le Roux et al., "SDR — half-baked or well done?" arXiv: https://arxiv.org/abs/1811.02508 .

Datasets commonly used for this exercise:

- **VoiceBank+DEMAND** — https://datashare.ed.ac.uk/handle/10283/2791 (University of Edinburgh).
- **MS-SNSD** — Microsoft Scalable Noisy Speech Dataset — https://github.com/microsoft/MS-SNSD .

Closed-source product (for the evaluation half):

- **Krisp SDK** — https://krisp.ai/ (commercial SDK; developer trial available under vendor terms). Treat as black-box; do not assume technical details.

Adjacent pipeline components (not substitutes for RNNoise):

- **WebRTC AEC3** — acoustic echo canceller in the WebRTC audio-processing module: https://webrtc.googlesource.com/src/+/refs/heads/main/modules/audio_processing/README.md .
- **Speex AEC** — alternative AEC used in some stacks: https://www.speex.org/ .

Local curriculum anchors:

- `mod-106-vad-turn-taking-barge-in` — why RNNoise's VAD probability is not the system VAD.
- `mod-108-safety-compliance-voice` — provenance / licence story for third-party SDKs (NIST AI RMF Govern function).
- `mod-109-latency-budget-architecture` — where the denoise stage costs; how much latency the ASR / VAD / AEC chain adds in total.
- `mod-112-voice-multimodal-eval-and-observability` — the broader eval framework this exercise feeds into.

Source policy note. NIST AI RMF applies lightly here (third-party-component governance for the Krisp half); OWASP ML Top 10 and MITRE ATLAS are not load-bearing for this exercise.
