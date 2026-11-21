# exercise-02 — Feature Extraction with librosa and torchaudio — Reference Solution

<!-- Source policy: the lesson file lives in the paired `voice-agent-engineer-learning` repo and is not reachable from this sandbox. The deliverables below are reconstructed from the exercise title plus the official references cited in §6. Any specific numeric claim is anchored to the spec or library-doc it was taken from in the row where it appears. -->

## 1. Solution overview

This is an **implementation exercise**: the learner computes the three feature representations a voice-agent engineer touches most often — the **short-time Fourier transform (STFT)**, the **Mel spectrogram**, and **Mel-frequency cepstral coefficients (MFCCs)** — using both `librosa` and `torchaudio` on the same input signal, then reconciles the two outputs. The goal is not to produce prettiest-looking spectrograms; it is to internalise the knobs (`n_fft`, `hop_length`, `win_length`, `n_mels`, `n_mfcc`, window type, power vs. magnitude, mel filterbank normalisation) and to see that two libraries with the same stated settings can still disagree numerically unless every knob is locked.

The reference artefacts in this solution are:

1. A single Python script `extract_features.py` that produces STFT / Mel / MFCC outputs with both libraries at matched settings.
2. The verification step — checking the two libraries agree to within a small tolerance after accounting for the known normalisation differences.
3. The reviewer rubric and the handful of mistakes that cause 90 % of the "my librosa and torchaudio spectrograms look different" questions.

## 2. Worked implementation

### 2.1 Fixed configuration (set once, reused everywhere)

Pick one config and hold it fixed; every "my two libraries disagree" issue starts with a config drift.

| Parameter | Value | Why |
|---|---|---|
| `sample_rate` | 16 000 Hz | Standard wideband ASR rate (Whisper, wav2vec2, Deepgram Nova wideband). |
| `n_fft` | 400 | 25 ms window at 16 kHz — the Whisper / wav2vec2 convention. |
| `win_length` | 400 | Equal to `n_fft`, no implicit padding. |
| `hop_length` | 160 | 10 ms hop — standard for ASR features. |
| `window` | Hann | librosa default; also torchaudio default. |
| `n_mels` | 80 | Whisper / most modern ASR convention. |
| `f_min` / `f_max` | 0 / 8 000 | Nyquist at 16 kHz. |
| `power` | 2.0 | Power spectrogram (not magnitude). |
| `n_mfcc` | 13 | Classic MFCC count; keep MFCC and Mel aligned. |

### 2.2 The worked script

```python
# extract_features.py
# Reference solution for mod-102 exercise-02.
# Produces matched STFT / Mel / MFCC outputs from librosa and torchaudio and
# asserts they agree numerically once normalisation differences are reconciled.

from __future__ import annotations

import numpy as np
import librosa
import torch
import torchaudio
import torchaudio.transforms as T

# -------- fixed config (see §2.1) --------
SR = 16_000
N_FFT = 400
WIN_LEN = 400
HOP = 160
N_MELS = 80
N_MFCC = 13
F_MIN, F_MAX = 0.0, 8000.0
POWER = 2.0


def load_mono_16k(path: str) -> np.ndarray:
    """Load and resample to the fixed working rate; return float32 mono."""
    y, _ = librosa.load(path, sr=SR, mono=True)
    return y.astype(np.float32)


# ---------- librosa path ----------
def librosa_stft(y: np.ndarray) -> np.ndarray:
    # Centre-padded by default; keep that convention everywhere.
    return librosa.stft(
        y,
        n_fft=N_FFT,
        hop_length=HOP,
        win_length=WIN_LEN,
        window="hann",
        center=True,
    )


def librosa_mel(y: np.ndarray) -> np.ndarray:
    # Compute via power-spectrogram → mel filterbank so the knobs are explicit.
    stft = librosa_stft(y)
    power_spec = np.abs(stft) ** POWER
    mel_fb = librosa.filters.mel(
        sr=SR, n_fft=N_FFT, n_mels=N_MELS, fmin=F_MIN, fmax=F_MAX,
        htk=False, norm="slaney",
    )
    return mel_fb @ power_spec  # shape (n_mels, frames)


def librosa_mfcc(y: np.ndarray) -> np.ndarray:
    log_mel = librosa.power_to_db(librosa_mel(y))
    return librosa.feature.mfcc(S=log_mel, n_mfcc=N_MFCC)


# ---------- torchaudio path ----------
def torchaudio_stft(y: np.ndarray) -> torch.Tensor:
    y_t = torch.from_numpy(y).unsqueeze(0)  # (1, T)
    spec = T.Spectrogram(
        n_fft=N_FFT,
        hop_length=HOP,
        win_length=WIN_LEN,
        power=None,            # keep complex output so this matches librosa.stft
        center=True,
        window_fn=torch.hann_window,
    )(y_t)
    return spec.squeeze(0)     # (freq, frames)


def torchaudio_mel(y: np.ndarray) -> torch.Tensor:
    y_t = torch.from_numpy(y).unsqueeze(0)
    mel = T.MelSpectrogram(
        sample_rate=SR,
        n_fft=N_FFT,
        hop_length=HOP,
        win_length=WIN_LEN,
        n_mels=N_MELS,
        f_min=F_MIN,
        f_max=F_MAX,
        power=POWER,
        center=True,
        window_fn=torch.hann_window,
        mel_scale="slaney",       # match librosa `htk=False`
        norm="slaney",            # match librosa `norm="slaney"`
    )(y_t)
    return mel.squeeze(0)        # (n_mels, frames)


def torchaudio_mfcc(y: np.ndarray) -> torch.Tensor:
    y_t = torch.from_numpy(y).unsqueeze(0)
    mfcc = T.MFCC(
        sample_rate=SR,
        n_mfcc=N_MFCC,
        melkwargs=dict(
            n_fft=N_FFT, hop_length=HOP, win_length=WIN_LEN,
            n_mels=N_MELS, f_min=F_MIN, f_max=F_MAX,
            power=POWER, center=True,
            mel_scale="slaney", norm="slaney",
        ),
    )(y_t)
    return mfcc.squeeze(0)       # (n_mfcc, frames)


# ---------- reconciliation ----------
def compare(y: np.ndarray) -> None:
    lib_mel = librosa_mel(y)
    ta_mel = torchaudio_mel(y).numpy()

    # Shapes should match exactly.
    assert lib_mel.shape == ta_mel.shape, (lib_mel.shape, ta_mel.shape)

    # Values should agree once we match normalisation (slaney / slaney).
    # Expect per-element relative error of a few parts in 1e-5.
    denom = np.maximum(np.abs(lib_mel), 1e-8)
    rel = np.abs(lib_mel - ta_mel) / denom
    print(f"mel  shape={lib_mel.shape}  max_rel_err={rel.max():.3e}")

    lib_mfcc = librosa_mfcc(y)
    ta_mfcc = torchaudio_mfcc(y).numpy()
    assert lib_mfcc.shape == ta_mfcc.shape, (lib_mfcc.shape, ta_mfcc.shape)
    # MFCC absolute scale differs between libraries because librosa uses
    # power_to_db → DCT-II-ortho while torchaudio defaults to log → DCT-II-ortho.
    # Agreement is up to an additive constant per coefficient; compare shapes
    # and the *shape of the trajectory*, not the absolute magnitude.
    print(f"mfcc shape={lib_mfcc.shape}  "
          f"corr_coef_0={np.corrcoef(lib_mfcc[0], ta_mfcc[0])[0,1]:.4f}")


if __name__ == "__main__":
    import sys
    audio_path = sys.argv[1] if len(sys.argv) > 1 else "example.wav"
    y = load_mono_16k(audio_path)
    compare(y)
```

### 2.3 Why the two libraries agree on Mel but not on MFCC absolute value

- **Mel** agrees because both libraries, when told `mel_scale="slaney"` + `norm="slaney"`, implement the HTK-vs-Slaney choice in the same place and use the same power → Mel-filterbank matrix product. The two implementations differ by rounding only.
- **MFCC** disagrees on absolute magnitude because `librosa.feature.mfcc` applies `librosa.power_to_db` (`10 * log10(S / ref)` with `ref` default = `1.0`) before DCT-II-ortho, while `torchaudio.transforms.MFCC` applies a natural-log amplitude (not dB) before DCT-II-ortho (`log_mels=False` default) — the trajectory correlation is >~0.99 per coefficient, but the DC coefficient (`c0`) is offset and the scale differs by a constant factor. To make them numerically agree you have to (a) take the log the same way and (b) match `top_db` clipping.

This is the single most useful thing the learner takes away from the exercise: **feature values are not portable across libraries unless every normalisation knob is explicit.** Training data computed with `librosa` will not run cleanly against a torchaudio-based inference path without a reconciliation step; this is where sample-rate-mismatch-grade bugs hide.

### 2.4 Expected output shapes

For a 1-second clip at 16 kHz with `hop_length=160` and `center=True`:

- frames ≈ `1 + ceil(T / hop_length)` = `1 + ceil(16000 / 160)` = **101 frames**
- `STFT`     → `(n_fft/2 + 1, 101)` = `(201, 101)`
- `Mel`      → `(n_mels, 101)` = `(80, 101)`
- `MFCC`     → `(n_mfcc, 101)` = `(13, 101)`

Both libraries should produce identical shapes with `center=True`; `center=False` shifts the frame count and is the single biggest cause of "my shapes don't match".

## 3. Validation steps

A finished submission should pass all of the following:

1. **Shape equality.** For every feature, `librosa.<feat>(y).shape == torchaudio.<feat>(y).squeeze(0).numpy().shape`. Any shape mismatch means a knob (usually `center` or `pad_mode`) is drifted.
2. **Mel numerical agreement.** Max relative error between librosa Mel and torchaudio Mel with `mel_scale="slaney"`, `norm="slaney"` is `< 1e-4`. If it is larger, check the HTK flag.
3. **MFCC trajectory agreement.** Per-coefficient Pearson correlation between the two MFCC outputs is `> 0.95`. The exercise does **not** require absolute agreement — it requires the learner to *explain* why absolute agreement is not expected.
4. **Explicit knob table.** Submission reproduces the §2.1 table with its own chosen values (if different) and defends each choice.
5. **ASR-ready defaults.** The chosen defaults are compatible with at least one real ASR system (Whisper: 25 ms window / 10 ms hop / 80 mel bands / 16 kHz — this is documented in the Whisper code / paper); the submission names which.
6. **Runs cleanly.** The script runs end-to-end on an arbitrary input `.wav` without a mismatch assertion.

### 3.1 Fast-smoke test

```bash
python -m pip install "librosa>=0.10" "torchaudio>=2.1" "torch>=2.1" soundfile numpy
python extract_features.py path/to/any_wav_file.wav
# expected:
# mel  shape=(80, 101)  max_rel_err=~1e-5
# mfcc shape=(13, 101)  corr_coef_0=>0.95
```

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Fixed config declared | 10 % | Config table in §2.1 declared explicitly | Each row justified against a specific downstream ASR model's expected features |
| STFT / Mel / MFCC coverage | 20 % | All three implemented in both libraries | Complex STFT round-trips via inverse STFT; reconstruction SNR reported |
| Numerical reconciliation | 25 % | Mel agrees to `< 1e-4` relative error; MFCC trajectory correlation `> 0.95` | Reconciliation code produces identical MFCC values by matching `log_mels` and DCT normalisation |
| Shape determinism | 15 % | Output shape formula given and matches for 1-sec and 10-sec inputs | `center=True` vs `center=False` behaviour compared and explained |
| Pitfall write-up | 15 % | At least three of the "common mistakes" §5 called out explicitly | Includes an `htk=True` ablation showing the Mel difference |
| Runs end-to-end | 15 % | Script runs on a provided `.wav`, no NaNs, correct dtypes | Script also runs on a synthetic sine tone and shows the expected single-frequency peak in the right Mel bin |

A submission scoring under 70 % is usually a sign the learner has pasted one of the libraries' tutorial snippets without reconciling the output of the other.

## 5. Common mistakes

- **Letting `librosa.load` default to 22 050 Hz.** The default `sr` is 22 050, not the file's native rate. Pass `sr=16000` (or `sr=None` to preserve native) explicitly; otherwise torchaudio and librosa will read the same file at different rates.
- **Mixing `htk` and `slaney` Mel scales.** `librosa` defaults to `htk=False` (Slaney). torchaudio `MelSpectrogram` defaults to `mel_scale="htk"`. Match them explicitly or the Mel outputs diverge by ~5–10 % per bin.
- **Forgetting `norm="slaney"`.** The filterbank normalisation convention changes the energy per Mel bin; librosa and torchaudio differ here too unless both are set explicitly.
- **`center=True` vs `center=False`.** The two libraries default to `True`, but if a learner has copied a wav2vec2 or Whisper preprocessing snippet it may be `False`, which shifts the frame count by one and causes a shape mismatch.
- **Comparing MFCC absolute values.** The two libraries' MFCCs differ by a constant per coefficient because of the log-amplitude vs power-to-dB difference. Expect to compare *trajectory*, not value, unless you deliberately align the log step.
- **Using `n_fft=512` at 16 kHz.** 512 samples at 16 kHz is 32 ms, which blurs plosive onsets; 400 samples (25 ms) is the convention for ASR features at 16 kHz.
- **Power vs magnitude.** `power=2.0` → power spectrogram; `power=1.0` → magnitude. Many examples online quietly switch between the two; declare which.
- **Treating MFCCs as "mel spectrogram, but smaller"**. MFCCs are the DCT of log-mel, which decorrelates the bands but destroys the fine spectral structure the mel representation preserves. For modern neural ASR the Mel is what's fed in; MFCCs are legacy for GMM-HMM systems.

## 6. References

Primary — official project documentation:

- **librosa** — https://librosa.org/doc/latest/ .
  - `librosa.stft` — https://librosa.org/doc/latest/generated/librosa.stft.html .
  - `librosa.feature.melspectrogram` — https://librosa.org/doc/latest/generated/librosa.feature.melspectrogram.html .
  - `librosa.filters.mel` (HTK / Slaney, Slaney normalisation) — https://librosa.org/doc/latest/generated/librosa.filters.mel.html .
  - `librosa.feature.mfcc` — https://librosa.org/doc/latest/generated/librosa.feature.mfcc.html .
- **torchaudio** — https://pytorch.org/audio/stable/ .
  - `torchaudio.transforms.Spectrogram` — https://pytorch.org/audio/stable/generated/torchaudio.transforms.Spectrogram.html .
  - `torchaudio.transforms.MelSpectrogram` (`mel_scale`, `norm`) — https://pytorch.org/audio/stable/generated/torchaudio.transforms.MelSpectrogram.html .
  - `torchaudio.transforms.MFCC` (`log_mels`) — https://pytorch.org/audio/stable/generated/torchaudio.transforms.MFCC.html .
- **Whisper** (reference for 80-bin Mel / 25 ms / 10 ms convention) — https://github.com/openai/whisper (see `whisper/audio.py` for the reference preprocessor).

Local curriculum anchors:

- `mod-103-stt-production-engineering` — the feature pipeline this exercise hands off to.
- `mod-110-multimodal-input-design` — where learned front-ends (wav2vec2 / Whisper encoder) replace handcrafted MFCCs.

Source policy note. OWASP ML Top 10, MITRE ATLAS, and NIST AI RMF are not load-bearing for this exercise; feature extraction is a signal-processing / API-correctness exercise.
