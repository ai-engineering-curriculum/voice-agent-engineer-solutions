# exercise-04 — Pipeline vs Speech-Native Decision Drill — Reference Solution

## 1. Solution overview

Two shapes compete for the voice-agent slot:

- **Pipeline** — STT → LLM → TTS, three separate models (possibly three vendors) stitched by your code. Examples: LiveKit Agents' `VoicePipelineAgent`, Pipecat, Vocode, and the Claude relay from exercise-03.
- **Speech-native** — one model that accepts audio and emits audio directly over a bidirectional stream. Examples: OpenAI Realtime (ex-01), Gemini Live (ex-02), Moshi (ex-05/ex-06), the GPT-4o "advanced voice" mode behind the Realtime API.

This exercise is a **decision drill**. There is no single correct answer; the correct answer is *a defensible matrix across seven dimensions, two concrete routes, and a one-sentence "what would flip this" per route*. The dimensions are chosen because each one changes the winner on at least some real product.

This solution provides:

- The seven decision dimensions and why each matters for the voice loop.
- A worked matrix across three routes: inbound-contact-centre, voice-first consumer app, and regulated clinical scribe.
- Decision rationale, rubric, and common-mistakes list.

All vendor-specific numbers are either pinned to doc URLs and marked `<!-- verify-at-runtime: ... -->`, or clearly labelled as **illustrative composites**. When you do the exercise, re-pull each cell and run your own small-N measurements — this doc is scaffolding, not scoreboard.

<!-- verify-at-runtime: all model IDs, latency claims, and pricing figures should be re-pulled from each vendor's current docs before quoting externally. -->

## 2. Implementation

### 2.1 The seven dimensions

The seven dimensions below are chosen because each one, on its own, has been enough to flip the pipeline-vs-native decision on a real product. The suggested weights are a *starting point* for a typical English inbound-voice-agent route; you re-weight per route.

| # | Dimension | Why it matters | Suggested weight |
|---|---|---|---|
| 1 | **Voice-to-voice latency** (user stop → first speakable audio) | The user's perception of "liveness." Target < 1.0 s on short answers. | 20 % |
| 2 | **Barge-in & interruption handling** | How gracefully the agent stops when the user cuts in. | 10 % |
| 3 | **Affect / paralinguistics preservation** | Can the model notice the user is angry / whispering / laughing? Can it speak with matching tone? | 10 % |
| 4 | **Controllability & auditability** | Can you pin a written system prompt, log the LLM's reasoning as text, A/B the LLM independently of the voice? | 15 % |
| 5 | **Cost per audio-minute** (fully loaded: STT + LLM + TTS vs native) | Unit economics. Varies by prompt-caching, voice-type, and session length. | 10 % |
| 6 | **Compliance, data residency, provider count** | Fewer vendors is easier to legally certify; some native endpoints lack HIPAA / regional options. | 15 % |
| 7 | **LLM choice freedom** | Does your product need a specific LLM (Claude for safety posture, Llama for self-hosting)? Native endpoints lock you to the host's LLM. | 20 % |

Total 100 %. For a consumer games-companion route you would move 3 up and 6 down; for a clinical scribe you'd move 6 and 4 up and 1 down. The re-weighting is the exercise.

### 2.2 Approach comparison across the seven dimensions

Scoring: `+` meets a reasonable bar; `++` exceeds; `~` partial; `−` below bar. `??` means you have to measure.

| Dimension (weight) | Pipeline (STT→LLM→TTS) | Speech-native (Realtime / Live / Advanced Voice) |
|---|---|---|
| 1. Voice-to-voice latency (20%) | ~ Adds 3 serial first-token latencies. With streaming at every stage + sentence-level pipeline-break + prompt caching, ~ 900–1200 ms voice-to-voice is reachable. **Illustrative composite** — measure for your route. | ++ One model, one stream. < 500–800 ms voice-to-voice on cached short answers is reachable. <!-- verify-at-runtime --> |
| 2. Barge-in (10%) | + You own the cancellation points (STT, LLM, TTS, playout). More code, more control. Can be tuned to "cancel on first user phoneme." | ++ Built-in VAD + server-side truncate (OpenAI) / interrupt flag (Gemini). One API call to handle it. |
| 3. Affect / paralinguistics (10%) | − STT discards affect; TTS adds synthetic prosody. The LLM only sees text. "User was shouting" is invisible to the agent. | ++ Native-audio models (OpenAI Realtime audio-in-audio-out, Gemini native-audio-dialog, Moshi) preserve some paralinguistics in both directions. <!-- verify-at-runtime per model --> |
| 4. Controllability / auditability (15%) | ++ Each stage is text or signal you can inspect, A/B, redact, and version. The system prompt is yours alone. Easy to swap the LLM. | ~ The model's "reasoning" never materialises as text unless you asked for the transcript modality. Harder to A/B the LLM independently of voice. Easier to accidentally leak affect into a transcript. |
| 5. Cost (10%) | ?? Three per-minute streams plus LLM tokens. Prompt caching helps heavy-context agents. Measure on real sessions. | ?? Audio tokens are priced differently from text tokens — typically more expensive per second than a text-only LLM, less expensive than STT+LLM+TTS put together on short sessions, more expensive on long sessions. Measure on real sessions. <!-- verify-at-runtime --> |
| 6. Compliance / residency (15%) | ++ Mix vendors by region: STT and TTS can be regionally hosted independently of the LLM. BYOC / self-hosted combinations are possible on each stage. | ~ You accept the native endpoint's compliance posture whole. Regional availability of Realtime / Live is narrower than the batch APIs. HIPAA-eligibility depends on the specific endpoint / plan. <!-- verify-at-runtime --> |
| 7. LLM choice freedom (20%) | ++ Any LLM via its text API. Claude, Llama-self-hosted, Mistral, Gemini-text all work. | − Locked to the host's LLM. Realtime API means GPT-4o-class; Gemini Live means Gemini-Live-class. If you need a specific LLM for safety / compliance / policy reasons, the native option is off the table. |

Two things a mature reviewer expects to see in this matrix:

1. **Measured cells, not claimed cells.** The dimensions that move most between products (latency, cost, affect) should carry your own numbers. "Faster" copied from a vendor blog is a tell.
2. **"When does the sign flip" cells.** The native column wins latency *on short cached-prompt answers*; that superiority shrinks on long chain-of-thought answers where the LLM generation time dominates. The pipeline column wins LLM-choice freedom *only if you actually want a different LLM*; if you are happy with GPT-4o, that dimension collapses.

### 2.3 Three worked routes

#### Route A — inbound contact-centre voice agent (English, US, HIPAA-adjacent)

Re-weighted:

| Dim | Default | This route | Why |
|---|---|---|---|
| 1 Latency | 20 % | 25 % | Perception of liveness is the whole sale. |
| 2 Barge-in | 10 % | 15 % | Inbound callers interrupt. A lot. |
| 3 Affect | 10 % | 5 % | Nice to have; HIPAA concerns put affect-logging in a weird legal spot anyway. |
| 4 Controllability | 15 % | 15 % | Need to prove exactly what the model was told. |
| 5 Cost | 10 % | 10 % | Matters; not dominant. |
| 6 Compliance | 15 % | 20 % | BAA, SOC2, data residency. Non-negotiable. |
| 7 LLM freedom | 20 % | 10 % | Happy to use the host's LLM if compliance + latency check out. |

**Winner:** speech-native, specifically **OpenAI Realtime API** or **Gemini Live on Vertex AI**, depending on which cloud has the BAA. Reasoning: the top-weighted dimensions (latency, compliance, controllability) all favour the shortest architecture whose compliance posture matches the customer's legal paperwork. Pipeline is the fallback if LLM-freedom becomes a hard constraint.

**What would flip this?** Legal insists on a specific LLM with a published safety posture that is not GPT-4o or Gemini — for example Claude for a customer whose risk team has signed off on Anthropic's Acceptable Use. Then dimension 7 jumps to 25–30 %, and the pipeline with Claude (exercise-03's relay) wins.

#### Route B — voice-first consumer companion app (iOS/Android, English + Spanish)

Re-weighted:

| Dim | Default | This route | Why |
|---|---|---|---|
| 1 Latency | 20 % | 20 % | Perceived liveness matters, but < 800 ms is already good; sub-500 ms no longer wins meaningfully. |
| 2 Barge-in | 10 % | 10 % | Standard. |
| 3 Affect | 10 % | 25 % | A companion app that cannot tell the user is upset is a bad companion. |
| 4 Controllability | 15 % | 10 % | Standard. |
| 5 Cost | 10 % | 15 % | Consumer unit economics. |
| 6 Compliance | 15 % | 5 % | Light. |
| 7 LLM freedom | 20 % | 15 % | Mild. |

**Winner:** speech-native, with a **native-audio** model variant (Gemini native-audio-dialog, OpenAI Realtime audio-in-audio-out on GPT-4o). Reasoning: affect preservation (both in and out) is the step-function the pipeline cannot match today. Pipeline would require STT with paralinguistic annotations + an LLM that treats them as inputs + a TTS with controllable prosody — doable but a lot of engineering for a weaker outcome.

**What would flip this?** The vendor-side voice catalogue does not include the characteristic the product's brand needs. Then you flip to pipeline with a boutique TTS (ElevenLabs voice clone, Cartesia custom voice) — with the cost being loss of affect-in, and the workaround being an affect-detecting STT or a secondary audio-classifier.

#### Route C — regulated clinical scribe (English, ambient listening + structured note generation)

Re-weighted:

| Dim | Default | This route | Why |
|---|---|---|---|
| 1 Latency | 20 % | 5 % | Not a voice loop — a listen-then-write loop. Latency is near-irrelevant. |
| 2 Barge-in | 10 % | 0 % | No agent to barge in on. |
| 3 Affect | 10 % | 0 % | Not a feature. |
| 4 Controllability | 15 % | 30 % | Need to prove exactly what the model saw and wrote. |
| 5 Cost | 10 % | 10 % | Standard. |
| 6 Compliance | 15 % | 30 % | HIPAA. Non-negotiable. |
| 7 LLM freedom | 20 % | 25 % | Clinical guardrails; specific LLM is often mandated. |

**Winner:** **pipeline.** The matrix inverts: latency doesn't matter, controllability and auditability do, LLM choice is often dictated by compliance. Speech-native is overkill and comes with weaker auditability.

**What would flip this?** It would not — this route is a clean pipeline case. The sign of a mature decision is calling out that *some routes should not be a drill at all*; the "consider both" framing is overused.

### 2.4 Decision tree (quick form)

```
Is the LLM pinned by compliance / policy / self-host mandate?
    yes → Pipeline. Stop. (Route C and the flip-case of Route A)
    no  → continue.

Is affect-in or affect-out a product requirement?
    yes → Speech-native with a native-audio model. (Route B)
    no  → continue.

Is voice-to-voice latency < 500 ms a measurable product requirement?
    yes → Speech-native. (Route A default)
    no  → continue.

Is the product a listen-only or batch-shape flow (scribe, meeting notes)?
    yes → Pipeline, batch mode. (Route C variant)
    no  → Default to speech-native, review cost on real traffic.
```

The tree is a *starting* heuristic. Dimensions 5 (cost) and 6 (compliance) are tiebreakers that can override the top of the tree for specific customers.

### 2.5 Hybrid patterns

Not every product is one or the other. Patterns worth naming:

- **Native front-end + pipeline fallback.** Use the Realtime API for the hot path; when it rate-limits or errors, fall back to a Claude relay. Both must share the same conversation history.
- **Pipeline + native "voice-mode" toggle.** A text-first product with a voice button that opens a Realtime session scoped to one turn. Avoids running a voice stack for text-only users.
- **Speech-native for understanding, pipeline for speaking.** Use a speech-native model as a *listener* (audio-in → text-out, including affect annotations), then a conventional TTS to speak. Useful when you need a specific TTS voice but want affect-in. Not currently a mainstream product shape; worth knowing.
- **Pipeline with a native-audio STT variant.** `gpt-4o-transcribe` et al. transcribe audio *and* capture some paralinguistic info compared to vanilla ASR. Lifts dimension 3 on a pipeline without going fully native.

## 3. Validation steps

A finished submission should pass each check.

1. **Both approaches named precisely.** The submission uses "pipeline" and "speech-native" (or synonyms with a definition) and does not conflate the two.
2. **Dimensions defined.** At least 6 dimensions with weights that sum to 100 %. Each weight is justified rather than copied.
3. **Three routes (or two with distinct re-weighting).** Each route has its own weight column, chosen winner, and a one-sentence "what would flip this" tied to a measurable.
4. **Measured cells where it matters.** The top-weighted dimension for each route has at least one measurement (yours or a cited vendor number with a `<!-- verify-at-runtime: ... -->` marker) — not a vibe.
5. **LLM-freedom dimension is weighted non-zero somewhere.** A matrix that gives every route an LLM-agnostic score has not understood that native endpoints lock you to the host's LLM.
6. **Affect dimension is weighted non-zero somewhere.** A matrix that treats affect as free in all cases misunderstands the pipeline's main weakness.
7. **Hybrid pattern named.** The submission mentions at least one hybrid pattern (§2.5 or an equivalent) and says when it applies.
8. **No "pipeline is always worse" / "native is always worse" conclusion.** Either absolutist stance indicates the drill wasn't run.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Dimension list + weights | 15 % | ≥ 6 dimensions, sum to 100 %, each justified | All 7 with route-specific re-weighting shown |
| Approach comparison | 20 % | Comparison table across both approaches on every dimension | Measurements (yours or cited) for latency, cost, affect on the chosen route |
| Route treatment | 25 % | 3 distinct routes with re-weighting and a chosen winner | Each route has a quantified flip condition and a plan B |
| Decision tree / heuristic | 10 % | Decision tree or equivalent walkthrough that reaches a terminal in finite steps | Decision tree names the tiebreakers (cost, compliance) and where they override the top |
| Hybrid awareness | 10 % | ≥ 1 hybrid pattern described with a concrete use case | ≥ 2 hybrid patterns described; failure modes of each noted |
| LLM-freedom awareness | 10 % | Non-zero weight on dimension 7 in at least one route | Specific compliance / safety-posture reasons for LLM pinning named |
| Affect awareness | 10 % | Non-zero weight on dimension 3 in at least one route | Native-audio vs half-cascade distinction correctly named |

Scoring < 70 % is a fail-and-redo. Common sub-70 failures: (a) a single-route matrix (not a decision — a pitch deck), (b) absolutist conclusions, and (c) latency as the only weighted dimension.

## 5. Common mistakes

- **Treating latency as the only dimension.** Latency wins get oversold. A model that is 200 ms faster but the wrong LLM, the wrong compliance posture, or the wrong voice characteristic is still the wrong model.
- **Comparing "cold" pipeline against "warm" native.** Both approaches have warm-up effects. A fair comparison warms both — prompt-cached pipeline vs session-warmed native — and measures at steady state.
- **Ignoring tool-call tax on the pipeline.** A tool call in a pipeline inserts a *second* LLM call plus a TTS ramp-up. A tool call on a native endpoint inserts a `toolCall` round-trip. Both are user-visible; the measurement should include at least one tool-using turn.
- **Pretending cost is a solved question.** Audio-token pricing changes frequently. Any cost comparison without a date and a `<!-- verify-at-runtime: ... -->` is stale by publication.
- **Assuming native endpoints are "end-to-end" the whole way.** Many native endpoints are half-cascade internally — STT → LLM → TTS behind one WebSocket. They lose some of the affect advantage but keep the latency advantage. Native-audio variants are the ones that do single-model audio I/O; name the variant.
- **Not naming an LLM-freedom scenario.** Half of all real voice-agent decisions are driven by the business wanting a specific LLM for compliance, safety, or cost reasons. A matrix that ignores that is the wrong matrix.
- **Hybrid as a cop-out.** Picking "hybrid" without naming when each branch runs and how state is shared is the same as not deciding. Treat hybrid as its own designed shape, with an explicit split criterion.
- **Latency measurements on the first call of the morning.** Cold starts bias the number. Measure after warm-up, in the busy-hour window, and report P50 / P95, not mean.

## 6. References

Primary — provider documentation (verify URLs still resolve at exercise-run time):

- OpenAI Realtime API — `https://platform.openai.com/docs/guides/realtime`
- Google Gemini Live API — `https://ai.google.dev/gemini-api/docs/live`
- Google Vertex AI Live API — `https://cloud.google.com/vertex-ai/generative-ai/docs/live-api`
- Anthropic Messages API — `https://docs.anthropic.com/en/api/messages`
- Anthropic prompt caching — `https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching`
- LiveKit Agents — `https://docs.livekit.io/agents/`
- Pipecat framework — `https://docs.pipecat.ai/`
- Deepgram docs — `https://developers.deepgram.com/docs`
- AssemblyAI docs — `https://www.assemblyai.com/docs`
- Cartesia TTS — `https://docs.cartesia.ai/`
- ElevenLabs — `https://elevenlabs.io/docs`
- OpenAI TTS — `https://platform.openai.com/docs/guides/text-to-speech`

Research / architecture notes (open-weights native-audio reading):

- Kyutai Moshi — `https://kyutai.org/moshi.pdf` (full-duplex speech-native model; context for ex-05/ex-06)
- Meta SeamlessM4T — `https://ai.meta.com/research/seamless-communication/`
- Qwen2-Audio — `https://github.com/QwenLM/Qwen2-Audio`

Related exercises inside this module:

- exercise-01 (OpenAI Realtime lifecycle) — the "native" side of this drill.
- exercise-02 (Gemini Live lifecycle) — same, Google variant.
- exercise-03 (Claude via relay) — the "pipeline" side of this drill.
- exercise-05 (half vs full duplex) — explains why *neither* current widely-deployed speech-native is truly full-duplex, which tempers the "native always wins affect" claim.

Standards referenced indirectly:

- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — the "measure, govern, document" discipline behind the measurement expectations.
- OWASP Machine Learning Security Top 10 — `https://owasp.org/www-project-machine-learning-security-top-10/` — relevant when weighting the controllability / auditability dimension against adversarial-audio and prompt-injection risks.
- MITRE ATLAS — `https://atlas.mitre.org/` — threat-model reference for the compliance / auditability framing.

Source policy note. No vendor-supplied numeric claim in this document is portable to a different product without re-measurement. All quantitative cells carry `<!-- verify-at-runtime: ... -->` markers; the matrix shape is the stable artifact, the cells are not.
