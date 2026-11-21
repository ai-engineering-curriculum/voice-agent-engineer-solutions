# exercise-05 — WER Benchmark — LibriSpeech + In-Domain Gold Set — Reference Solution

## 1. Solution overview

This is an implementation exercise. The deliverable is a reproducible WER-benchmark harness that:

1. Runs each shortlisted ASR provider (and self-hosted Whisper / Parakeet baselines) on **LibriSpeech test-clean and test-other** — the public, well-understood baseline most speech papers cite.
2. Runs the same providers on **your own in-domain gold set** — the dataset that matches the audio your product actually serves.
3. Reports WER (and companion metrics — CER, substitution/insertion/deletion rates) with sample sizes and confidence intervals.
4. Produces the per-provider + per-dataset decision input feeding exercises 01 and 04.

WER on LibriSpeech alone is the number vendor marketing pages lead with; WER on your in-domain gold set is the number that predicts your product. A mature submission runs both and reports them side by side.

Numbers reported below under "Illustrative results" are explicitly labelled **illustrative composites**. They are *the shape* of a credible output, not published benchmarks. Real provider leaderboards on LibriSpeech should be pulled from the vendor's own blog posts / papers at exercise-run time; the authoritative LibriSpeech leaderboard across models is the Papers-with-Code board (`https://paperswithcode.com/sota/speech-recognition-on-librispeech-test-clean`).

## 2. Implementation

### 2.1 Dataset setup

#### LibriSpeech

LibriSpeech is a ~1000-hour English read-speech corpus released by Panayotov et al. (`https://www.openslr.org/12/`, licence CC-BY 4.0). The two test splits commonly reported:

- `test-clean` — 5.4 hours, clean broadband read speech. Typical "ceiling" ASR comparison. Historically, modern models cluster in the 1.5–3 % WER range on this split. <!-- verify-at-runtime: confirm current SOTA at exercise-run time -->
- `test-other` — 5.1 hours, harder (noisier, more accented). Modern models cluster in the 3–7 % WER range. <!-- verify-at-runtime -->

Download:

```bash
# Illustrative — accept licence terms, verify integrity
mkdir -p data/librispeech && cd data/librispeech
curl -O https://www.openslr.org/resources/12/test-clean.tar.gz
curl -O https://www.openslr.org/resources/12/test-other.tar.gz
echo "<expected sha256>  test-clean.tar.gz" | sha256sum -c -
tar xzf test-clean.tar.gz
tar xzf test-other.tar.gz
```

Each `*.flac` file has a matching reference transcript in a per-chapter `*.trans.txt` file. The transcripts are all uppercase, no punctuation, apostrophes retained — normalise your hypotheses to match before measuring WER (see §2.3).

#### In-domain gold set

The in-domain gold set is what distinguishes a serious benchmark from a demo. Minimum viable gold set:

- **Audio.** 50–200 utterances matching your product's audio profile exactly: same codec (e.g., Opus at 24 kHz for WebRTC, G.711 µ-law at 8 kHz for telephony), same channel count, same microphone class, same environment noise profile (quiet office, car hands-free, call-centre background).
- **Reference transcripts.** Two independent humans transcribe each utterance; a third reconciles disagreements. Record each pass and the disagreement rate; your WER measurement is only meaningful up to the human inter-annotator disagreement.
- **Metadata.** Per-utterance: speaker-id, language/dialect, channel/codec, duration, accent (self-declared or labelled), any domain-term density flag. Metadata lets you slice WER (§2.4).
- **Splits.** `dev` and `test` splits with no overlap of speakers — do NOT tune thresholds against the test split.
- **Consent / provenance.** Every utterance has a recorded consent trail. No audio sourced from users who did not consent to its use for model evaluation.

Target 100 utterances as the "we can make decisions" bar; 200+ for publishing externally. Smaller gold sets don't move WER differences out of the noise floor.

A reasonable gold-set directory layout:

```
data/gold-set/v1/
├── manifest.jsonl          # one line per utterance: {"id","audio_path","reference","meta":{...}}
├── audio/
│   ├── 0001.wav
│   ├── 0002.wav
│   └── ...
└── CONSENT_TRAIL.md        # per-utterance provenance + consent evidence
```

### 2.2 Harness architecture

```python
# Illustrative harness structure. Real code: see your repo.
#
# One Provider class per ASR; same signature. Each provider is responsible for
# authentication, request/response, and normalising its output to {transcript,
# word_confidences, duration_s}.

from dataclasses import dataclass
from typing import Protocol, Iterable
import jiwer

@dataclass
class Hypothesis:
    transcript: str
    audio_duration_s: float
    # Confidence fields optional — used by exercise-03, not by WER measurement.

class Provider(Protocol):
    name: str
    def transcribe(self, audio_path: str, *, language: str = "en-US") -> Hypothesis: ...

def run_benchmark(providers: list[Provider], manifest: Iterable[dict], normalise, out_path: str):
    rows = []
    for row in manifest:
        ref = row["reference"]
        for p in providers:
            hyp = p.transcribe(row["audio_path"], language=row["meta"]["language"])
            n_ref = normalise(ref)
            n_hyp = normalise(hyp.transcript)
            measures = jiwer.compute_measures(n_ref, n_hyp)
            rows.append({
                "dataset": row["meta"]["dataset"],
                "utt_id":  row["id"],
                "provider": p.name,
                "wer":   measures["wer"],
                "cer":   jiwer.cer(n_ref, n_hyp),
                "subs":  measures["substitutions"],
                "ins":   measures["insertions"],
                "dels":  measures["deletions"],
                "n_ref_words": len(n_ref.split()),
                "duration_s":  hyp.audio_duration_s,
            })
    write_jsonl(rows, out_path)
```

Aggregate per `(dataset, provider)` at report time, weighting by reference-word count (so a 50-word utterance contributes 10× a 5-word one).

### 2.3 Text normalisation

Normalising both reference and hypothesis to the same form is the single most error-prone step in the harness. Unnormalised text produces artificially high WER (e.g., counting "9" vs "nine" as a substitution).

Minimum normalisation steps:

1. Case-fold (lower).
2. Strip punctuation except apostrophes in contractions (`don't`, `it's`).
3. Collapse whitespace; strip leading/trailing.
4. Normalise disfluencies (`uh`, `um`, `hmm`) consistently — either keep in both or strip from both.
5. Number normalisation — spell out digits ("9" → "nine") OR digitise words ("nine" → "9") — pick one direction and apply to both. The Whisper normaliser in `openai-whisper` is the de-facto reference for English.
6. Keep contractions atomic (`don't` is one word, not `do + nt`).

Normalisation is one of the directions people silently game WER. If the normaliser strips numbers or hyphens entirely, you can get a "lower" WER that doesn't reflect user experience. Audit the normaliser.

Suggested starting point: use the `jiwer` default transforms plus the Whisper English normaliser as a consistency check. Document the normaliser used in every report.

### 2.4 Slicing

Overall WER hides the signal. Slice:

- By **dataset** (LibriSpeech test-clean, test-other, in-domain dev, in-domain test).
- By **duration bucket** (<2 s, 2–5 s, 5–10 s, 10–30 s, >30 s) — very short utterances dominate error rate by fraction; providers differ on whether they do better on short or long audio.
- By **speaker** / **accent** (if labelled).
- By **codec** (broadband vs 8 kHz telephony) — this is often the biggest drop on in-domain data.
- By **noise condition** (quiet / medium / noisy, if labelled).
- By **keyterm density** (feeds back into exercise-04).

Report the slice table, not just an overall number.

### 2.5 Illustrative results — what a reviewer should expect

Numbers below are **illustrative composite** values intended to show the shape of a credible output. They are not drawn from any specific vendor's published measurements, and should not be cited anywhere else.

| Provider | LibriSpeech test-clean (WER) | LibriSpeech test-other (WER) | In-domain gold-set test (WER) |
|---|---|---|---|
| Deepgram Nova-2 | ~2–3 % | ~5–7 % | ~9–12 % |
| AssemblyAI Universal | ~2–3 % | ~5–7 % | ~8–11 % |
| Google Chirp | ~2–4 % | ~5–8 % | ~10–14 % |
| Azure Speech | ~3–5 % | ~7–9 % | ~11–14 % |
| OpenAI gpt-4o-transcribe | ~2–3 % | ~5–7 % | ~9–12 % |
| Whisper large-v3 (self-hosted) | ~2–3 % | ~5–7 % | ~11–15 % |
| Parakeet (NVIDIA, self-hosted) | ~2–3 % (English) | ~5–7 % (English) | ~10–14 % |

Readings a reviewer should draw from a table *of this shape*:

- On LibriSpeech test-clean the modern top models are close. The product-relevant differentiation comes from test-other and the in-domain gold set.
- The in-domain gap (gold set WER minus LibriSpeech-clean WER) is the *domain gap* — it predicts your product's user experience far better than the LibriSpeech leaderboard.
- One provider's advantage on the leaderboard does not always port to your gold set. Measure.
- Self-hosted Whisper is often behind on in-domain 8 kHz telephony audio because its training is broadband-weighted. Parakeet's English-only focus sometimes closes the gap on telephony.

### 2.6 Statistical rigour

Reporting a single WER number from 50 utterances is close to noise. Minimum rigour:

- **Sample size.** State it explicitly. Prefer ≥ 100 utterances / ~1000 reference words per slice before quoting a point value.
- **Bootstrap confidence intervals.** Resample utterances with replacement B=1000 times; report the 95 % percentile interval.
- **Paired comparisons.** When comparing providers, compare on the *same* utterances. Report the *paired* difference distribution, not two independent means.
- **Multiple comparisons.** If you sweep across several providers and datasets, apply a correction (Bonferroni or Benjamini-Hochberg) before declaring statistical significance.
- **Pre-register the gold set.** Freeze it before running the comparison. Rotating the gold set to make a provider "win" is a well-trodden path to a shipping decision you later regret.

### 2.7 Running cost

Benchmarking 200 utterances × 6 providers = 1200 API calls per run. For most commercial ASRs, each ~5-second utterance is a fraction of a cent; a full run costs < US $5. Self-hosted runs are GPU-bound (a few minutes on a modern GPU). Document the per-run cost in the README so the reviewer knows what reruns cost.

## 3. Validation steps

A finished submission should pass each of the following checks.

1. **LibriSpeech test-clean and test-other both run.** For every provider, both splits are included; results are reproducible by running the harness on another machine.
2. **In-domain gold set exists.** Manifest + consent trail + metadata. Minimum 100 utterances; dev / test split; speaker-disjoint.
3. **Normalisation documented.** The normaliser used is named and version-pinned (e.g., `jiwer ==X.Y.Z` + Whisper English normaliser revision).
4. **Metrics beyond WER.** CER, substitution / insertion / deletion rates are reported alongside WER.
5. **Slicing reported.** At least two slices (duration and codec, or duration and noise) beyond the overall number.
6. **Statistical rigour.** Sample sizes stated; paired comparisons when comparing providers; 95 % confidence intervals via bootstrap.
7. **Cost and reproducibility.** The per-run cost (API dollars + GPU minutes) is documented; a reviewer could rerun and get the same number up to API-side non-determinism.
8. **Illustrative numbers labelled.** Any numeric quoted in the write-up is either a measurement from this harness or clearly labelled as illustrative.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Dataset setup | 15 % | LibriSpeech test-clean + test-other + ≥ 100-utterance in-domain gold set | Multi-codec in-domain gold set (broadband + 8 kHz µ-law); consent trail published |
| Harness code | 15 % | Runnable; one Provider class per ASR; same signature | CI-compatible; providers mockable for test; paralleliseable |
| Normalisation | 10 % | Normaliser named and version-pinned | Normaliser-audit page showing before/after on a sample of hypotheses |
| Metrics | 10 % | WER + CER + sub/ins/del per provider per dataset | Keyterm-error-rate alongside (feeds exercise-04) |
| Slicing | 15 % | ≥ 2 slices beyond overall | Slices by accent, speaker, noise, and codec, with per-slice confidence intervals |
| Statistical rigour | 15 % | Sample sizes stated + bootstrap CIs + paired comparisons when comparing providers | Pre-registered gold set; multiple-comparison correction applied |
| Reproducibility + cost | 10 % | README with run command + per-run cost estimate | Deterministic re-run on same inputs (where provider-side allows) + model-version pinning |
| Source policy compliance | 10 % | Illustrative numbers labelled; vendor claims pinned to URLs + dates | `<!-- verify-at-runtime -->` markers present on claims that move (SOTA numbers, pricing); human-annotator disagreement reported |

Scoring < 70 % is a fail-and-redo. The most common sub-70 failure: a benchmark that reports one point-WER-per-provider from a 30-utterance sample, with no normaliser audit and no paired comparison.

## 5. Common mistakes

- **Benchmarking on LibriSpeech only.** The vendor marketing already did this for you, in their favour. Your in-domain gold set is where the decision is made.
- **Reference transcripts from a single annotator.** Any WER below the inter-annotator disagreement rate is noise. You need the disagreement number before you can talk about measurement precision.
- **Gold set from ad-hoc staff recordings.** A gold set that doesn't match the production audio profile (codec, noise, device) will over-predict production accuracy. Match the audio, not just the content.
- **Normaliser does the work.** Aggressive normalisation (strip all numbers, strip all hyphens, map all synonyms) flatters every provider equally but hides the differences that matter for your product. Audit the normaliser.
- **Comparing WERs from different normalisers.** Not comparable. Everyone must use the same normaliser.
- **Reporting a mean without a CI.** 11.4 % and 11.9 % mean nothing without a sample size and a bootstrap interval.
- **Non-paired comparisons.** Comparing provider A's WER to provider B's WER, each computed on a *different* random subset, hides half the signal. Always pair.
- **Overlap between dev and test.** Threshold-tuning on the test split is leakage. Keep dev strictly separate.
- **Not pinning model version.** Deepgram Nova-2.1 is not Deepgram Nova-2.0; Whisper large-v3 is not Whisper large-v3.1. Pin and record.
- **Not re-running on model-version updates.** Vendors change models silently. A WER measured six months ago is a measurement about a previous product. Re-run on every announced model version change and on your own monthly cadence.
- **Benchmarking only the "happy" slice.** 8 kHz telephony, strong accents, background noise are where the providers diverge. A benchmark that excludes them will mis-rank providers for a telephony product.
- **Using the gold set for everything.** If you also use it as a keyterm-boost tuning set (exercise-04) and as a threshold-sweep set (exercise-03) and as a final benchmark, you've overfit to it. Keep a held-out test slice that is only ever used for final comparisons.

## 6. References

Primary — dataset and tooling:

- LibriSpeech (OpenSLR 12) — `https://www.openslr.org/12/`
- LibriSpeech paper (Panayotov et al., ICASSP 2015) — `https://www.danielpovey.com/files/2015_icassp_librispeech.pdf`
- Papers-with-Code LibriSpeech test-clean leaderboard — `https://paperswithcode.com/sota/speech-recognition-on-librispeech-test-clean`
- `jiwer` — `https://github.com/jitsi/jiwer`
- OpenAI Whisper English normaliser — `https://github.com/openai/whisper/blob/main/whisper/normalizers/english.py`
- `kaldialign` — `https://github.com/pzelasko/kaldialign`

Related datasets (for stretch):

- FLEURS (Google, multilingual) — `https://huggingface.co/datasets/google/fleurs`
- CommonVoice (Mozilla, multilingual + accents) — `https://commonvoice.mozilla.org/en/datasets`
- VoxPopuli (multilingual) — `https://github.com/facebookresearch/voxpopuli`

Provider documentation (for the per-provider SDK code you'll ship in the harness):

- Deepgram — `https://developers.deepgram.com/docs`
- AssemblyAI — `https://www.assemblyai.com/docs`
- Google Cloud Speech-to-Text — `https://cloud.google.com/speech-to-text/docs`
- Microsoft Azure Speech — `https://learn.microsoft.com/azure/ai-services/speech-service/`
- OpenAI speech-to-text — `https://platform.openai.com/docs/guides/speech-to-text`
- Whisper — `https://github.com/openai/whisper`
- NVIDIA Riva / Parakeet — `https://developer.nvidia.com/riva`

Related exercises:

- exercise-01 (STT provider decision matrix) — consumes the measurement outputs produced here.
- exercise-04 (keyterm boosting) — reuses this harness with term lists; introduces the keyterm-error-rate metric.
- exercise-06 (multilingual ladder) — extends the harness to multilingual datasets.

Standards referenced indirectly:

- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — "measure, govern, document" discipline behind §2.6 / §3.
- OWASP ML Top 10 — `https://owasp.org/www-project-machine-learning-security-top-10/` — relevant when the gold set extends to adversarial / poisoned audio.
- MITRE ATLAS — `https://atlas.mitre.org/` — audio-adversarial threat model reference; useful when stress-testing providers with adversarial inputs.

Source policy note. All WER numbers in §2.5 are illustrative composite values intended to show the shape of a credible output. SOTA LibriSpeech numbers move with every ICASSP / Interspeech cycle and should be re-pulled from the Papers-with-Code board or the vendor's own publications at exercise-run time — the ranges marked `<!-- verify-at-runtime -->` in §2.1 should be verified before quoting any specific number downstream.
