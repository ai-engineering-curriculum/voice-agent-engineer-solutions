# exercise-04 — Ship Custom Vocab and Keyterm Prompting — Reference Solution

## 1. Solution overview

This is an implementation exercise. The deliverable is a reproducible pipeline that:

1. Collects a list of in-domain terms the generic ASR mis-transcribes (brand names, product SKUs, medical terms, person names, local-geography names).
2. Configures each candidate provider's customisation surface (keyterms, word-boost, custom vocab, speech-adaptation PhraseSets, Custom Speech models).
3. Measures WER on the same gold set before and after, per provider.
4. Decides which terms to ship, which to drop, and when to escalate from term-boosting to a custom-trained model.

The worked example uses an illustrative *insurance claims* voice agent with a small jargon list and a fictional company name ("Boreal Mutual"). Numbers reported in §2.4 are labelled as **illustrative composites** and are the shape of what a reasonable measurement produces — not a published benchmark.

Any vendor-specific detail (parameter names, boost-value ranges, maximum list sizes) is pinned to a doc URL. These move; re-verify at exercise-run time. Cells that cannot be verified from vendor docs carry `<!-- verify-at-runtime: ... -->`.

## 2. Implementation

### 2.1 Build the term list

A good keyterm list is *small* and *targeted*. The failure mode is a 10 000-word list that over-boosts everything; the general-case accuracy drops and the vendor warns you about this in the docs.

Build the list from measurement, not from a brainstorm:

1. Run baseline transcription over 100–300 in-domain audio clips (see exercise-05 for the gold set).
2. Diff the ASR transcript against the human-labelled truth per word. The error-word list, grouped by frequency, is the candidate set.
3. Keep only words that (a) appear ≥ 3 times across clips, (b) are proper nouns, domain terms, or product names (the kinds of terms a general LM wouldn't learn), and (c) are not already handled correctly by the baseline on ≥ 80 % of occurrences.

A worked candidate set for the illustrative insurance-claims route:

| Term | Baseline error example | Frequency in gold set (illustrative) | Confidence (illustrative) |
|---|---|---|---|
| Boreal Mutual | "borel mutual" / "boreal natural" | 48 | 0.71 |
| HO-3 (homeowners policy form) | "H O three" / "H oh three" | 22 | 0.69 |
| adjuster | "a juster" / correct | 31 | 0.84 |
| deductible | mostly correct | 36 | 0.91 |
| actuarial | "actuary all" | 7 | 0.70 |
| ACORD (industry standard form family) | "accord" | 14 | 0.72 |
| claim number 2024-XXXXXX | mostly correct on digits | 29 | 0.86 |

Decision: ship "Boreal Mutual", "HO-3", "ACORD", "actuarial" as custom terms. Don't ship "deductible" (already fine) or "adjuster" (fine most of the time; boost risks over-inserting). Don't ship claim-number formats as terms — those need number-formatting / regex handling, not keyterm boosting.

Keyterm boosting fixes *vocabulary*, not *grammar*. If your ASR mis-handles claim-number formatting, that's an output-formatting or regex-post-processing problem, not a term-list problem.

### 2.2 Provider-specific configuration

Below are the current customisation surfaces per provider, with doc URLs. The shape is similar; the parameter names differ.

#### 2.2.1 Deepgram — `keyterm` / `keywords`

Deepgram's streaming and batch APIs take term-boost parameters on the request:

```python
# Deepgram SDK (Python) — illustrative
from deepgram import DeepgramClient, PrerecordedOptions

client = DeepgramClient(api_key="...")
options = PrerecordedOptions(
    model="nova-2",
    language="en-US",
    smart_format=True,
    keyterm=["Boreal Mutual", "HO-3", "ACORD", "actuarial"],  # Nova-3 / keyterm feature
)
# Older models / streaming: use `keywords=["Boreal Mutual:5", "HO-3:5", ...]`
# where the integer is a per-term intensity.
response = client.listen.prerecorded.v("1").transcribe_file(..., options)
```

See `https://developers.deepgram.com/docs/keyterm` for the current keyterm feature (model-version-gated) and `https://developers.deepgram.com/docs/keywords` for the legacy keywords parameter. The keyterm feature is the preferred surface when supported by the model; `keywords` remains for older models. <!-- verify-at-runtime: confirm which models currently support keyterm vs keywords at exercise-run time -->

Operational notes:

- Keep the boost intensity (where supported) low. Over-boosting inserts the term even when it wasn't said.
- Boost single terms or very short phrases. Long phrases are better handled by the model's language context, not by boosting.
- Measure with and without term-boost on the *same* audio; boosting can reduce WER on target terms at the cost of raising it on neighbouring words.

#### 2.2.2 AssemblyAI — `word_boost`

```python
# AssemblyAI SDK — illustrative
import assemblyai as aai

aai.settings.api_key = "..."
transcriber = aai.Transcriber(
    config=aai.TranscriptionConfig(
        word_boost=["Boreal Mutual", "HO-3", "ACORD", "actuarial"],
        boost_param="high",   # one of low | default | high
    )
)
transcript = transcriber.transcribe(audio_url)
```

See `https://www.assemblyai.com/docs/speech-to-text/pre-recorded-audio/customize-vocabulary` for the canonical page. Documented limits (size of list, allowed characters) are on that page and change; re-pull at exercise-run time. <!-- verify-at-runtime -->

#### 2.2.3 Google Cloud Speech-to-Text — Speech Adaptation (PhraseSets + CustomClasses)

Google's customisation is deeper than a boost list: PhraseSets hold phrases, CustomClasses hold token-level vocab (e.g., a list of product SKUs that fit into a `$PRODUCT` slot in a phrase). A phrase set is attached to a `recognizer` or passed per-request.

```python
# Google Cloud Speech-to-Text v2 — illustrative (python-speech)
from google.cloud.speech_v2 import SpeechClient, types

client = SpeechClient()
adaptation = types.SpeechAdaptation(
    phrase_sets=[
        types.SpeechAdaptation.AdaptationPhraseSet(
            inline_phrase_set=types.PhraseSet(
                phrases=[
                    types.PhraseSet.Phrase(value="Boreal Mutual", boost=15),
                    types.PhraseSet.Phrase(value="HO-3", boost=15),
                    types.PhraseSet.Phrase(value="ACORD", boost=15),
                    types.PhraseSet.Phrase(value="actuarial", boost=10),
                ]
            )
        )
    ],
)
config = types.RecognitionConfig(adaptation=adaptation, ...)
```

See `https://cloud.google.com/speech-to-text/docs/adaptation-model` and `https://cloud.google.com/speech-to-text/v2/docs/phrase-sets`. Boost magnitude ranges are documented on those pages and are different for Chirp and non-Chirp models. <!-- verify-at-runtime -->

For routes with large term sets (thousands of SKUs) prefer CustomClasses with a referencing phrase (e.g., `"my product is $product"` with `$product` resolved against a CustomClass).

#### 2.2.4 Microsoft Azure — Custom Speech

Azure's customisation is split by what you're customising:

- *Phrase lists* — in-request boost of specific words / phrases (for endpointing-friendly tuning). Available via the Speech SDK's `PhraseListGrammar.FromRecognizer(...).AddPhrase(...)`.
- *Custom Speech* — a trained custom model built from pronunciation dictionaries, custom-language-model text corpora, and acoustic adaptation data. Used when you have hundreds of terms, bespoke pronunciations, or a dialect the base model under-serves.

See `https://learn.microsoft.com/azure/ai-services/speech-service/how-to-custom-speech-overview` for the Custom Speech story and `https://learn.microsoft.com/azure/ai-services/speech-service/improve-accuracy-phrase-list` for phrase lists.

```python
# Azure Speech SDK (Python) — illustrative
import azure.cognitiveservices.speech as speechsdk

recognizer = speechsdk.SpeechRecognizer(...)
grammar = speechsdk.PhraseListGrammar.from_recognizer(recognizer)
for term in ["Boreal Mutual", "HO-3", "ACORD", "actuarial"]:
    grammar.add_phrase(term)
```

#### 2.2.5 OpenAI — prompt-string biasing

OpenAI's transcription endpoints (`whisper-1`, `gpt-4o-transcribe`, `gpt-4o-mini-transcribe`) accept a `prompt` string that biases decoding toward terms in the prompt. It is a shallower surface than the dedicated-provider boost lists:

```python
# OpenAI Python SDK — illustrative
from openai import OpenAI
client = OpenAI()
transcript = client.audio.transcriptions.create(
    model="gpt-4o-transcribe",
    file=open("call.wav", "rb"),
    prompt="This call involves Boreal Mutual, an HO-3 policy, ACORD forms, and actuarial review.",
)
```

See `https://platform.openai.com/docs/guides/speech-to-text`. Caveats: the prompt is treated as a decoding hint with a token budget; a prompt that is too long or too contradictory degrades accuracy on unrelated speech. Keep it short.

#### 2.2.6 Self-hosted Whisper / Parakeet

Whisper's `initial_prompt` parameter is the open-weight equivalent of the OpenAI prompt (same mechanism; Whisper is the lineage). For deeper customisation on open-weight models, options include:

- Custom-trained CTC decoders with an external language model (KenLM).
- Fine-tuning the acoustic or language head on in-domain data.
- Hot-word / biasing extensions (model-specific).

Depth increases with ambition: prompt → biased decoding → fine-tuned head → fully fine-tuned model. Each step has a 10× cost and a 2–5× accuracy jump on the target terms (illustrative order-of-magnitude; measure for your case).

See `https://github.com/openai/whisper` and `https://github.com/SYSTRAN/faster-whisper` for the initial-prompt API; `https://developer.nvidia.com/riva` for Riva / Parakeet customisation paths.

### 2.3 Measurement harness (what you actually run)

Reuse the exercise-05 WER harness here. The measurement loop:

```python
# Illustrative harness pseudocode
providers = [DeepgramProvider(...), AssemblyAIProvider(...), GoogleProvider(...),
             AzureProvider(...), OpenAIProvider(...), WhisperLocalProvider(...)]
keyterms = ["Boreal Mutual", "HO-3", "ACORD", "actuarial"]
gold_set = load_gold_set("insurance-claims-v1")  # (audio, reference_transcript) pairs

results = []
for provider in providers:
    for use_boost in (False, True):
        hypotheses = [provider.transcribe(a, keyterms=(keyterms if use_boost else None))
                      for a, _ in gold_set]
        wer_overall = jiwer.wer([r for _, r in gold_set], hypotheses)
        wer_on_keyterms = keyterm_error_rate(gold_set, hypotheses, keyterms)
        results.append((provider.name, use_boost, wer_overall, wer_on_keyterms))

print_table(results)
```

`keyterm_error_rate` is a custom metric: for every occurrence of a target term in the reference, was the hypothesis word-aligned to the same term? This number is often the thing you actually care about, separate from the global WER.

### 2.4 Illustrative results — what a reviewer should expect to see

Numbers below are **illustrative composite** values to illustrate the *shape* of what the measurement produces. They are not published benchmarks and should not be cited anywhere else.

| Provider | WER overall (baseline) | WER overall (with boost) | Keyterm-error rate (baseline) | Keyterm-error rate (with boost) |
|---|---|---|---|---|
| Deepgram Nova-2 | 12.1 % | 11.9 % | 38 % | 9 % |
| AssemblyAI Universal | 11.4 % | 11.3 % | 32 % | 12 % |
| Google Chirp | 13.0 % | 12.7 % | 41 % | 15 % |
| Azure Speech (base) | 12.7 % | 12.6 % | 39 % | 18 % |
| OpenAI gpt-4o-transcribe (prompt) | 10.9 % | 10.8 % | 29 % | 19 % |
| Whisper large-v3 self-hosted | 12.9 % | 12.5 % (initial_prompt) | 36 % | 24 % |

Readings a reviewer should draw from a table *of this shape*:

- Term-boost typically produces a **small** overall-WER improvement and a **large** keyterm-error-rate improvement. That is the win case: the general-case accuracy is unchanged, and the target terms are now reliable.
- If overall WER goes *up* with boost, you have over-boosted. Lower the boost intensity (where tuneable), shorten the list, or move to a trained Custom Speech / fine-tuned model.
- The OpenAI prompt and Whisper `initial_prompt` are consistently shallower than the dedicated-provider boost surfaces on keyterm error. For a term-dense route, dedicate-provider customisation is usually worth it.
- One provider leading overall WER does not mean it leads keyterm error. Decide per metric, per route.

### 2.5 When to escalate from boost to a trained model

Rule of thumb: boost first, train only when boost has stopped moving the keyterm-error-rate.

Escalation triggers:

- Keyterm-error-rate after boost > 10 % on a term you promised the customer would "just work."
- The term is a *pronunciation* problem (e.g., brand names pronounced in a non-English-native way) — boost the written form all you want; the acoustic model still mis-hears. Custom pronunciations (Azure Custom Speech) or fine-tuning is the fix.
- The term list is too large to hold in a boost list (vendor-side limits) and the natural shape is actually hundreds of SKUs.

Training is the expensive option (data collection + per-vendor training pipeline + model-release gating). Don't jump to it before measuring the boost ceiling.

## 3. Validation steps

A finished submission should pass each of the following checks.

1. **Term list is measurement-driven.** The list was produced from a baseline error analysis, not a brainstorm. The per-term frequency and baseline-error-rate are included.
2. **Per-provider configuration shown.** For at least three providers, the actual API/SDK call that applies the term list is included and runs.
3. **Boost-vs-baseline measurement per provider.** For at least three providers, WER overall and keyterm-error-rate are reported *both* with and without boost, on the same gold set.
4. **Keyterm-error-rate is defined.** The per-term alignment metric is defined explicitly — not just "we checked by eye."
5. **Over-boost check.** The analysis explicitly checks whether overall WER rose after boost; if it did, the submission names the mitigation.
6. **Escalation criteria stated.** The "when to train" criteria are listed, not left implicit.
7. **Numbers labelled.** Any WER value presented is labelled with the sample size and whether it is a measurement or an illustrative composite. No numbers are presented as real measurements unless they are.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Term list method | 15 % | Measurement-driven list, with frequency + baseline-error per term | Terms stratified by category (proper noun / product / jargon / entity); over-selection called out |
| Per-provider configuration | 20 % | ≥ 3 providers with runnable SDK/CLI snippet | All six providers, each with a note on the surface's limits (list size, boost range) |
| Measurement rigour | 20 % | WER overall + keyterm-error-rate, with and without boost, same gold set | Confidence intervals / sample-size reported; per-term break-down included |
| Over-boost / regression check | 10 % | Explicit check that overall WER didn't degrade | Regression on non-target words quantified, not just asserted |
| Escalation criteria | 10 % | Criteria for moving boost → Custom Speech / fine-tune listed | Criteria tied to the measurement gates, not hand-waved |
| Reproducibility | 15 % | Gold set + harness code reachable from the submission | Harness reruns cleanly on a different provider without edits |
| Source policy compliance | 10 % | Vendor docs cited; illustrative numbers labelled as such | `<!-- verify-at-runtime -->` markers present where vendor claims are pulled and date-pinned |

Scoring < 70 % is a fail-and-redo. The most common sub-70 failure: boost applied without a measurement showing it helped on the target terms, or (worse) with a measurement that quietly includes a global-WER regression.

## 5. Common mistakes

- **Term list as brainstorm.** "Add every word the PM thinks is important." No. Use the baseline error list.
- **Term list too long.** A 10 000-word list over-boosts everything. Under 100 is a good starting ceiling; some providers quote lower caps.
- **No before / after.** Submitting "we added keyterms" without measurement is not shipping — it's wishing.
- **Only measuring overall WER.** The keyterm-error-rate is the metric that moves on this exercise. Overall WER can be flat while keyterm errors drop 4×.
- **Over-boosting.** Insertion errors when the term wasn't said. Lower intensity, shorter list.
- **Boosting words the LM already handles.** No value, slight risk of regression on neighbours. Only boost terms whose baseline error rate is meaningfully bad.
- **Confusing formatting problems with vocabulary problems.** "Claim number 2024-012345" being transcribed as "2024, 012 345" is a formatting problem — a boost won't fix it. Regex post-processing or `smart_format=True` / equivalent will.
- **Boosting phrases longer than ~5 words.** Boosting a phrase is a token-budget concept; short phrases work, long sentences don't.
- **Treating OpenAI's `prompt` as equivalent to Deepgram's keyterm.** The mechanisms are different; the prompt is a decoding hint, keyterm is a bias on recognition. Measure both; don't assume equivalence.
- **Fine-tuning before boosting.** Fine-tuning is the expensive last resort. Hit the boost ceiling first, then measure, then decide.
- **Not re-measuring after a vendor model update.** Vendor model version changes can reset your gains. Add the keyterm-error-rate to the CI harness that runs on model-version changes.

## 6. References

Primary — provider customisation docs (verify URLs at exercise-run time):

- Deepgram keyterm — `https://developers.deepgram.com/docs/keyterm`
- Deepgram keywords (legacy) — `https://developers.deepgram.com/docs/keywords`
- AssemblyAI customize vocabulary / word_boost — `https://www.assemblyai.com/docs/speech-to-text/pre-recorded-audio/customize-vocabulary`
- Google Cloud Speech-to-Text speech adaptation — `https://cloud.google.com/speech-to-text/docs/adaptation-model`
- Google Cloud Speech-to-Text v2 PhraseSets — `https://cloud.google.com/speech-to-text/v2/docs/phrase-sets`
- Microsoft Azure Custom Speech overview — `https://learn.microsoft.com/azure/ai-services/speech-service/how-to-custom-speech-overview`
- Microsoft Azure phrase list — `https://learn.microsoft.com/azure/ai-services/speech-service/improve-accuracy-phrase-list`
- OpenAI speech-to-text guide — `https://platform.openai.com/docs/guides/speech-to-text`
- OpenAI Whisper — `https://github.com/openai/whisper`
- faster-whisper — `https://github.com/SYSTRAN/faster-whisper`
- NVIDIA Riva / Parakeet — `https://developer.nvidia.com/riva`

Measurement tools:

- `jiwer` (WER / CER library) — `https://github.com/jitsi/jiwer`
- `kaldialign` (alignment for per-word analysis) — `https://github.com/pzelasko/kaldialign`

Related exercises:

- exercise-01 (STT provider decision matrix) — chose the providers this exercise configures.
- exercise-02 (streaming contract) — where the boost parameters are passed in on the streaming path.
- exercise-03 (confidence-informed handoff) — consumes the improved keyterm transcripts; many tier-1-with-wrong-answer errors resolve via this exercise.
- exercise-05 (WER benchmark) — the harness code this exercise reuses.

Standards referenced indirectly:

- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — governs the "measure before shipping" discipline of §2.3 / §3.
- OWASP Machine Learning Security Top 10 — `https://owasp.org/www-project-machine-learning-security-top-10/` — relevant when the boost list crosses into regulated terms (named individuals, PHI).
- MITRE ATLAS — `https://atlas.mitre.org/` — relevant when keyterm boosting is adversarially probed (speech crafted to force insertion of target keyterms).

Source policy note. All WER numbers in §2.4 are explicitly labelled as illustrative composite values intended to show the *shape* of a correct measurement; no vendor-specific accuracy claim in this document is presented as a real measurement. Vendor-specific parameter names, boost ranges, list-size caps, and model-version gating change frequently and carry `<!-- verify-at-runtime -->` markers; re-pull them against current vendor docs when running the exercise.
