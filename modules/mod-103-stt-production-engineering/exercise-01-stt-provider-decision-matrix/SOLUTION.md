# exercise-01 — STT Provider Decision Matrix — Reference Solution

## 1. Solution overview

This is a decision / selection exercise. There is no single correct answer — the correct answer is **a defensible matrix, built from docs and small reproducers you actually ran**, that commits to one shortlisted provider per route and names what would flip the decision. This solution shows:

- The dimensions a reviewer should expect to see weighted.
- A worked matrix across six candidate providers and two contrasting routes (inbound-contact-centre and outbound-reminder).
- Decision rationale with "what would flip this" sentences for the two closest runners-up.
- A rubric and common-mistake list.

Because vendor pricing, model names, feature flags, and SLAs move every quarter, all vendor-specific numbers in this reference are either:

1. drawn from the vendor's public documentation page at the time this reference was written, with a URL, and marked with a `<!-- verify-at-runtime: confirm at exercise-run time -->` comment where they are the kind of number that changes often (price, "fastest" claim, model-version headline WER), **or**
2. clearly labelled as **illustrative composite** when used as a measurement example (e.g., `~15 % WER`).

When you do the exercise, **re-pull every cell from the current vendor documentation** and run a small-N measurement (10–50 utterances) rather than trusting any secondary source, this one included.

## 2. Implementation

### 2.1 Candidate list (what gets into the matrix)

The six providers below are the ones the voice-agent curriculum points at as the realistic shortlist for English-centric production voice agents in 2025–2026. Not every project needs all six — a mature pick is usually two kept warm and one in production:

- **Deepgram** — streaming-first commercial ASR; Nova family models. (`https://developers.deepgram.com/docs`)
- **AssemblyAI** — Universal / Universal-Streaming model family; strong on English accuracy and audio-intelligence add-ons. (`https://www.assemblyai.com/docs`)
- **Google Cloud Speech-to-Text** — Chirp family + legacy models; strong multilingual coverage. (`https://cloud.google.com/speech-to-text/docs`)
- **Microsoft Azure Speech (Speech to Text)** — unified Speech service; Azure region alignment for compliance-anchored customers. (`https://learn.microsoft.com/azure/ai-services/speech-service/`)
- **OpenAI** — `gpt-4o-transcribe` / `gpt-4o-mini-transcribe` (batch) and Realtime-API ASR path (streaming). (`https://platform.openai.com/docs/guides/speech-to-text`, `https://platform.openai.com/docs/guides/realtime`)
- **Self-hosted Whisper / whisper.cpp / faster-whisper / NVIDIA Parakeet / Canary** — open-weight batch (and semi-streaming via chunking). (`https://github.com/openai/whisper`, `https://github.com/ggerganov/whisper.cpp`, `https://github.com/SYSTRAN/faster-whisper`, `https://developer.nvidia.com/riva`)

Providers deliberately omitted and why:

- **Amazon Transcribe** — can and should be added for AWS-anchored customers; omitted here to keep the worked matrix at six. (`https://docs.aws.amazon.com/transcribe/`)
- **Rev AI, Speechmatics, iFlytek, Soniox** — valid but narrower fit; the matrix shape below generalises.
- **Nvidia Riva as a managed SaaS** — treated under self-hosted / managed-open-weights row rather than as a separate commercial-API column.

### 2.2 Dimensions (what gets weighted)

The dimensions below come straight from the deliverables a voice-agent engineer is on the hook for in production. Weights are a *suggested* starting point for a typical English inbound-contact-centre route; you re-weight for your route.

| # | Dimension | Why it matters for the voice loop | Suggested weight (inbound CX) |
|---|---|---|---|
| 1 | Streaming TTFT (time to first *partial*) | Drives perceived responsiveness. Budget rule of thumb: ASR partial under 300 ms. | 15 % |
| 2 | Streaming finalisation latency (end-of-utterance → final) | Feeds the "start the LLM" trigger; interacts with VAD endpointing. | 10 % |
| 3 | WER on your in-domain gold set | Translation of accuracy into downstream LLM quality. See exercise-05. | 20 % |
| 4 | Robustness (noise, codec degradation, accents) | Real call audio is 8 kHz µ-law or Opus, not studio. | 10 % |
| 5 | Customisation (keyterms, custom vocab, custom models) | Entities and jargon are where generic ASR fails. See exercise-04. | 10 % |
| 6 | Output richness (word timestamps, confidence, diarisation, speaker-change events, PII redaction) | Every missing field becomes code you write. See exercise-03. | 5 % |
| 7 | Language / dialect coverage | Determines whether you ladder (exercise-06) or route per-language. | 5 % |
| 8 | Price per audio-minute (and streaming vs batch tier split) | Unit economics; usually 2× between tiers. | 10 % |
| 9 | Data residency, compliance posture, BYOK, on-prem option | HIPAA, GDPR, PCI, FedRAMP constraints. | 5 % |
| 10 | Operational maturity (SLA, status page, incident history, support responsiveness) | What you depend on when you are paged at 03:00. | 5 % |
| 11 | SDK / integration surface (WebSocket, gRPC, native SDKs, LiveKit / Pipecat plugin) | Reduces time-to-first-call and time-to-recovery. | 5 % |

Total 100 %. Re-weight — don't keep the suggestion. For a clinical scribe route you'd move dimension 9 up and 1 down; for a low-latency games-companion route the opposite.

### 2.3 Worked matrix — inbound contact-centre route (illustrative)

Scoring key: `+` meets bar, `++` exceeds, `~` partial, `−` below bar, `??` requires measurement at exercise time. Cells labelled `??` are *the point*: a mature reviewer expects most high-weight cells to be filled from your own small-N measurements rather than copied from vendor claims.

| Dimension (weight) | Deepgram Nova | AssemblyAI Universal-Streaming | Google Chirp | Azure Speech | OpenAI (Realtime / gpt-4o-transcribe) | Self-hosted Whisper / Parakeet |
|---|---|---|---|---|---|---|
| 1. Streaming TTFT (15%) | ++ Streaming is a first-class product; sub-300 ms partial latency is advertised as a design point. <!-- verify-at-runtime: confirm current latency claims --> | ++ Universal-Streaming is positioned as a low-latency streaming path. <!-- verify-at-runtime --> | + Streaming supported; latency typically dependent on region and Chirp vs legacy model. <!-- verify-at-runtime --> | + Streaming supported via Speech SDK; latency varies by region tier. <!-- verify-at-runtime --> | + Realtime API delivers streaming partials; TTFT depends on connection warm-up. <!-- verify-at-runtime --> | ~ Whisper is batch-architecturally; "streaming" is chunked-batch (300–1000 ms chunks) and behaves worse than true streaming models. Parakeet/Riva streaming is better. |
| 2. Finalisation latency (10%) | + Endpointing is a provider-level event. | + Endpointing is first-class. | + Available; configurable. | + Available. | ~ Finalisation semantics differ by endpoint; need to measure. | ~ Depends on your chunker. |
| 3. In-domain WER (20%) | ?? | ?? | ?? | ?? | ?? | ?? |
| 4. Robustness (10%) | ++ Trained on telephony data historically. | + Strong on clean broadband; measure on 8 kHz µ-law. | + Varies by model (Chirp vs telephony-tuned legacy). | + Telephony models available. | ?? Realtime path is relatively new on telephony audio; measure. | ~ Whisper is trained on broadband; 8 kHz performance is a known weak spot. |
| 5. Customisation (10%) | ++ `keywords` / keyterm prompting + custom model training. (`https://developers.deepgram.com/docs/keywords`) | ++ Word boost / custom vocabulary. (`https://www.assemblyai.com/docs/speech-to-text/pre-recorded-audio/customize-vocabulary`) | ++ Speech adaptation / PhraseSets / class tokens. (`https://cloud.google.com/speech-to-text/docs/adaptation-model`) | ++ Custom Speech (custom pronunciations + custom models). (`https://learn.microsoft.com/azure/ai-services/speech-service/how-to-custom-speech-overview`) | ~ Hint-string style via prompt; richer custom-vocab story is narrower than dedicated providers. | ++ Full control — initial prompt, custom LM / custom decoder possible. |
| 6. Output richness (5%) | ++ Word-level timestamps, confidence, diarisation, redaction. | ++ Word timestamps, confidence, diarisation, audio-intelligence. | + Word timestamps, confidence, diarisation. | + Word timestamps, confidence, diarisation. | ~ Shape and granularity depends on endpoint; confirm per endpoint. | Depends on model; Whisper gives segment-level timestamps, Parakeet/Riva give word-level. |
| 7. Language / dialect (5%) | + 30+ languages advertised; English has the deepest tuning. <!-- verify-at-runtime --> | + Growing list; English remains strongest. <!-- verify-at-runtime --> | ++ Google's multilingual coverage is a historical strength. <!-- verify-at-runtime --> | ++ Azure likewise. <!-- verify-at-runtime --> | + Multilingual (Whisper lineage). <!-- verify-at-runtime --> | ++ Whisper multilingual; Parakeet English-centric. |
| 8. Price per minute (10%) | ?? | ?? | ?? | ?? | ?? | GPU-hour cost; break-even depends on utilisation. See §2.4. |
| 9. Compliance (5%) | HIPAA / SOC 2 available on applicable plans. (`https://deepgram.com/trust`) | HIPAA / SOC 2 on applicable plans. (`https://www.assemblyai.com/security`) | FedRAMP / HIPAA via Google Cloud compliance programs. (`https://cloud.google.com/security/compliance`) | FedRAMP High / HIPAA / etc via Azure compliance programs. (`https://learn.microsoft.com/azure/compliance/`) | SOC 2, HIPAA under BAA on applicable plans. (`https://trust.openai.com`) | Fully in your VPC / on-prem — strongest story when the customer requires it. |
| 10. Ops maturity (5%) | Public status page. (`https://status.deepgram.com/`) | Public status page. (`https://status.assemblyai.com/`) | GCP status. (`https://status.cloud.google.com/`) | Azure status. (`https://status.azure.com/`) | OpenAI status. (`https://status.openai.com/`) | You *are* the status page. |
| 11. SDK surface (5%) | WebSocket + SDKs (Node, Python, Go, Rust). First-class LiveKit / Pipecat plugins. | WebSocket + SDKs. Pipecat / LiveKit plugins. | gRPC + REST + SDKs. | WebSocket + SDKs. | WebSocket Realtime; REST for batch. | Your SDK story is your own. |

Where cells say `??`, the exercise expects you to measure (see exercise-05 for the WER harness and exercise-02 for streaming latency). A matrix with all cells pre-filled from marketing pages is a *bad* matrix — it's a vendor pitch deck, not a decision.

### 2.4 Decision rationale — the two routes

**Route A — inbound contact-centre voice agent (English, US, HIPAA-adjacent).**

*Winner:* Deepgram Nova as production, AssemblyAI as the warm backup, self-hosted Whisper off the table for latency reasons on the hot path.

Reasoning tied to weights:

- Dimension 1 (streaming TTFT, 15 %) and Dimension 4 (robustness on telephony audio, 10 %) are the two cells that matter most for the "does the bot feel responsive on a real call" judgement. Deepgram's streaming-first product posture and telephony-audio training heritage make it the default; AssemblyAI is in the same neighbourhood.
- Dimension 5 (customisation, 10 %) is near-tie between Deepgram and AssemblyAI; both support keyterm boosting (exercise-04). The decision isn't made here.
- Dimension 8 (price, 10 %) and Dimension 10 (ops maturity, 5 %) are near-tie between the top two commercial providers; measure both on *your* traffic shape, not the vendor's price calculator.
- Dimension 9 (compliance, 5 %) — both have HIPAA-eligible plans; if the customer's legal team has already signed a BAA with Google or Azure, your "winner" moves to whichever cloud they're on.
- Self-hosted Whisper is a *bad* fit for this route specifically: its streaming story is chunked batch, and 8 kHz µ-law robustness is a known weak spot. Parakeet / Riva would be a better open-weight option for this route, not Whisper.

**What would flip this?**

- *Contract term flip.* If the customer insists on BYOC / on-prem, self-hosted (Parakeet or Whisper-large-v3) becomes the only option and the dimensions re-rank.
- *WER flip.* If your in-domain gold-set WER (exercise-05) shows a 3+ absolute-point gap in AssemblyAI's favour on your actual voicemails, flip. Three points is a reasonable minimum detectable difference at the scale most teams can gather in a week; smaller gaps risk being within measurement noise.
- *Cost flip.* If you cross the utilisation threshold (see §2.5) above which self-hosted amortises, revisit — but amortise against *your own GPU utilisation*, not vendor-quoted hour rates.

**Route B — outbound reminder voice agent (English + Spanish, lower-concurrency, quality-sensitive).**

*Winner:* AssemblyAI (or Deepgram) streaming for the English route; Google Chirp or Azure Speech for the Spanish route because their multilingual coverage is historically stronger and dialect coverage matters more when the call volume is lower per language.

Reasoning: when per-language volume is low, you cannot amortise a custom-trained model, so you rely on provider-side multilingual coverage.

**What would flip this?**

- A mandate to run a single provider for operational simplicity would re-weight dimension 11 (SDK surface, 5 %) and dimension 10 (ops maturity, 5 %) upward — and push toward Google / Azure as the one-provider answer.
- If outbound-reminder calls are short enough that per-second billing dominates, pricing (10 %) jumps in weight and self-hosted may become viable *because the per-call cost of self-hosting a single-language model is predictable*.

### 2.5 Break-even sketch (self-hosted vs commercial)

This is the back-of-envelope every voice-agent engineer should carry mentally; the numbers below are the *structure*, not the current answer — plug in real quotes at exercise-run time.

```
monthly commercial cost ≈ (minutes_of_audio_per_month) × ($ / minute_commercial_tier)
monthly self-hosted cost ≈ GPU_cost_per_hour × hours_billed_per_month
                           + storage + bandwidth + on-call + model-retraining amortised
break-even minutes ≈ (GPU_cost_per_hour × hours_billed + ops_overhead) / ($/minute_commercial)
```

The failure mode is to count only the GPU line item. Ops overhead (on-call, retraining, model-release testing, dependency upgrades) is typically 20–50 % of the GPU line and is the thing that bites after month six. Call this out in the matrix explicitly — a bare "self-hosted is cheaper" line gets challenged in review.

## 3. Validation steps

A finished matrix should pass each of the following checks.

1. **Dimension completeness.** All 11 dimensions are present and weighted. Weights sum to 100 %. Any weight ≤ 5 % is justified (otherwise why include it).
2. **Provider spread.** At least four commercial providers plus at least one self-hosted option. A matrix that omits either side is a matrix that can't surface the right trade-off.
3. **Measured cells.** For the three highest-weight dimensions — streaming TTFT, in-domain WER, and robustness — you have your *own* measurement for at least two providers. No `??` in the top three dimensions for the shortlisted providers.
4. **"What would flip this" sentences.** Two runner-up providers each have a one-sentence flip condition, each referencing a specific measurable, not a vibe.
5. **Break-even sketch.** The self-hosted-vs-commercial line is quantified, even if roughly, and names the ops-overhead line item explicitly.
6. **Freshness check.** Every vendor-sourced cell has a documentation URL and a note of the date it was pulled. Pricing and "fastest" claims carry a `<!-- verify-at-runtime: confirm at exercise-run time -->` marker so a reviewer knows not to trust them past their freshness.
7. **Compliance row isn't a checkbox.** If HIPAA / PCI / FedRAMP matters, the row names the specific control family at risk (data residency, encryption-at-rest, BYOK) rather than just "yes."

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Dimension list + weights | 15 % | ≥ 8 dimensions, sum to 100 %, each weight justified | All 11 dimensions, with route-specific re-weighting shown for at least two routes |
| Provider shortlist | 10 % | ≥ 4 commercial + 1 self-hosted, each with a doc URL | Explicit "excluded and why" list, not just "included and why" |
| Cell freshness | 15 % | Every vendor cell has a URL + pull date | Pricing / "fastest" cells marked with freshness caveats |
| Measurement | 20 % | Own measurement for ≥ 2 providers on ≥ 2 high-weight dimensions | Measurements posted with sample size and confidence interval, not just a point value |
| Decision + flip conditions | 20 % | One winner per route, each with a "what would flip this" sentence tied to a measurable | Flip conditions use minimum-detectable-effect reasoning, not "if the vendor is faster" |
| Break-even sketch | 10 % | Self-hosted-vs-commercial quantified at least roughly | Ops-overhead line item itemised (on-call, retraining, upgrade testing) |
| Compliance detail | 10 % | Where compliance matters, named control family + source | Failure mode of each posture called out (e.g., "BYOK but key-management burden on us") |

Scoring < 70 % is a fail-and-redo; the most common sub-70 failure is a matrix with marketing-page cells and no measurement.

## 5. Common mistakes

- **Scoring from vendor pages.** The marketing claim and your measurement differ, often by a lot. The entire point of the matrix is to produce *your* measurements for *your* traffic shape.
- **One-provider matrix.** If the matrix evaluates one provider, it's a pitch deck, not a decision document. A decision implies comparison.
- **Streaming TTFT measured on the first call of the morning.** Cold-start artefacts will bias the number. Measure after warm-up, in the busy-hour window your product actually serves, and report a P-value (P50 / P95 / P99), not a mean.
- **Treating "streaming" and "batch" as a single column.** For OpenAI, Deepgram, AssemblyAI, Google and Azure, streaming and batch are different endpoints with different latency/accuracy/price profiles. Build the matrix per *endpoint*, not per vendor. See exercise-02.
- **Ignoring the ops row.** A provider with great accuracy and a flaky status page is a provider that will page you at 03:00. The ops maturity row is doing real work — if you weight it at 0 %, you are saying you're okay being paged at 03:00.
- **Self-hosted == cheaper, by default.** Only true after you cross a utilisation threshold. The break-even sketch exists to force you to put a number on this.
- **Compliance = "yes."** Compliance is never a scalar — it's a mapping from legal-approved control families to vendor evidence. A row that says "HIPAA: yes" is a row that fails on first legal review.
- **No self-measurement on multilingual / dialect.** Vendor-claimed language coverage is advertised; vendor-measured WER on *your* dialect is not. See exercise-06 for the ladder you should actually run.

## 6. References

Primary — provider documentation (verify URLs still resolve at exercise-run time; vendor doc trees are frequently reorganised):

- Deepgram developer docs — `https://developers.deepgram.com/docs`
- Deepgram keyterm / keywords feature — `https://developers.deepgram.com/docs/keywords`
- Deepgram status page — `https://status.deepgram.com/`
- Deepgram trust / compliance — `https://deepgram.com/trust`
- AssemblyAI docs — `https://www.assemblyai.com/docs`
- AssemblyAI custom vocabulary — `https://www.assemblyai.com/docs/speech-to-text/pre-recorded-audio/customize-vocabulary`
- AssemblyAI security / compliance — `https://www.assemblyai.com/security`
- AssemblyAI status page — `https://status.assemblyai.com/`
- Google Cloud Speech-to-Text docs — `https://cloud.google.com/speech-to-text/docs`
- Google Cloud Speech-to-Text adaptation — `https://cloud.google.com/speech-to-text/docs/adaptation-model`
- Google Cloud compliance — `https://cloud.google.com/security/compliance`
- Microsoft Azure Speech Service docs — `https://learn.microsoft.com/azure/ai-services/speech-service/`
- Azure Custom Speech overview — `https://learn.microsoft.com/azure/ai-services/speech-service/how-to-custom-speech-overview`
- Azure compliance — `https://learn.microsoft.com/azure/compliance/`
- OpenAI speech-to-text — `https://platform.openai.com/docs/guides/speech-to-text`
- OpenAI Realtime API — `https://platform.openai.com/docs/guides/realtime`
- OpenAI trust portal — `https://trust.openai.com`
- Amazon Transcribe docs — `https://docs.aws.amazon.com/transcribe/`
- OpenAI Whisper — `https://github.com/openai/whisper`
- whisper.cpp — `https://github.com/ggerganov/whisper.cpp`
- faster-whisper — `https://github.com/SYSTRAN/faster-whisper`
- NVIDIA Riva / Parakeet — `https://developer.nvidia.com/riva`

Standards / policy anchors referenced indirectly (used by the compliance row):

- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — frame for the "data residency + auditable logging" line of the compliance dimension.
- OWASP Machine Learning Security Top 10 — `https://owasp.org/www-project-machine-learning-security-top-10/` — relevant when the matrix extends into adversarial-input and model-supply-chain concerns.
- MITRE ATLAS — `https://atlas.mitre.org/` — threat-model reference for audio-adversarial scenarios (overlaps with mod-108).

Source policy note. The 11-dimension list is a local-to-this-curriculum operational convention assembled from the deliverables a voice-agent engineer owns; no vendor-ranking produced by this reference is portable to a different customer. All vendor-specific latency, pricing, and "fastest" claims in this document carry `<!-- verify-at-runtime: confirm at exercise-run time -->` markers because they change frequently; do not cite them downstream without re-verifying against the vendor's own current page.
