# exercise-04 — Expressive vs. Neutral TTS A/B Drill — Reference Solution

## 1. Solution overview

This is a measurement + decision exercise. The learning target is **running an honest A/B between an expressive TTS voice (style tags, emotion, strong prosody) and a neutral voice on the same product copy, and defending the choice with numbers rather than taste**.

The output is not "expressive is better." It is a protocol that lets you tell, for *your* product and *your* listeners, when expressivity is worth the extra TTFB, the extra naturalness variance, and the extra prompt-tuning burden — and when the neutral voice is the mature choice.

This reference ships:

- The two-variant design (what counts as "expressive," what counts as "neutral").
- A corpus construction method (short, long, happy-path, failure-mode, number-heavy, apology, hand-off).
- A paired-listener MOS / preference protocol, with the statistical test.
- An operational-metrics side (TTFB, cost-per-minute, tag-stability under load).
- A decision rule that combines the two.
- Rubric and common-mistakes list.

The design is deliberately conservative: it's the smallest-N A/B that gives a defensible signal, not a 10,000-listener study.

## 2. Implementation

### 2.1 Target properties

A reviewer should see every one of these:

1. **Variant definitions are explicit and reproducible.** "Expressive" and "neutral" are pinned to a provider+voice_id+settings+style_tag tuple; both variants are pinned to the *same* text.
2. **Corpus is diverse.** At least seven utterance classes (§2.3). A biased corpus (all short greetings) gives a biased answer.
3. **Blinding.** Listeners don't know which variant is A and which is B. Variant order is counterbalanced.
4. **Primary metric is a listener-reported preference** (pairwise, forced-choice), not MOS alone. MOS is secondary.
5. **Operational metrics are measured in parallel.** Expressivity costs TTFB, cost per minute, and sometimes stability. Expressivity-wins-on-preference isn't the whole decision.
6. **Statistics match the design.** Paired preference → two-sided binomial / McNemar; MOS → paired-t with CI. Sample size is justified by MDE, not pulled from a hat.
7. **One decision, with a flip condition.** A one-line call: *expressive for X route, neutral for Y route; here's what would flip it*.

### 2.2 Variant definitions (pinned)

**Variant N — Neutral.**

- Provider: `ElevenLabs` *(for illustration; swap to your winner from exercise-01)*
- Voice: `voice_id = "<neutral-default-voice-id>"`
- Settings: `stability = 0.65`, `similarity_boost = 0.85`, no style tags, no `<prosody>` tuning
- SSML: `<speak>…</speak>` wrapper only
- Rationale: the default "professional read" the provider ships; the no-tuning baseline

**Variant E — Expressive.**

- Provider: `ElevenLabs` *(same provider as variant N — expressivity, not vendor, is the independent variable)*
- Voice: `voice_id = "<expressive-voice-id>"`
- Settings: `stability = 0.35`, `similarity_boost = 0.85`, `style = 0.6`, speaker-boost on
- SSML: provider-supported style surface + measured `<prosody rate="95%" pitch="+1st">` on the greeting / sign-off
- Rationale: provider's expressive path with the exercise-03 restraint rules applied

Pinning both variants to the *same provider* isolates expressivity as the independent variable. If you also change provider, you're running two experiments and will not be able to separate the two effects — a common pitfall.

### 2.3 Corpus (what gets spoken by both variants)

Seven utterance classes, at least four utterances per class, 28 utterances minimum. All utterances come from (or are faithful paraphrases of) your product's real reply corpus — not vendor demo scripts.

| # | Class | Why it's in the corpus | Example |
|---|---|---|---|
| 1 | Short greeting (≤ 10 words) | Where expressivity is most often wanted | "Hi, this is Weizenmann Support — how can I help?" |
| 2 | Medium informational (20–40 words) | The bread-and-butter length | "Your order ORD-10428-B for Pan Dulce Clásico has shipped and should arrive by Friday." |
| 3 | Long narrative (60+ words) | Where expressivity drift or over-acting shows up | A three-sentence policy explanation |
| 4 | Number / entity-heavy | Where expressivity can break pronunciation | Order IDs, prices, phone numbers, tracking URLs |
| 5 | Apology / de-escalation | High-stakes emotional register | "I'm really sorry about the delay. Let me pull up your account right now." |
| 6 | Hand-off / disclaimer | Register must remain neutral regardless of voice | "I'll transfer you to a human agent now — this may take up to a minute." |
| 7 | Short refusal / limit statement | Register must not sound sarcastic | "I can't make changes to this order after it's shipped. I can help with a return instead." |

Classes 4, 6, 7 are the diagnostic classes — expressive voices often lose here even when they win overall. The corpus must include them to produce an honest answer.

### 2.4 Listener protocol

**Participants.** 20 listeners, drawn from the product's actual user demographic where feasible. 20 is the floor for a two-sided binomial preference test with 28 pairs and a medium effect; smaller samples publish on taste, not evidence.

**Task per listener.**

1. Receive 28 pairs (A/B order random; counterbalanced across listeners so neither position gets an advantage).
2. For each pair, hear both renderings of the same utterance, then answer two forced-choice questions:
   - **Preference.** "Which of A or B would you rather hear from this product?" (A / B / no-preference allowed but encoded as 0.5 to both)
   - **MOS (per rendering).** "Rate the naturalness of each clip on a 5-point scale (1 = very bad, 5 = very good)." (ITU-T P.808 MOS scale.)
3. Short free-text comment per pair, optional.

**Blinding.** Listeners do not know which clip is expressive; the provider name is never shown. Clips are served by opaque filename (`clip_0011_a.wav`).

**Counterbalancing.** Half the listeners hear "expressive first" on even-numbered pairs; the other half on odd-numbered pairs. Prevents a "first-clip-of-the-pair" bias.

**Audio normalisation.** Both variants are peak-normalised to the same target loudness (−16 LUFS) before serving. An un-normalised A/B leaks loudness as a confound.

**Platform.** A headless listening harness is sufficient (no paid platform required); if you use a paid platform (Prolific, MTurk, Toloka) note the pre-screen and the median task time.

### 2.5 Operational metrics (measured in parallel)

Each variant is also measured on the exercise-02 harness under representative load:

| Metric | How | Why |
|---|---|---|
| TTFB P50 / P95 / P99 | exercise-02 `measure.py` | Expressive voices often cost 50–150 ms more TTFB; this is the thing listener-preference studies silently hide |
| Inter-chunk gap distribution | exercise-02 harness | Expressive paths sometimes jitter more |
| Cost per audio-minute | Provider pricing × measured duration | Expressive voices sometimes bill at a premium tier |
| Stability under re-synth | Run the same utterance 10× | Expressive voices with stability < 0.5 can drift noticeably run-to-run; this matters for A/B replicability |
| Lexicon honouring | Submit exercise-03's PLS; audit outputs | Some expressive paths quietly ignore `<phoneme>` for stylistic consistency |

### 2.6 Statistics

**Primary — pairwise preference.**

Null hypothesis: `p(expressive preferred) = 0.5`. Count pairs where the majority of listeners prefer expressive (ties = 0.5). Test with a two-sided binomial / sign-test on pair-level outcomes; alternatively, McNemar if you treat each (listener × pair) as the unit. Report the exact binomial CI for `p`, not just a point estimate.

**MDE sanity check.** With 28 pairs and 20 listeners, the setup detects a pair-level preference rate of ≈ 0.72 vs 0.50 at α = 0.05 / power 0.8. Shift smaller than ≈ 20 percentage points is undetectable at this N — flag that explicitly in the report.

**Secondary — MOS.**

Paired-t on per-utterance mean MOS. Report mean difference, 95 % CI, and Cohen's d. MOS is bimodally noisy across listeners; a 0.2-MOS difference is typically under the detectable floor at N=20.

**Confound checks.**

- Loudness (should be equal after normalisation; verify with a loudness measurement).
- Variant order (no main effect of A-first vs B-first).
- Listener fatigue (no trend across trial-number).

### 2.7 Decision rule

Combine the primary and operational metrics into a single rule you can commit to in writing:

```
decision := expressive
  IF   pairwise_preference_rate(E vs N) ≥ 0.60
       AND binomial_pvalue ≤ 0.05
       AND ttfb_p95_delta(E - N) ≤ +80 ms                 # within the latency budget (mod-109)
       AND cost_per_minute_ratio(E / N) ≤ 1.5
       AND stability_variance_E ≤ 2 × stability_variance_N
  ELSE neutral
```

The thresholds are route-specific; the shape is universal. "Expressive wins on preference" is not sufficient — the operational envelope must also hold.

### 2.8 Example report (worked, illustrative — not real numbers)

> **Route A — inbound contact-centre voice agent.**
>
> Pairwise preference for expressive (variant E): 18 of 28 pairs (0.64, 95 % CI 0.44–0.81, p = 0.185). MOS: E 4.1 vs N 3.9 (Δ 0.2, 95 % CI −0.05 to +0.45). TTFB P95: E 420 ms vs N 290 ms (Δ +130 ms). Cost per minute: E/N = 1.0 (same tier). Stability (re-synth MOS σ): E 0.31 vs N 0.12.
>
> **Call: neutral.** Preference lift is not significant at N=20/28; MOS Δ is within CI; the TTFB P95 delta of +130 ms exceeds the mod-109 route budget by 50 ms. Expressivity would have to beat preference p ≤ 0.05 *and* fit within +80 ms TTFB to justify the switch.
>
> **What would flip this.** A hero-call route where the product deliberately trades TTFB for warmth (e.g., bereavement call-centre) re-weights the latency ceiling; stability drift would still be a concern.
>
> *All numbers above are illustrative composites — see §2.6 for the computation and the exercise-02 harness for the TTFB measurement protocol.*

## 3. Validation steps

1. **Variant pinning.** The provider+voice_id+settings+SSML tuple for N and E are committed to the repo (`variants.yaml` or equivalent); both variants ran against the identical text set.
2. **Corpus coverage.** At least 7 classes × ≥ 4 utterances; the diagnostic classes (4, 6, 7) are present.
3. **Loudness equalised.** Both variants peak-normalised / loudness-normalised; a measurement (ffmpeg loudnorm or equivalent) shows ≤ 0.5 LUFS difference.
4. **Blinding real.** Listener-facing filenames are opaque; the task page does not name the provider; the task copy does not use words like "expressive."
5. **Counterbalance real.** A/B order is balanced across listeners; a post-hoc test shows no main effect of position.
6. **Operational metrics measured.** TTFB P50/P95/P99 for both variants, from exercise-02 harness, in the same busy-hour window. Cost per minute for both, per the provider pricing page at report time.
7. **Statistics match the design.** Preference — binomial / sign-test (or McNemar) with exact CI; MOS — paired-t with Cohen's d. MDE is reported alongside.
8. **Decision rule committed in writing.** The rule from §2.7 is in the report; the chosen value for each threshold is stated.
9. **Report is auditable.** All 28 pair-level preferences, per-listener MOS, and operational metrics are in a CSV in the repo. The report script re-derives the headline numbers from the CSV; a reviewer can re-run.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Variant definition | 10 % | Same provider, different voice/settings; both variants pinned | Also runs a "same voice, style-tag on/off" variant to isolate style tag effect from voice choice |
| Corpus design | 10 % | ≥ 7 classes × ≥ 4 utterances; diagnostic classes present | Corpus sampled from real product traffic with sampling method documented |
| Blinding & counterbalance | 10 % | Opaque filenames; A/B position counterbalanced; post-hoc position-effect check | Multi-rater listener reliability (Krippendorff α or similar) reported |
| Listener protocol | 10 % | ≥ 20 listeners, pairwise forced choice, MOS per clip | Pre-screen and median task time disclosed if paid platform |
| Loudness / audio hygiene | 5 % | Both variants loudness-normalised; measurement included | Codec-robustness A/B also run (variants at 8 kHz µ-law) |
| Operational metrics | 15 % | TTFB P95 delta + cost delta + stability delta reported for both variants | Metrics come from the exercise-02 harness and are reproducible |
| Statistics | 15 % | Binomial / McNemar for preference; paired-t for MOS; MDE stated | Confound checks (loudness, order, fatigue) itemised |
| Decision rule | 15 % | Rule from §2.7 committed; chosen thresholds stated | Flip condition tied to a measurable (not vibes) |
| Reproducibility | 10 % | Pair-level CSV + report script committed; re-run produces the headline numbers | CI in the repo runs the report script on every push |

Scoring < 70 % is a fail-and-redo. The most common sub-70 failures are: (a) the expressive variant is loudness-boosted and listeners vote for "louder" not "more expressive," (b) N < 10 listeners so nothing is detectable, (c) the decision is "expressive wins!" with no reference to TTFB delta.

## 5. Common mistakes

- **Taste-driven decision.** "I like the expressive one." The entire exercise exists to replace this with a number. Even an N=20 study beats intuition by a lot when the numbers go the other way than your ear.
- **Un-normalised loudness.** Expressive voices are often *louder* out of the box. Listeners reliably prefer louder clips, and this effect is huge — it will dominate your preference signal if left uncontrolled.
- **Changing two things at once.** Running expressive-provider-A vs neutral-provider-B tests *provider*, not expressivity. Pin provider, pin voice family where possible.
- **Vendor-demo corpus.** Vendor demo scripts are tuned to the vendor's strengths. Use your real product copy.
- **No diagnostic classes.** Expressive voices often win on greetings and lose on apologies, refusals, and number-heavy clips. A corpus without 4/6/7 hides this.
- **Ignoring stability variance.** Expressive voices with stability < 0.5 can re-synthesise differently each run. If your regression tests (mod-112) expect a stable reference, stability variance is a *disqualifier*, not a nice-to-have.
- **Ignoring TTFB cost.** Expressive wins on preference, loses on TTFB, net is a worse product. The decision rule in §2.7 forces you to acknowledge this trade.
- **Reporting "mean preference" without CI.** A point estimate of 0.60 at N=20 is not significantly different from 0.50. Always report the binomial CI.
- **Pretending MOS is reliable at N=20.** It isn't. MOS deltas under ≈ 0.4 are typically inside the CI; don't claim an effect you can't detect.
- **A one-shot listening session.** Fatigue is real; break sessions into ≤ 15-minute blocks or report a fatigue check.
- **Treating expressive as a binary.** Provider style surfaces usually offer multiple knobs (stability, similarity, style, speaker-boost). The "expressive" variant you chose is *one* operating point; call out that the result is for *this operating point*, not for "expressive in general."

## 6. References

Standards and perceptual-audio methodology:

- ITU-T Recommendation P.808 — "Subjective evaluation of speech quality with a crowdsourcing approach" — `https://www.itu.int/rec/T-REC-P.808` — the MOS methodology the listener protocol is adapted from.
- ITU-T Recommendation P.800 — "Methods for subjective determination of transmission quality" — `https://www.itu.int/rec/T-REC-P.800` — the parent MOS framework.
- EBU R 128 loudness normalisation — `https://tech.ebu.ch/publications/r128/` — the loudness normalisation convention for audio A/B tests (−16 LUFS is one anchor; −23 LUFS is the broadcast norm).

Provider expressivity / style-tag documentation (verify URLs at exercise time; vendor doc trees are frequently reorganised):

- ElevenLabs voice settings (stability, similarity, style, speaker-boost) — `https://elevenlabs.io/docs/speech-synthesis/voice-settings`
- Microsoft Azure AI Speech `<mstts:express-as>` (styles, roles) — `https://learn.microsoft.com/azure/ai-services/speech-service/speech-synthesis-markup-voice`
- Amazon Polly `<amazon:emotion>` and NTTS styles — `https://docs.aws.amazon.com/polly/latest/dg/ntts-emotions.html`
- Google Cloud Text-to-Speech expressive voices (Chirp 3 HD / Studio) — `https://cloud.google.com/text-to-speech/docs/voices`
- OpenAI Realtime API — `https://platform.openai.com/docs/guides/realtime`
- Deepgram Aura voices — `https://developers.deepgram.com/docs/tts-voices`

Related curriculum anchors:

- TTFB measurement protocol — `mod-104 exercise-02`
- Latency budget framing — `mod-109-latency-budget-architecture`
- Regression / replay evaluation — `mod-112-voice-multimodal-eval-and-observability`

Standards referenced indirectly:

- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — frames the "report MDE and CI, not point estimates" posture.

Source policy note. All numbers in the example report (§2.8) are illustrative composites, labelled as such. ITU-T P.808 / P.800 and EBU R 128 are the authoritative references for the listener protocol and loudness normalisation respectively. Provider style-tag URLs are included for traceability and are expected to change; confirm against each provider's current documentation at exercise time.
