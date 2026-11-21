# exercise-06 — Multilingual and Code-Switching Ladder — Reference Solution

## 1. Solution overview

This is a design + measurement exercise. A multilingual voice agent is not a single engineering decision — it is a *ladder* of choices, each adding complexity and each solving a specific user-visible problem. This solution:

- Enumerates the ladder from "English only" to "code-switch-aware".
- Names the user-visible failure mode that promotes you up each rung.
- Documents the per-provider capability map against the ladder.
- Supplies a measurement methodology for deciding where on the ladder your product actually sits.

Any vendor-specific capability claim is pinned to a doc URL and carries a `<!-- verify-at-runtime: ... -->` marker if it's the kind of claim that changes frequently (language list, Chirp vs non-Chirp, provider-side auto-detect vs user-picked).

## 2. Worked answer

### 2.1 The ladder (five rungs)

| Rung | Capability | User-visible failure mode that promotes you to this rung | Typical implementation |
|---|---|---|---|
| 0 | **Single-language, hard-coded** | None — appropriate when your market is single-language and you can commit to it | Set `language="en-US"` on every request. |
| 1 | **User-picked language per session** | "I speak Spanish, why is the bot responding in English?" | At session start, ask (or detect from caller ID / profile / IVR prompt) and pin the language for the session. |
| 2 | **Auto-detect at session start, then pin** | You cannot ask ("inbound call, language unknown; asking makes a bad first impression"); need to detect from the first utterance | Send the first 1–3 seconds to a language-identification (LID) step or use provider-side auto-detect; pin the session's language once resolved. |
| 3 | **Dynamic language switching mid-session** | Multilingual user switches languages between turns, mid-session | Re-detect per turn (or with hysteresis); update the ASR configuration between turns. |
| 4 | **Code-switching aware** (within-utterance) | User mixes two languages in a single utterance ("I need un presupuesto for the auto insurance") | Use a model that supports code-switch output natively, or stream parallel language pipelines and merge. |

The failure-mode column is load-bearing: a product that doesn't hit the failure mode doesn't need the rung. Climbing the ladder before you need to is pure cost burn.

### 2.2 Decision tree (which rung are you on?)

```
Is your market credibly monolingual?
    yes → Rung 0 — stop here.
    no  → continue

Can you ask the user to pick at session start (voice-menu "press 1 for English", or profile)?
    yes → Rung 1 — stop here unless you also hit the next test.
    no  → continue (inbound-cold-call, high-volume, multilingual market)

Does the mid-session failure case "user switches language turn-over-turn" happen > ~1 % of sessions?
    no  → Rung 2 (auto-detect then pin) is sufficient
    yes → continue

Does the within-utterance failure case "user mixes two languages in one utterance" happen regularly in real
data (not just hypothetically)?
    no  → Rung 3 (dynamic switch, utterance-grained)
    yes → Rung 4 (code-switching aware), with the cost warning in §2.5
```

The "> 1 %" and "regularly" thresholds are rough; tune them against your traffic. The decision tree still works if you move them.

### 2.3 Capability map — where providers sit

(Pin URLs; language lists change often; **re-verify against the current vendor docs at exercise-run time**.)

| Provider | R0 | R1 | R2 (auto-detect at start) | R3 (dynamic switch) | R4 (code-switching aware) |
|---|---|---|---|---|---|
| Deepgram | ✅ 30+ languages; `language="multi"` option gates multilingual behaviour on supported models. (`https://developers.deepgram.com/docs/models-languages-overview`) <!-- verify-at-runtime --> | ✅ | ✅ Auto-detect supported on some models. <!-- verify-at-runtime --> | ~ Requires client-side re-configuration between turns. | ~ `language="multi"` allows multi-language transcription on some model variants; range of language pairs supported varies. <!-- verify-at-runtime --> |
| AssemblyAI | ✅ (`https://www.assemblyai.com/docs/speech-to-text/pre-recorded-audio/supported-languages`) <!-- verify-at-runtime --> | ✅ | ✅ Language detection endpoint. (`https://www.assemblyai.com/docs/speech-to-text/pre-recorded-audio/automatic-language-detection`) | ~ Client-driven re-configuration between turns. | ~ Varies by model; verify. <!-- verify-at-runtime --> |
| Google Cloud Speech-to-Text | ✅ 125+ languages advertised with Chirp. (`https://cloud.google.com/speech-to-text/docs/languages`) <!-- verify-at-runtime --> | ✅ | ✅ `alternative_language_codes` + auto-detect. (`https://cloud.google.com/speech-to-text/docs/multiple-languages`) | ✅ v2 Chirp supports more flexible language handling. <!-- verify-at-runtime --> | ~ Some Chirp variants handle within-utterance mixing better than others. <!-- verify-at-runtime --> |
| Microsoft Azure Speech | ✅ 100+ languages. (`https://learn.microsoft.com/azure/ai-services/speech-service/language-support`) <!-- verify-at-runtime --> | ✅ | ✅ `AutoDetectSourceLanguageConfig`. (`https://learn.microsoft.com/azure/ai-services/speech-service/language-identification`) | ✅ Continuous language identification supported. (`https://learn.microsoft.com/azure/ai-services/speech-service/how-to-automatic-language-detection`) | ~ Mixed-language transcription available on some configurations. <!-- verify-at-runtime --> |
| OpenAI (Whisper lineage, `gpt-4o-transcribe`, Realtime) | ✅ Whisper-lineage models cover ~100 languages. (`https://github.com/openai/whisper#available-models-and-languages`) | ✅ | ✅ Auto-detect on by default when `language` is omitted. | ~ No mid-stream language switch API at the moment on the pure Whisper path; Realtime behaviour varies. <!-- verify-at-runtime --> | ~ Whisper-lineage models do some within-utterance mixing but outputs are often a single translated/transliterated form; verify. <!-- verify-at-runtime --> |
| Self-hosted Whisper | ✅ All above languages. | ✅ | ✅ Built-in language ID (first few seconds). | ~ Chunk the audio; LID per chunk; swap the decoder. | ~ Depends on chunking strategy and model. |
| NVIDIA Parakeet / Canary | Parakeet is English-centric; Canary is multilingual (EN/DE/FR/ES advertised). (`https://developer.nvidia.com/riva`) <!-- verify-at-runtime --> | ✅ | ~ Varies by model deployment. | ~ | ~ Canary is designed with multilingual translation / transcription in mind; verify. |

"✅" = documented support at the time this reference was written; "~" = partial / workaround-dependent. All cells `<!-- verify-at-runtime -->` must be re-pulled before quoting externally.

### 2.4 Per-rung implementation sketch

#### R1 — user-picked

Simplest. Pin the session.

```python
session.asr_config.language = user_selection  # e.g., "es-ES"
# For every stream the user opens this session, pass this value.
```

Guard: if the detector (R2) sees a mismatch between the user's selection and the audio's language with high confidence over N utterances, surface a "did you mean to switch to X?" dialog. Don't silently override — the user picked.

#### R2 — auto-detect at session start, then pin

A representative implementation runs LID on the first 1–3 seconds:

```python
# Illustrative — Azure Speech SDK auto-detect
auto_detect = speechsdk.AutoDetectSourceLanguageConfig(
    languages=["en-US", "es-ES", "pt-BR", "fr-FR"],  # closed set is more accurate than open set
)
recognizer = speechsdk.SpeechRecognizer(
    speech_config=speech_config,
    auto_detect_source_language_config=auto_detect,
    audio_config=audio_config,
)
# After first utterance, pin and don't auto-detect again this session.
```

Rules of thumb:

- Give LID a closed, small (≤ 6) candidate language set. LID accuracy drops sharply with open-set language lists.
- Require ≥ 1.5 seconds of non-silence. The failure mode is "the user said 'yes' and LID guessed random."
- Pin on the first high-confidence detection. Re-detecting every utterance is R3; don't accidentally promote to it.

#### R3 — dynamic switch (utterance-grained)

Re-detect per turn with hysteresis — once in a language, require two consecutive turns of a different language before switching. Hysteresis prevents LID flapping on short utterances.

```python
class LanguageState:
    def __init__(self, initial, hysteresis_turns=2):
        self.current = initial
        self.pending = None
        self.pending_count = 0
        self.hysteresis = hysteresis_turns

    def observe(self, detected):
        if detected == self.current:
            self.pending = None
            self.pending_count = 0
            return self.current
        if detected == self.pending:
            self.pending_count += 1
            if self.pending_count >= self.hysteresis:
                self.current = self.pending
                self.pending = None
                self.pending_count = 0
        else:
            self.pending = detected
            self.pending_count = 1
        return self.current
```

Open problem: the current turn's audio may need re-transcription under the new language if the switch is confirmed only *after* this turn is done. Choices: (a) accept a one-turn lag ("the first turn in the new language is transcribed under the old model"), (b) re-run the current turn's audio under the new model and merge, (c) use provider-side continuous-LID (Azure) and let the provider handle it.

#### R4 — code-switching aware

Within-utterance mixing — "I need un presupuesto for the auto insurance" — is the hardest rung. Three strategies:

1. **Provider-native code-switch.** Some providers advertise handling of within-utterance mixing (Azure continuous LID; Chirp; Deepgram `multi`). Verify empirically. Code-switch *output* varies: some providers return a single transcript with inline code-switching; some return per-language segments; some silently transliterate one language into the other.
2. **Parallel pipelines.** Stream the audio to two ASR pipelines (one per expected language); align word-level timings; take the higher-confidence word per time window. Works but doubles API cost and pipeline complexity.
3. **Fine-tuned code-switch model.** Open-weight models fine-tuned on code-switch corpora (Hindi-English, Spanish-English, Mandarin-English). Highest quality for the specific pair; requires training pipeline.

Pragmatic default when you need rung 4: start with provider-native, measure, and only fall back to strategies 2 or 3 if measurements require it.

### 2.5 Cost / complexity warning per rung

Each rung climb has a step up in cost and operational surface. Rough shape:

| Rung | API-call cost multiplier | Engineering surface | Debug complexity |
|---|---|---|---|
| R0 | 1× | tiny | trivial |
| R1 | 1× | small (session plumbing) | trivial |
| R2 | 1× (if provider-side LID is included in base price) to ~1.1× (LID as separate endpoint) | moderate (LID flow, pin logic) | moderate — "why did LID pick the wrong language?" is a real bug class |
| R3 | 1× if same pipeline + re-config; could be higher if parallel | real (hysteresis state machine, re-transcription edge cases) | high — flapping cases are hard to reproduce |
| R4 | 1× (native) to 2× (parallel pipelines) | large (merge logic, per-word alignment) | high — measurement is harder because reference transcripts for code-switch data are rare |

Climb only when the failure mode in §2.1 is actually observed in your data.

### 2.6 Measurement methodology

Extend the exercise-05 harness with multilingual data:

1. **FLEURS** (Google) — 2 009 hours of parallel read speech across 102 languages; standard multilingual benchmark. (`https://huggingface.co/datasets/google/fleurs`)
2. **CommonVoice** (Mozilla) — crowd-sourced; diverse accents per language. (`https://commonvoice.mozilla.org/en/datasets`)
3. **Your in-domain multilingual gold set** — same as exercise-05's gold set but stratified by language and (where possible) by code-switch pair.

Metrics:

- **Per-language WER** — per provider per language.
- **LID accuracy** — classification accuracy of the language-identification step on your closed language set. Report confusion matrix.
- **LID latency** — seconds of audio consumed before a stable LID verdict.
- **Switch-latency** (R3) — turns between the true language change and the pipeline's switch commit.
- **Code-switch quality** (R4) — custom metric: in a corpus of code-switched utterances with word-level language labels, what fraction of words were transcribed in the correct language?

Slice by accent / dialect where you can; the "English" rung of your product probably serves several distinguishable accents, each with its own WER.

## 3. Validation steps

A finished submission should pass each of the following checks.

1. **Ladder enumerated** with all five rungs and the failure-mode-that-promotes-you column filled.
2. **Decision tree applied** to the specific product — a one-line statement of which rung this product lives on and why.
3. **Capability map** for ≥ 4 providers against the five rungs. Cells cite doc URLs; moving claims carry `<!-- verify-at-runtime -->`.
4. **R2 implementation** present and functional. LID restricted to a closed candidate language set; pin logic documented.
5. **R3 implementation** present (if the product sits at or above R3). Hysteresis state machine documented; current-turn re-transcription edge case called out and resolved.
6. **R4 strategy** chosen (if the product sits at R4) with the measurement trigger that would flip from provider-native to parallel-pipeline to fine-tuned.
7. **Measurement extension** of the exercise-05 harness with per-language WER and LID metrics. At least two languages measured on an in-domain dataset.
8. **Cost / complexity warning** stated. Rung climbs are tied to observed failure modes, not aspiration.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Ladder | 10 % | Five rungs enumerated with failure-mode-per-rung | Decision tree with numeric thresholds backed to actual traffic observation |
| Capability map | 15 % | ≥ 4 providers mapped | All six providers mapped; cells carry pull dates and `<!-- verify-at-runtime -->` where appropriate |
| R2 implementation | 15 % | Working LID with closed candidate set and pin logic | Threshold tuning + "did you mean X?" surface for mismatches |
| R3 implementation (if applicable) | 15 % | Hysteresis state machine documented with the current-turn edge case resolved | Re-transcription merge logic shown; switch-latency measured |
| R4 strategy (if applicable) | 10 % | Strategy chosen with explicit trigger for escalation | Measurement loop that quantifies within-utterance code-switch quality |
| Measurement | 20 % | Per-language WER for ≥ 2 languages + LID accuracy / latency | Slices by accent / dialect; confusion matrix for LID; code-switch metric where applicable |
| Cost / complexity analysis | 10 % | Per-rung cost multiplier + engineering surface noted | Explicit decision log of where on the ladder the product sits and why it is not higher |
| Source policy compliance | 5 % | Vendor claims cited; `<!-- verify-at-runtime -->` present for moving claims | Dates pinned on every vendor cell |

Scoring < 70 % is a fail-and-redo. The most common sub-70 failure: a ladder jumped to R3 or R4 before the data showed the user-visible failure mode actually happened.

## 5. Common mistakes

- **Climbing before you need to.** Building R4 code-switch support for a product whose traffic is 99 % monolingual is cost burn with no user-visible benefit.
- **Open-set LID.** Letting the detector pick from "any language" instead of your product's closed set tanks accuracy and introduces weird-language misfires.
- **LID on 300 ms of audio.** Doesn't work. Give it 1.5 s of non-silence, minimum.
- **Flapping on short utterances.** A single "yes" can get LID-classified wrong. Hysteresis is not optional at R3.
- **Silently overriding the user.** User picked Spanish, LID says English, pipeline swaps to English: bad UX. Surface the mismatch; don't gaslight.
- **Benchmarking only on FLEURS.** FLEURS is read speech. Your product is conversational. The WER gap between the two can be 2–4× on the same language. Measure your in-domain audio.
- **Ignoring dialect.** "Spanish" is not one thing; "English" is not one thing. If your product serves both LATAM and Iberian Spanish, slice.
- **Code-switch handled by transliteration.** Some providers silently transliterate one language into the orthography of the other (e.g., output "yo quiero un presupuesto for the auto insurance" as "yo quiero un presupuesto fur de otto insurance"). If you don't measure code-switch quality explicitly, you won't catch this.
- **Not re-measuring on model-version changes.** Multilingual coverage is one of the things that changes most frequently vendor-side (new Chirp versions, new Whisper-lineage models). Re-run the harness.
- **Confusing "supported" with "production-quality".** A language being listed as supported does not mean it meets your product's quality bar. Measure, don't trust the list.
- **Parallel-pipeline merge that doesn't align word-time.** If two ASRs' outputs aren't aligned on time, the merge is random. Keep word-level timings and align.

## 6. References

Primary — provider multilingual docs:

- Deepgram models and languages overview — `https://developers.deepgram.com/docs/models-languages-overview`
- AssemblyAI supported languages — `https://www.assemblyai.com/docs/speech-to-text/pre-recorded-audio/supported-languages`
- AssemblyAI language detection — `https://www.assemblyai.com/docs/speech-to-text/pre-recorded-audio/automatic-language-detection`
- Google Cloud Speech-to-Text languages — `https://cloud.google.com/speech-to-text/docs/languages`
- Google Cloud Speech-to-Text multiple languages — `https://cloud.google.com/speech-to-text/docs/multiple-languages`
- Microsoft Azure Speech language support — `https://learn.microsoft.com/azure/ai-services/speech-service/language-support`
- Microsoft Azure language identification — `https://learn.microsoft.com/azure/ai-services/speech-service/language-identification`
- Microsoft Azure continuous LID — `https://learn.microsoft.com/azure/ai-services/speech-service/how-to-automatic-language-detection`
- OpenAI Whisper language list — `https://github.com/openai/whisper#available-models-and-languages`
- NVIDIA Riva / Parakeet / Canary — `https://developer.nvidia.com/riva`

Datasets:

- FLEURS — `https://huggingface.co/datasets/google/fleurs`
- CommonVoice — `https://commonvoice.mozilla.org/en/datasets`
- VoxPopuli — `https://github.com/facebookresearch/voxpopuli`
- SEAME (English-Mandarin code-switch) — `https://catalog.ldc.upenn.edu/LDC2015S04`
- Miami Bangor (English-Spanish code-switch) — `https://bangortalk.org.uk/speakers.php?c=miami`

Related exercises:

- exercise-01 (provider decision matrix) — the per-language capability check is one input to the matrix.
- exercise-04 (keyterm boosting) — in multilingual products, boost lists should be per-language (a term mapping may differ).
- exercise-05 (WER benchmark) — extended here to multilingual datasets; the harness is the same.

Standards referenced indirectly:

- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — the "measure, govern, document" posture that gates ladder climbs.
- OWASP ML Top 10 — `https://owasp.org/www-project-machine-learning-security-top-10/` — relevant when adversarial-language-mixing is a threat vector (feeding crafted code-switch audio to probe the system).
- MITRE ATLAS — `https://atlas.mitre.org/` — threat-model reference, including audio-channel attacks that exploit language-switch state transitions.

Source policy note. Language lists, auto-detect / multilingual features, and model-variant support change frequently vendor-side; every `<!-- verify-at-runtime -->` marker in §2.3 should be re-verified against the vendor's own current documentation before quoting externally. The ladder itself is a local-to-this-curriculum operational framing; no vendor or standards body endorses the five-rung split as such.
