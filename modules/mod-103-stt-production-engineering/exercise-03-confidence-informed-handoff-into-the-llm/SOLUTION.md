# exercise-03 — Confidence-Informed Handoff into the LLM — Reference Solution

## 1. Solution overview

This exercise sits at the ASR→LLM seam. The stock pattern — "take the final transcript, drop it into the LLM as the user turn" — leaves on the table the richest signal the ASR gave you: *how sure it was*. This exercise builds the contract that passes that signal downstream, defends the thresholds, and decides when to:

- pass the transcript as-is,
- pass the transcript with an uncertainty hint to the LLM,
- ask the user to repeat / confirm, or
- pass multiple hypotheses (N-best or alternatives).

The worked example uses the Deepgram message shape from exercise-02 and a telephony voice-agent persona (appointment-management). The contract generalises; the thresholds do not — you tune them against your own data.

Thresholds in this document are **illustrative starting values**. Treat them as the "where to start a sweep" numbers, not as production values.

## 2. Implementation

### 2.1 The envelope — what the ASR→LLM handoff looks like

Instead of passing just the transcript string, pass a structured user-turn envelope:

```python
# Worked user-turn envelope. Fields align to the Deepgram message shape from exercise-02.
user_turn = {
    "transcript": "i'd like to change my appointment",
    "confidence_mean": 0.864,     # mean of per-word confidences
    "confidence_min": 0.812,      # worst-word confidence
    "low_confidence_words": [     # words below the per-word threshold
        {"word": "appointment", "start": 3.86, "end": 4.16, "confidence": 0.812}
    ],
    "alternatives": [             # N-best, when the provider returns them
        {"transcript": "i'd like to change my appointment", "confidence": 0.864},
        {"transcript": "i'd like to change my disappointment", "confidence": 0.071}
    ],
    "audio_duration_s": 1.02,
    "language": "en-US",
    "asr_provider": "deepgram",
    "asr_model": "nova-2",
    "endpointed_via": "speech_final"  # or "vad_backup", see exercise-02
}
```

This envelope is the contract the downstream LLM turn consumes; everything the ASR knew that the LLM would otherwise have to infer is exposed as a field.

### 2.2 The decision ladder — four tiers

Given an envelope, the pipeline makes one of four choices. The ladder is intentionally narrow — most envelopes fall into tier 1 or 2.

| Tier | Condition (illustrative starting thresholds) | Action |
|---|---|---|
| 1 — **Pass clean** | `confidence_mean ≥ 0.88` AND `confidence_min ≥ 0.75` | Pass the transcript to the LLM as-is. No system-prompt annotation. |
| 2 — **Pass with hint** | `confidence_mean ∈ [0.70, 0.88)` OR 1–2 low-confidence words | Pass the transcript with a bracketed uncertainty marker inline for each low-confidence word, e.g., `"i'd like to change my [appointment?]"`, and a one-line hint in the system message: "Some words are uncertain; if the user's intent depends on them, ask to clarify before committing." |
| 3 — **Ask to confirm** | `confidence_mean < 0.70` OR the low-confidence word is a critical entity (name, number, date, amount, intent-bearing noun) | Do **not** call the LLM for a substantive answer. Instead, invoke a scripted clarifying turn ("Sorry — I thought you said *appointment*, is that right?"). Log both the original and the clarification. |
| 4 — **N-best pass** | A single utterance returned multiple hypotheses with the top two within a small confidence gap *and* the top hypothesis is tier-2 | Pass `alternatives[:2]` to the LLM with the hint: "Two readings; pick the one that fits the context; if neither fits, ask to clarify." Track resolution rate — if the LLM routinely cannot disambiguate, the tier-4 threshold is too generous. |

**Why four tiers and not a scalar.** A single threshold ("confidence > X, trust it") collapses the two failure modes — *the whole thing is mushy* (handled by tier 3) vs. *one load-bearing word is mushy* (handled by tiers 2 and 4). Different actions.

### 2.3 Threshold-sweep methodology

Starting thresholds are a guess. The sweep makes them data.

1. Collect 200–500 user turns from your production (or staged) traffic. Each turn needs: audio, ASR transcript, confidence fields, and a human-labelled truth (what the user *actually* said) + a human-labelled intent (what the user *wanted*).
2. For each threshold candidate (sweep `confidence_mean` on 0.70 / 0.75 / 0.80 / 0.85 / 0.88, and `confidence_min` similarly):
   - compute **false-pass rate**: tier-1 turns where the ASR was wrong on a load-bearing word.
   - compute **false-confirm rate**: tier-3 turns where the ASR was actually right and the clarification was unnecessary user friction.
3. Pick the operating point that trades the two acceptably for your product. Contact-centre routes tend to prefer low false-pass at the cost of higher false-confirm ("always ask"); smart-speaker consumer routes tend the other way ("never annoy").
4. Re-sweep whenever the ASR model version changes. **Confidence calibration is model-specific.** A Nova-2 confidence of 0.80 is not the same as an AssemblyAI Universal confidence of 0.80.

### 2.4 Entity-aware override

The threshold ladder alone is not enough. A 0.95-confidence "**apple**" is uncontroversial. A 0.85-confidence appointment time ("**four-thirty**" vs "**four-fifteen**") is not. The entity-aware override:

```python
CRITICAL_ENTITY_TYPES = {"DATE", "TIME", "PERSON_NAME", "PHONE", "MONEY", "ACCOUNT_ID", "ID"}

def should_clarify(envelope, detected_entities):
    tier = ladder_tier(envelope)  # tiers 1-4 from §2.2
    if tier in (3, 4):
        return True
    for ent in detected_entities:
        if ent["type"] in CRITICAL_ENTITY_TYPES:
            words_in_span = [w for w in envelope["words"] if _overlaps(w, ent)]
            if any(w["confidence"] < 0.90 for w in words_in_span):
                return True
    return False
```

Rule of thumb: critical entities get a tighter per-word threshold (0.90 vs. 0.75), and *any* word in the entity span below the tighter threshold forces clarification even if the global tier was 1. Rationale: a wrong name is worse than a wrong verb.

### 2.5 LLM-facing prompt fragment

When the pipeline is in tier 2 (pass with hint), include a small system-message fragment alongside the user turn:

```
System note (not from user): the transcription confidence for parts of the user's
turn is moderate. The user's turn may contain the following uncertainties:
  - [appointment?]  (confidence 0.81)
If any uncertain word would change your reply materially, ask the user to clarify
before answering. Do not speculate about uncertain entity values (names,
dates/times, numbers, IDs); if they are uncertain, confirm first.
```

Why not pass the uncertainty as plain text inside the user turn: because then the LLM is incentivised to answer the "user said [appointment?]" question literally rather than treating it as a hint. Keep the uncertainty in the system channel; keep the user turn as what the user actually said as best you could tell.

### 2.6 Logging / observability

Every turn should log the ladder tier, the thresholds in effect, and the resolution. In production you grep this log when a user complains the bot "didn't understand" — typically the complaint resolves to one of three patterns:

| Log pattern | Root cause |
|---|---|
| Tier 1 pass, downstream LLM gave a wrong answer, transcript was wrong | ASR made a confident error (homophone, accent, keyterm). Fix path is exercise-04 (keyterm prompting) or an ASR swap. |
| Tier 3 clarification, user got frustrated after N clarifications | Threshold too tight, or the audio is genuinely unintelligible (poor codec, background noise). Fix path is the audio layer, not the ladder. |
| Tier 2 pass, LLM asked to clarify when it didn't need to | LLM prompt too conservative given the hint. Loosen the system-note wording. |

## 3. Validation steps

A finished submission should pass each of the following checks.

1. **Envelope defined.** The ASR→LLM contract is specified as a structured object, not a string. All envelope fields from §2.1 are present and typed.
2. **Four-tier ladder.** The ladder has four tiers, each with an explicit condition and an explicit action. Thresholds are named as starting values, not as ground truth.
3. **Entity-aware override present.** Critical entity types are enumerated and the per-word threshold is tighter than the global threshold.
4. **Sweep methodology present.** The submission describes the data collection, the sweep variables, the two error rates, and the re-sweep trigger (model-version change).
5. **LLM-facing hint lives in the system channel, not the user turn.** The hint-vs-user-turn boundary is explicit.
6. **Logging named.** The three debug patterns (or an equivalent list) are explicitly addressed — the reviewer can see how the pipeline would be debugged when a user complains.
7. **N-best handling.** Either the pipeline passes alternatives when the provider returns them, or the submission explicitly justifies why N-best is not used (e.g., provider doesn't return it).

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Envelope design | 15 % | Structured contract with confidence_mean, confidence_min, word-level confidences, optional alternatives | All fields typed (TypedDict / pydantic); provider-agnostic field names decoupled from vendor field names |
| Decision ladder | 20 % | Four tiers with distinct conditions + actions | Tier boundaries justified against the two-failure-mode framing, not a single scalar |
| Thresholds | 15 % | Illustrative starting thresholds given with the labelled caveat | Thresholds tied to a sweep methodology that would land them on data |
| Entity-aware override | 15 % | Named critical entity types + tighter per-word threshold | Mechanism for extending the entity-type list per route (clinical, financial, legal) |
| LLM-facing prompt | 10 % | Hint delivered via system message, not user turn, with concrete wording | Wording tested against failure patterns (LLM speculating vs. LLM over-clarifying) |
| Sweep methodology | 15 % | Data collection + false-pass / false-confirm rates + re-sweep trigger | Sample-size reasoning (why 200–500) and cross-model-version re-sweep discipline |
| Logging / debugging | 10 % | Per-turn log of tier + resolution | The three canonical failure-pattern grep-recipes documented |

Scoring < 70 % is a fail-and-redo. The most common sub-70 failure: a single threshold with no sweep, and no entity awareness.

## 5. Common mistakes

- **Treating confidence as a scalar.** A single threshold conflates "the whole thing is mushy" and "one load-bearing word is mushy." They need different actions.
- **Calibration assumed to be cross-provider.** ASR confidence is *not* calibrated the same across providers. A 0.80 from Deepgram is a different beast from a 0.80 from AssemblyAI or Whisper. Sweep per provider.
- **Threshold frozen across model versions.** Vendor-side model upgrades silently shift the confidence distribution. Re-sweep on every model version change.
- **Passing raw partial-transcripts as the user turn.** Partials get revised; see exercise-02. Wait for the final.
- **Dumping the uncertainty into the user channel as literal text.** The LLM answers the literal brackets; your user-visible reply ends up quoting bracketed uncertainty markers. Put the hint in the system channel.
- **No entity awareness.** The fastest way for a voice agent to get a bad reputation is to confidently book "4:15" when the user said "4:30." Critical entities need a tighter threshold.
- **Passing all N-best to the LLM, always.** N-best is useful in tier 4 (two close hypotheses); it is noise in tiers 1 and 2. Only pass alternatives when the top hypothesis is borderline.
- **Clarifying twice in a row.** A reasonable product policy: at most one clarification before degrading to "let me connect you to a human" or "let's try that again." Infinite clarification loops are a user-abandonment pattern.
- **No log of the ladder decision.** When a user complains, you need to know what tier the pipeline was in and why. If the log doesn't carry the tier, you'll debug with your eyes.

## 6. References

Primary — provider documentation on confidence fields and alternatives:

- Deepgram word-level confidence + alternatives — `https://developers.deepgram.com/docs/confidence-score`
- AssemblyAI confidence + alternatives — `https://www.assemblyai.com/docs/speech-to-text/pre-recorded-audio/confidence-scores` and `https://www.assemblyai.com/docs/`
- Google Cloud Speech-to-Text word-level confidence — `https://cloud.google.com/speech-to-text/docs/basics#word-level-confidence`
- Azure Speech NBest results — `https://learn.microsoft.com/azure/ai-services/speech-service/get-started-speech-to-text`
- OpenAI Realtime API transcription event shape — `https://platform.openai.com/docs/guides/realtime`
- Whisper / whisper.cpp word-level timestamps + token logprobs — `https://github.com/openai/whisper`, `https://github.com/ggerganov/whisper.cpp`

Related exercises:

- exercise-02 (streaming contract) — produces the raw stream the envelope is built from.
- exercise-04 (custom vocab / keyterm prompting) — one of the fixes for recurring tier-1-pass-with-wrong-answer failures.
- exercise-05 (WER benchmark) — the dataset you build there is the natural input to the threshold sweep described in §2.3.

Standards referenced indirectly:

- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — the "measure, govern, document" discipline behind the sweep + logging stance of §2.3 / §2.6.
- OWASP Machine Learning Security Top 10 — `https://owasp.org/www-project-machine-learning-security-top-10/` — relevant when confidence-informed handoff is deliberately attacked (feed audio engineered to produce low-confidence tier-3 clarifications as a side-channel to extract system behaviour).
- MITRE ATLAS — `https://atlas.mitre.org/` — audio-adversarial / prompt-injection threat-model reference; the "entity-aware override" is the closest control to recommendation `AML.TA0005`-shaped evasion vectors in audio form.

Source policy note. All thresholds in this document are explicitly labelled as illustrative starting values; the real thresholds are produced by the sweep methodology against your own data. No vendor-side confidence-calibration claims are made beyond pointing at the provider doc URLs.
