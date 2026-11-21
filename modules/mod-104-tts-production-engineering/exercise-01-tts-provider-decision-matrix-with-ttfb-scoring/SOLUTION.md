# exercise-01 — TTS Provider Decision Matrix With TTFB Scoring — Reference Solution

## 1. Solution overview

This is a selection exercise. There is no single correct answer — the correct answer is **a defensible matrix built from provider documentation and small reproducers you actually ran**, that commits to one shortlisted provider per route and names what would flip the decision. It mirrors the shape of mod-103 exercise-01 (STT), but TTS adds two dimensions that STT didn't own:

- **TTFB** (time to first *audio byte* returned, measured on a real streaming request) — the cell that will dominate the "does the agent sound alive" judgement.
- **Expressivity / controllability** — style tags, prosody, emotion, SSML support — because TTS output is a product surface in a way that STT input is not.

This reference shows the dimensions a reviewer should expect to see weighted, a worked matrix across seven candidate providers plus a self-hosted row, decision rationale with "what would flip this" sentences for two runners-up, and a rubric + common-mistakes list.

Vendor pricing, model names, and "fastest" claims move every quarter. All vendor-specific numbers below are either (a) drawn from the vendor's public docs at the time this reference was written and marked `<!-- verify-at-runtime: confirm at exercise-run time -->`, or (b) clearly labelled as **illustrative composite** when used as a measurement example. Re-pull every cell at exercise time.

## 2. Implementation

### 2.1 Candidate list (what gets into the matrix)

The seven commercial + one self-hosted row below are the realistic English-centric shortlist for production voice agents in 2025–2026. Pick 4–6 for your own matrix; measuring more than that is wasted effort before you have signal.

- **ElevenLabs** — streaming-first commercial TTS, large voice library + instant-voice-clone, strong expressivity. (`https://elevenlabs.io/docs`)
- **Cartesia (Sonic)** — streaming-first, built around aggressive TTFB targets. (`https://docs.cartesia.ai/`)
- **Deepgram Aura** — streaming TTS designed to pair with Deepgram STT inside a single voice-agent stack. (`https://developers.deepgram.com/docs/tts-getting-started`)
- **OpenAI TTS / Realtime audio out** — batch TTS (`tts-1`, `tts-1-hd`) and Realtime API audio-out path. (`https://platform.openai.com/docs/guides/text-to-speech`, `https://platform.openai.com/docs/guides/realtime`)
- **Google Cloud Text-to-Speech** — WaveNet / Neural2 / Studio / Chirp 3 HD voices; broad multilingual coverage. (`https://cloud.google.com/text-to-speech/docs`)
- **Microsoft Azure AI Speech (TTS)** — Neural voices + custom-neural-voice (CNV) with consent-gated cloning. (`https://learn.microsoft.com/azure/ai-services/speech-service/text-to-speech`)
- **Amazon Polly** — Neural + Long-Form + Generative voices; AWS-anchored customers. (`https://docs.aws.amazon.com/polly/`)
- **Self-hosted open-weight — Coqui XTTS v2, Piper, VITS, Bark, Parler-TTS** — covered as a single row. (`https://github.com/coqui-ai/TTS`, `https://github.com/rhasspy/piper`, `https://github.com/jaywalnut310/vits`, `https://github.com/suno-ai/bark`, `https://github.com/huggingface/parler-tts`)

Providers deliberately omitted and why:

- **PlayHT, Resemble, WellSaid, Murf, Speechify** — valid but narrower fit; the matrix shape generalises.
- **Hume EVI** — treated as an orchestration layer, not a raw TTS provider in this exercise. (`https://dev.hume.ai/`)

### 2.2 Dimensions (what gets weighted)

| # | Dimension | Why it matters | Suggested weight (inbound CX) |
|---|---|---|---|
| 1 | **TTFB (time-to-first-audio-byte) on streaming endpoint** | The single number that most predicts perceived responsiveness; sets the LLM→speak budget. | **25 %** |
| 2 | Audio-chunk steadiness (jitter, underrun risk under load) | TTFB alone is not enough — an early chunk followed by a 400 ms gap sounds worse than a slightly later steady stream. | 10 % |
| 3 | Naturalness / MOS on your in-domain copy | Downstream user-facing quality. | 15 % |
| 4 | Expressivity + controllability (SSML, style tags, prosody, emotion, pauses) | What lets the agent actually *speak* the brand voice. See exercise-03 and exercise-04. | 10 % |
| 5 | Voice coverage (library size, language/dialect spread, custom voice support) | Determines whether you can match a brand without cloning. | 5 % |
| 6 | Streaming protocol surface (WebSocket, chunked HTTP, gRPC, barge-in / cancel) | Barge-in-safe cancels, chunk pacing — see exercise-02 and mod-106. | 10 % |
| 7 | Audio format options (Opus, PCM 16k/24k/48k, µ-law 8k, MP3) | Telephony needs µ-law 8k; WebRTC needs Opus. A wrong format = a resample tax on the hot path. | 5 % |
| 8 | Price per audio-second (and streaming vs batch split) | Unit economics. TTS unit price is typically 3–10× STT per second. | 10 % |
| 9 | Compliance posture, data residency, BYOK, voice-clone consent surface | HIPAA/GDPR/PCI constraints; see exercise-05 for the clone-specific controls. | 5 % |
| 10 | Operational maturity (SLA, status page, incident history, support) | What you depend on when paged at 03:00. | 5 % |

Total 100 %. Re-weight — don't keep the suggestion. For an audiobook/long-form product you'd drop TTFB and move naturalness up; for a games companion you'd push TTFB above 30 %.

### 2.3 Worked matrix — inbound contact-centre route (illustrative)

Scoring key: `+` meets bar, `++` exceeds, `~` partial, `−` below bar, `??` requires measurement at exercise time. Cells labelled `??` are *the point* — a mature reviewer expects every high-weight cell to be filled from your own small-N measurements.

| Dimension (weight) | ElevenLabs | Cartesia Sonic | Deepgram Aura | OpenAI TTS / Realtime | Google Cloud TTS | Azure AI Speech | Amazon Polly | Self-hosted (XTTS / Piper) |
|---|---|---|---|---|---|---|---|---|
| 1. TTFB streaming (25%) | ++ Streaming is a first-class path; sub-500 ms TTFB advertised. <!-- verify-at-runtime --> | ++ Positions aggressive TTFB (sub-100-ms class claims). <!-- verify-at-runtime --> | ++ Streaming-first product; sub-300 ms targets. <!-- verify-at-runtime --> | ~ Realtime API streams audio out; TTFB depends on model and connection warm-up. <!-- verify-at-runtime --> | ~ Streaming available on some voices (e.g. Chirp 3 HD / Studio); Studio voices are not streaming. <!-- verify-at-runtime --> | + Streaming via Speech SDK. <!-- verify-at-runtime --> | + Streaming via `SynthesizeSpeech` streaming output on applicable voices. <!-- verify-at-runtime --> | ~ Piper is fast on CPU; XTTS first-byte depends on your serving layer (chunked inference, not a native streaming vocoder). |
| 2. Chunk steadiness (10%) | ?? | ?? | ?? | ?? | ?? | ?? | ?? | ?? |
| 3. Naturalness (15%) | ++ Industry-regarded naturalness on expressive English. | + Naturalness targeted around steady low-latency. | + Narrower voice library; competitive naturalness. | + `tts-1-hd` + Realtime voices rated highly. | + Chirp 3 HD / Studio voices are strong. | + Neural / Multilingual neural voices. | + Neural + Long-Form + Generative voices. | ~ XTTS/Parler good on clean prompts; Piper is efficient-not-expressive; Bark creative but unstable. |
| 4. Expressivity / SSML (10%) | ++ Style tags + stability / similarity controls + speaker boost. (`https://elevenlabs.io/docs/speech-synthesis/voice-settings`) | + Prosody + pacing controls; SSML subset. (`https://docs.cartesia.ai/`) | + Growing SSML subset. (`https://developers.deepgram.com/docs/tts-ssml`) | ~ Instruction-style prompting on Realtime voices; limited SSML. | ++ Full SSML. (`https://cloud.google.com/text-to-speech/docs/ssml`) | ++ Full SSML + style/role tags on neural voices. (`https://learn.microsoft.com/azure/ai-services/speech-service/speech-synthesis-markup`) | ++ Full SSML. (`https://docs.aws.amazon.com/polly/latest/dg/ssml.html`) | + Depends on model; XTTS respects prompt-level style; most OSS vocoders don't do SSML. |
| 5. Voice coverage (5%) | ++ Large library + instant voice cloning. | + Library growing. | ~ Narrower voice library. | + Fixed list of Realtime voices. | ++ Very wide language + voice list. | ++ Wide list + custom neural voice. | ++ Wide list. | ++ Clone-your-own freedom with the strongest compliance story (your voices, your consent — see exercise-05). |
| 6. Streaming protocol (10%) | ++ WebSocket + HTTP chunked; barge-in supported via SDK. | ++ Low-latency WebSocket. | ++ WebSocket + HTTP chunked. | + Realtime WebSocket. | + gRPC + REST streaming. | + WebSocket. | + Streaming HTTP response. | You own this — your streaming is as good as your serving stack. |
| 7. Audio format options (5%) | ++ MP3, PCM, µ-law, Opus. | ++ PCM / Opus options. | ++ Linear16, mu-law, alaw, Opus. | ~ Limited to a handful of formats. <!-- verify-at-runtime --> | ++ LINEAR16 / MULAW / ALAW / OGG_OPUS / MP3. | ++ Riff / mu-law / Opus / various. | ++ pcm / mp3 / ogg_vorbis / pcm 8k mu-law. | Your choice. |
| 8. Price per second (10%) | ?? | ?? | ?? | ?? | ?? | ?? | ?? | GPU-hour cost; break-even depends on utilisation — see §2.5. |
| 9. Compliance (5%) | SOC 2 / HIPAA on applicable plans; voice-clone consent model is provider-managed. (`https://elevenlabs.io/trust`) | Posted security docs. (`https://cartesia.ai/security`) <!-- verify-at-runtime --> | Public trust page. (`https://deepgram.com/trust`) | SOC 2 / HIPAA under BAA. (`https://trust.openai.com`) | GCP compliance programs. (`https://cloud.google.com/security/compliance`) | Azure compliance programs. (`https://learn.microsoft.com/azure/compliance/`) | AWS compliance. (`https://aws.amazon.com/compliance/`) | In-VPC / on-prem — strongest when the customer requires it; consent surface is now *your* problem (see exercise-05). |
| 10. Ops maturity (5%) | Status page. (`https://status.elevenlabs.io/`) | Status page. (`https://status.cartesia.ai/`) <!-- verify-at-runtime --> | Status page. (`https://status.deepgram.com/`) | Status page. (`https://status.openai.com/`) | GCP status. (`https://status.cloud.google.com/`) | Azure status. (`https://status.azure.com/`) | AWS Health. (`https://health.aws.amazon.com/`) | You *are* the status page. |

The `??` cells are deliberate — a matrix pre-filled from vendor marketing pages is a pitch deck, not a decision. Measure the top three dimensions yourself.

### 2.4 Decision rationale — two routes

**Route A — inbound contact-centre voice agent (English, US, HIPAA-adjacent).**

*Winner:* ElevenLabs *or* Cartesia Sonic in production, Deepgram Aura as a warm backup paired naturally with Deepgram STT, self-hosted off the hot path.

Reasoning tied to weights:

- Dimension 1 (TTFB streaming, 25 %) and Dimension 2 (chunk steadiness, 10 %) together are 35 % of the matrix. Whichever of {ElevenLabs, Cartesia, Deepgram Aura} your *measured* TTFB + steadiness ranks highest should win; this is the single most important measurement in the module.
- Dimension 4 (expressivity, 10 %) adds weight for brand-voice products. ElevenLabs typically leads on expressivity; Cartesia leads on TTFB; the trade-off is real.
- Dimension 7 (audio format, 5 %) — all three serve µ-law 8k cleanly for telephony; no resample tax.
- Dimension 9 (compliance, 5 %) — HIPAA-eligible on applicable plans with each of them; if the customer already has a signed BAA with Azure or GCP, that re-anchors the winner. See exercise-05 for the clone-consent dimension specifically.
- Self-hosted is a *bad* fit for this route specifically: streaming TTS from open weights is usable but every ms of chunk-boundary jitter is yours to debug, and the clone-consent governance (exercise-05) now lands on your team rather than the vendor's.

**What would flip this?**

- *TTFB flip.* If measured P95 TTFB on a provider is ≥ 100 ms above your budget in the busy-hour window, drop them regardless of naturalness. A beautiful voice that arrives 300 ms late ruins the dialogue.
- *Brand-voice flip.* If the product requires a specific cloned voice, the winner moves to whichever provider's consent + provenance workflow (exercise-05) your legal team will sign off on — Azure Custom Neural Voice is the most paperwork-heavy; ElevenLabs Professional Voice Clone and Cartesia voice onboarding are lighter-weight.
- *Cost flip.* If measured usage × unit price × 12 months × 2 (for the warm backup) crosses the self-hosted break-even (§2.5), revisit self-hosted — but only if ops maturity (dimension 10) is budgeted.

**Route B — outbound reminder voice agent (English + Spanish, lower concurrency, quality-sensitive).**

*Winner:* ElevenLabs for English; Google Cloud TTS (Chirp 3 HD) or Azure Neural for the Spanish route because their multilingual coverage is historically stronger and dialect coverage matters more when per-language volume is low.

Reasoning: when per-language call volume is low, a cloned voice per language is not economically justified, so you rely on the provider's native multilingual library.

**What would flip this?** A mandate to run a single provider for operational simplicity re-weights dimension 10 (ops maturity) and 6 (streaming protocol) upward and pushes toward Google or Azure as the single-provider answer.

### 2.5 Break-even sketch (self-hosted vs commercial TTS)

```
monthly commercial cost ≈ seconds_of_audio_per_month × ($ / second_commercial_tier)
monthly self-hosted cost ≈ GPU_cost_per_hour × hours_billed_per_month
                           + storage + bandwidth + on-call + voice-library curation + consent-audit admin
break-even seconds ≈ (GPU_cost_per_hour × hours_billed + ops_overhead)
                     / ($ / second_commercial_tier)
```

The TTS-specific failure mode is to count only the GPU line. Beyond the standard ops overhead (on-call, retraining, upgrade testing — 20–50 % of the GPU line) you now also own:

- The **consent + provenance surface** (exercise-05) — forms, revocation workflows, watermark attestation.
- The **voice-library curation** — generating, auditioning, versioning, and permissioning voice IDs.
- **Live-safety review** when a voice is reported mis-used.

These are not GPU line items. Name them explicitly in the matrix; "self-hosted is cheaper" without them is wrong in review.

## 3. Validation steps

A finished matrix should pass each check.

1. **Dimension completeness.** All 10 dimensions present and weighted; weights sum to 100 %. Any weight ≤ 5 % is justified.
2. **Provider spread.** At least 4 commercial providers *and* at least 1 self-hosted option.
3. **Measured cells.** For TTFB (25 %), chunk steadiness (10 %), and naturalness (15 %), you have your *own* measurement for at least two providers. No `??` in the top three dimensions for shortlisted providers.
4. **TTFB measured correctly.** Point-value is reported as P50 / P95 / P99, in the busy-hour window, after warm-up, on the audio format and sample rate you actually ship. Not a one-shot cold-call number.
5. **Chunk steadiness isn't collapsed into TTFB.** You measured and reported inter-chunk interval distribution, not just first-byte time.
6. **"What would flip this" sentences** for two runners-up, each tied to a measurable.
7. **Break-even sketch** names voice-library curation and consent-audit admin explicitly, not just GPU-hours.
8. **Freshness check.** Every vendor cell has a doc URL; pricing / "fastest" cells carry `<!-- verify-at-runtime: confirm at exercise-run time -->` markers.
9. **Compliance isn't a checkbox.** Where HIPAA / PCI / GDPR matters, name the control family (data residency, encryption-at-rest, BYOK, consent-of-record) rather than "yes."

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Dimension list + weights | 10 % | ≥ 8 dimensions summing to 100 %, each weight justified | All 10, with route-specific re-weighting shown for ≥ 2 routes |
| Provider shortlist | 10 % | ≥ 4 commercial + 1 self-hosted, each with a doc URL | Explicit "excluded and why" list |
| TTFB measurement | 25 % | Own P95 TTFB measurement for ≥ 2 providers under representative load | Measurement includes P50/P95/P99 + inter-chunk interval distribution + sample size |
| Chunk-steadiness measurement | 10 % | Inter-chunk gap reported separately from TTFB | Underrun / jitter under concurrency shown |
| Decision + flip conditions | 15 % | One winner per route; each with a "what would flip" sentence tied to a measurable | Flip conditions use minimum-detectable-effect reasoning |
| Break-even sketch | 10 % | Self-hosted-vs-commercial quantified at least roughly | Ops-overhead line items itemised — including consent/provenance admin |
| Cell freshness | 10 % | Every vendor cell has a URL + pull date | Pricing / "fastest" cells marked `<!-- verify-at-runtime -->` |
| Compliance detail | 10 % | Where compliance matters, named control family + source | Consent-of-record workflow (voice cloning) addressed — points to exercise-05 |

Scoring < 70 % is a fail-and-redo. The most common sub-70 failure is a matrix with marketing-page TTFB numbers and no measurement.

## 5. Common mistakes

- **Scoring TTFB from marketing pages.** Every provider advertises "fastest." Your measurement on your telephony leg at your region at busy hour is what counts.
- **Reporting a single mean TTFB.** Latency distributions are heavy-tailed; the P95 is the user experience, not the mean. Report P50 / P95 / P99 and sample size, with warm-up excluded.
- **Treating "streaming" and "batch" as one column.** Nearly every provider has both endpoints with different latency, quality, and price. Build the matrix per *endpoint*, not per vendor.
- **Ignoring chunk steadiness.** A TTFB of 180 ms followed by a 500 ms gap before the second chunk is worse than a steady 350 ms stream. Measure both.
- **One-provider matrix.** If only one provider is evaluated, it's a pitch deck, not a decision.
- **Forgetting telephony audio format.** µ-law 8 kHz is the dominant telephony leg; if the provider only serves 24 kHz PCM, you pay a resample tax on every chunk. Weight dimension 7 appropriately.
- **"Self-hosted is cheaper, obviously."** Only after utilisation × unit-price crosses break-even *and* you've budgeted voice-library curation + consent admin (exercise-05). Call the latter out explicitly.
- **Compliance = "yes."** HIPAA / PCI / GDPR / FedRAMP are not scalars. For voice-cloning routes, the compliance row must additionally address consent-of-record and revocation (exercise-05).
- **Expressivity measured by listening to the vendor's demo page.** Vendor demos are hand-tuned scripts. Measure your own lines, including your worst-case long-tail utterances.
- **No multilingual self-measurement.** Vendor-claimed language coverage is advertised; measured naturalness on your dialect is not.

## 6. References

Primary — provider documentation (verify URLs still resolve at exercise-run time; vendor doc trees are frequently reorganised):

- ElevenLabs docs — `https://elevenlabs.io/docs`
- ElevenLabs voice settings / controllability — `https://elevenlabs.io/docs/speech-synthesis/voice-settings`
- ElevenLabs trust / compliance — `https://elevenlabs.io/trust`
- ElevenLabs status — `https://status.elevenlabs.io/`
- Cartesia docs — `https://docs.cartesia.ai/`
- Deepgram Aura (TTS) — `https://developers.deepgram.com/docs/tts-getting-started`
- Deepgram TTS SSML — `https://developers.deepgram.com/docs/tts-ssml`
- Deepgram trust — `https://deepgram.com/trust`
- OpenAI text-to-speech — `https://platform.openai.com/docs/guides/text-to-speech`
- OpenAI Realtime API — `https://platform.openai.com/docs/guides/realtime`
- OpenAI trust — `https://trust.openai.com`
- Google Cloud Text-to-Speech docs — `https://cloud.google.com/text-to-speech/docs`
- Google Cloud Text-to-Speech SSML — `https://cloud.google.com/text-to-speech/docs/ssml`
- Google Cloud compliance — `https://cloud.google.com/security/compliance`
- Microsoft Azure AI Speech (TTS) — `https://learn.microsoft.com/azure/ai-services/speech-service/text-to-speech`
- Azure SSML reference — `https://learn.microsoft.com/azure/ai-services/speech-service/speech-synthesis-markup`
- Azure Custom Neural Voice overview — `https://learn.microsoft.com/azure/ai-services/speech-service/custom-neural-voice`
- Azure compliance — `https://learn.microsoft.com/azure/compliance/`
- Amazon Polly developer guide — `https://docs.aws.amazon.com/polly/`
- Amazon Polly SSML — `https://docs.aws.amazon.com/polly/latest/dg/ssml.html`
- AWS compliance — `https://aws.amazon.com/compliance/`
- Coqui TTS (XTTS v2 lives in Coqui TTS) — `https://github.com/coqui-ai/TTS`
- Piper — `https://github.com/rhasspy/piper`
- VITS reference — `https://github.com/jaywalnut310/vits`
- Bark — `https://github.com/suno-ai/bark`
- Parler-TTS — `https://github.com/huggingface/parler-tts`

Standards / policy anchors referenced indirectly (used by the compliance row):

- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — frame for the "data residency + auditable logging + consent-of-record" line of the compliance dimension (NIST AI RMF, official standard).
- OWASP Machine Learning Security Top 10 — `https://owasp.org/www-project-machine-learning-security-top-10/` — relevant when the matrix extends into adversarial-input and model-supply-chain concerns (official project).
- MITRE ATLAS — `https://atlas.mitre.org/` — threat-model reference for audio-adversarial and voice-impersonation scenarios (official project).

Source policy note. The 10-dimension list is a local operational convention assembled from the deliverables a voice-agent engineer owns. All vendor-specific TTFB, pricing, and "fastest" claims carry `<!-- verify-at-runtime: confirm at exercise-run time -->` markers because they change frequently; do not cite them downstream without re-verifying against the vendor's own current page.
