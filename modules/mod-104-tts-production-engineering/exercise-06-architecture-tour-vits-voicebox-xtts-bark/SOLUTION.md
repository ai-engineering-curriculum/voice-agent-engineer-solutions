# exercise-06 — Architecture Tour — VITS, VoiceBox, XTTS, Bark — Reference Solution

## 1. Solution overview

This is an architecture-tour exercise. The learning target is **being able to read a TTS paper or model card and explain what the model is doing well enough to pick between it and three alternatives for a production voice agent**. The four architectures in the title cover the design space well:

- **VITS** (2021) — end-to-end, text → waveform, VAE + flow + GAN; the baseline "fast, trainable, open-source" architecture that Coqui / Piper / countless derivatives build on.
- **VoiceBox** (Meta FAIR, 2023) — non-autoregressive flow-matching generative model; large-scale in-context zero-shot TTS and audio-infilling.
- **XTTS v2** (Coqui, 2023) — multilingual, voice-cloning-first zero-shot TTS using a GPT-style decoder + decoder vocoder; the "cloneable, multilingual, open" default of 2024–2025.
- **Bark** (Suno, 2023) — fully generative, text + semantic → coarse → fine audio tokens; expressive, creative, less controllable.

For each, this reference gives (a) a mental model of the architecture, (b) the training-data and inference-shape consequences for production, (c) where each actually fits (and where it doesn't), and (d) the trade-offs that matter to a voice-agent engineer: latency, controllability, voice-cloning fidelity, stability, multilingual coverage, licence posture.

Claims about model behaviour are cited to the papers and model cards; everything else is labelled as the engineering-practitioner reading.

## 2. Implementation

### 2.1 Target properties

A reviewer should see every one of these:

1. **Each architecture summarised in a block diagram**, in words, with the generative modelling family named (VAE, flow, diffusion, autoregressive transformer, flow-matching, GAN vocoder).
2. **Training-data posture** — zero-shot cloning yes/no, how much data to fine-tune, multilingual yes/no.
3. **Inference shape** — autoregressive vs non-autoregressive, chunked vs full-utterance, approximate TTFB class (fast / medium / slow).
4. **Controllability surface** — SSML support, style tags, prosody controls, in-context prompts.
5. **Stability** — how deterministic the output is run-to-run; whether temperature matters.
6. **Licence / provenance** — training data and weight licence posture.
7. **Where it fits** — one line each on which production slot each architecture is a first-choice for.
8. **Trade-off summary table** — side-by-side on the dimensions the mod-104 exercise-01 matrix weighs.

### 2.2 VITS — end-to-end text → waveform (2021)

**Mental model.** VITS is an end-to-end TTS system that trains a single network to go from phoneme sequence to waveform, with no intermediate mel-spectrogram handoff. The three generative-modelling ideas it stacks:

- A **conditional VAE** latent for the acoustic content.
- A **normalising flow** that gives the VAE an invertible mapping from the latent to a rich prior (so the posterior can be expressive without blowing up variance).
- A **GAN vocoder** (HiFi-GAN style) that maps latents to waveform, with a monotonic alignment search handling phoneme-to-audio timing without a separate forced-aligner.

In practice: it is the baseline production-trainable open architecture. You pick a corpus (LJSpeech for English; your own for custom voices), train for days on a single GPU for one voice, get a usable voice at the end.

**Training-data posture.** Single-speaker: hours of clean studio speech + paired text per voice. Multi-speaker variant (VITS2 / YourTTS lineage) adds a speaker embedding. Zero-shot cloning: *not natively*; derivatives (YourTTS, StyleTTS 2) add it.

**Inference shape.** Non-autoregressive at the waveform level (the GAN vocoder is parallel); the flow step is sequential across frames but fast. TTFB class: **fast**. Streaming is achievable with a chunk-level inference wrapper.

**Controllability.** Minimal built-in. Pitch / energy / duration controllability lives in style extensions (StyleTTS 2 adds style vectors, Natural Speech 2 adds diffusion-guided control). Prosody via `<prosody>` / `<break>` requires a text-processing front-end you own.

**Stability.** Deterministic given seed; very stable in practice. A strong choice for regression-sensitive pipelines.

**Licence / provenance.** Reference code MIT; weights depend on the trained-on corpus. LJSpeech weights are permissive. Custom-voice weights inherit your own consent posture (exercise-05).

**Where it fits.**

- A single-voice product with a fixed, known script surface. Piper is the shortest path to deployed VITS on-device; XTTS and friends are heavier for this slot.
- On-device or resource-constrained CPU inference. Piper on a Raspberry Pi is a real thing.
- Base architecture for your own cloned voice where you own the training recording and consent (exercise-05).

**Doesn't fit.**

- Zero-shot cloning of a new voice from a 10-second sample — not the architecture for that.
- Highly expressive, emotional delivery — the architecture can be pushed there, but Bark / XTTS / commercial providers are closer to it out of the box.

### 2.3 VoiceBox — non-autoregressive flow-matching, in-context (2023)

**Mental model.** VoiceBox reframes TTS as *audio infilling*: given a surrounding audio context and a text transcript, generate the missing audio. The generator is a **non-autoregressive flow-matching** model — a continuous-time generative model (closely related to diffusion but trained with the flow-matching objective) that produces mel-spectrograms conditioned on text and prompt audio. A separate vocoder (HiFi-GAN or similar) turns mel into waveform.

The in-context framing is the clever part: because the model is conditioned on *audio context* and *the text of the gap*, it can do zero-shot cloning (give it 3 s of a speaker's voice and the text to speak next), zero-shot style transfer, and audio-editing-in-place from the same training objective.

**Training-data posture.** Trained at large scale (Meta reported >50 k hours; confirm current numbers in the paper and model card). Zero-shot cloning is a first-class capability. Multilingual was a design goal.

**Inference shape.** Non-autoregressive; the whole target spectrogram is generated in parallel ODE/flow steps. Fewer steps than diffusion typically. TTFB class: **medium-to-fast**, depending on step count and vocoder choice.

**Controllability.** The control surface is **audio prompts** more than SSML — you steer by example. Classifier-free guidance scale trades fidelity for diversity; step count trades latency for quality. Prosody via SSML is not the native idiom.

**Stability.** Non-deterministic in general (flow-matching draws from a noise source); stability needs a fixed seed or the classifier-free-guidance scale tuned up. Audible run-to-run variance exists.

**Licence / provenance.** VoiceBox has *not* been released with open weights by Meta (publication only; weights withheld citing misuse risk). This is a first-order fact for production selection: you cannot ship VoiceBox-the-model, only something architecturally similar or a successor. Read the paper and the model card for what Meta ships publicly at review time.

**Where it fits.**

- Research / architecture-tour reference for what "flow-matching TTS" looks like.
- Production slot: *not directly*; practitioners ship successor open-weight flow-matching TTS (F5-TTS, Matcha-TTS, NaturalSpeech family) that follow the same shape. Cite those when you need an actually-shippable architecture with a similar story.

**Doesn't fit.**

- Any slot where you need the specific weights — they aren't public.
- Low-latency streaming with aggressive TTFB budgets — flow-matching step count is a knob that trades step count for latency; practical streaming requires careful tuning or a chunked variant.

### 2.4 XTTS v2 — GPT-style multilingual cloner (Coqui, 2023)

**Mental model.** XTTS v2 is a text-conditioned **autoregressive transformer** (GPT-family decoder) that generates discrete audio tokens from a VQ-VAE codec, plus a **decoder/vocoder** that turns tokens back into waveform. Voice cloning is in-context: you pass a 6–10-second reference clip; its speaker representation conditions the generation. Multilingual at training time; code-switching at inference by setting the target language.

In practice: XTTS v2 is the open-weight "voice-cloning that works" default of 2024–2025. The Coqui org itself ended operations, but the model, code, and community forks remain.

**Training-data posture.** Hours → thousands-of-hours mixed-language corpus. Zero-shot cloning is a first-class capability; 10 s of reference audio is the typical minimum. Fine-tuning for a named voice is also supported (hours of speech, not days).

**Inference shape.** Autoregressive at the token level; the vocoder decode is parallel. Token-by-token generation means TTFB class: **medium**. First tokens come out quickly but the autoregressive serial path caps throughput. Streaming works by emitting audio as tokens complete; this is the Coqui `.inference_stream()` surface.

**Controllability.** Reference-audio-driven; you steer style and prosody by choosing the reference clip. SSML support exists (break, say-as) but is a thinner surface than commercial providers. Temperature controls creativity; too-high values produce audible drift.

**Stability.** Sampling-based; stability varies with temperature. Fine-tuned voices are more stable than zero-shot clones. Audible variance run-to-run at default settings; pin seed + temperature for reproducibility.

**Licence / provenance.** XTTS v2 weights under the Coqui Public Model License (CPML) — permissive for most uses but *not* OSI-open; read the licence before commercial deployment. Training-corpus provenance mixed; consent posture for cloned voices now sits with *you* (exercise-05) because you're providing the reference clip.

**Where it fits.**

- The default open-weight choice when you need multilingual voice cloning and can run a GPU in the serving tier.
- Prototyping a consented-voice product on your own GPU before deciding whether to move to a commercial provider.
- Multilingual products where the commercial provider's language coverage is thin.

**Doesn't fit.**

- Ultra-low-TTFB telephony production without serious serving-stack work. The autoregressive path and the typical container overhead will land you in the 500 ms-plus region unless you invest.
- Deterministic regression testing without seed-pinning discipline.

### 2.5 Bark — generative audio-token TTS (Suno, 2023)

**Mental model.** Bark is a **fully generative audio model** built around three transformer stacks:

1. A **semantic-token transformer** that goes from text (and optional voice preset) to a sequence of semantic tokens (what is being said, at a coarse level).
2. A **coarse-acoustic transformer** that turns semantic tokens into coarse acoustic tokens (EnCodec-style).
3. A **fine-acoustic transformer** that fills in the fine codec tokens; an EnCodec decoder turns tokens into waveform.

Because the pipeline is end-to-end generative over audio tokens, Bark does *everything* — speech, laughter, singing, music, sound effects — all from the same model. That's its appeal and its problem.

**Training-data posture.** Trained on a large mixed audio corpus; no first-party zero-shot cloning (voice presets pick from a known list; cloning a new voice is an add-on project).

**Inference shape.** Autoregressive at the token level, three stacks deep. TTFB class: **slow**. Bark's reputation for "weird, creative, delightful outputs" comes with the reputation for "sometimes it just mumbles for 3 seconds."

**Controllability.** Non-verbal cues (`[laughs]`, `[sighs]`, `[music]`) are in-text; SSML is not the control idiom. Voice preset by name; style is per-preset.

**Stability.** Low. Generations are sampling-based and will drift; same input can produce substantially different outputs on re-run.

**Licence / provenance.** Model code MIT; weights CC-BY-NC-4.0 historically (verify at review time) — non-commercial. First-order constraint for production selection.

**Where it fits.**

- Creative / entertainment / "voice character" use cases where novelty beats stability.
- Research reference for "fully generative audio-token TTS."
- Augmentation of a commercial TTS stack for specific non-verbal cues where the commercial provider's offering is thin.

**Doesn't fit.**

- Production customer-facing voice agents. Low controllability + low stability + non-commercial weights is the wrong combination for a billed product surface.

### 2.6 Side-by-side trade-off summary

The table uses the mod-104 exercise-01 dimension set so you can read this straight into the provider-matrix comparison.

| Dimension | VITS (Piper/Coqui) | VoiceBox-class (FAIR, closed) → open successors (F5-TTS, Matcha-TTS, NaturalSpeech 2) | XTTS v2 | Bark |
|---|---|---|---|---|
| Generative family | VAE + flow + GAN vocoder | Flow-matching (continuous-time) | Autoregressive transformer + codec vocoder | Three-stack autoregressive transformer on audio tokens |
| Native zero-shot cloning | No | Yes | Yes | No (preset-based) |
| Multilingual | Per-trained-voice | Yes | Yes (first-class) | Weak |
| TTFB class | Fast | Medium (step-count dependent) | Medium (autoregressive) | Slow |
| Streaming | Yes, with chunker | Yes, with careful tuning | Yes (first-class) | Impractical in production |
| SSML control | Thin (via front-end) | Thin; prompt-audio-driven | Growing | Non-verbal tags in text |
| Expressivity | Moderate | High (prompt-driven) | Moderate | Very high but unreliable |
| Stability | High (deterministic) | Medium (seed-dependent) | Medium (sampling) | Low |
| Compute footprint | Small (CPU-capable at Piper scale) | Medium–large | Large (GPU recommended for prod) | Large |
| Licence | Code MIT; voice weights per training corpus | Weights withheld (open successors permissive) | CPML — not OSI-open | Code MIT; weights non-commercial historically |
| First-choice slot | Single-voice on-device; custom-voice baseline | Research / near-term open flow-matching TTS | Open-weight multilingual cloning | Creative / entertainment |

### 2.7 How this maps back to mod-104 exercise-01

In the "self-hosted" row of the exercise-01 matrix, the architectures above correspond to four different operating points:

- **Fast / small / stable** → Piper (VITS-family) for single-voice on-device.
- **Multilingual / cloneable / open-weight** → XTTS v2 on your own GPU (and its licence posture).
- **Research-ahead / near-future** → flow-matching successors (F5-TTS, Matcha-TTS, NaturalSpeech 2) as the "what's coming next" column; they will displace the XTTS v2 slot over the next 12–24 months.
- **Creative** → Bark; not a production slot in the exercise-01 sense.

If you collapsed the self-hosted row into "Whisper's cousin" you'd miss this. The honest open-weight self-hosted *answer* depends on which of these four slots you need.

## 3. Validation steps

1. **Each architecture summarised in its own paragraph** with the generative family named (VAE+flow+GAN, flow-matching, autoregressive transformer, token-stack AR).
2. **Each architecture has an inference-shape note** — autoregressive vs non-autoregressive, chunked vs full-utterance, TTFB class.
3. **Each architecture has a licence / provenance note** — including the fact that VoiceBox weights are not public and Bark's weights are historically non-commercial. These first-order facts are what move a production decision.
4. **Trade-off table** aligns to the mod-104 exercise-01 dimension set.
5. **"Where it fits" / "Doesn't fit"** sentences present for each architecture; they name production slots, not vibes.
6. **References.** The paper (or an authoritative substitute) is cited for each architecture; model cards are cited where they exist.
7. **No invented metrics.** No MOS or LAS numbers are fabricated; where quantitative claims would be useful but can't be verified, this reference notes the fact rather than guessing.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Architecture explanation | 20 % | Generative family named; three blocks of the pipeline named for each | Block-diagram-level explanation of the training objective for each |
| Inference shape | 15 % | AR vs NAR, chunked vs full, TTFB class stated | Quantitative TTFB class with the measurement method for the model-you-ran |
| Training-data + licence | 15 % | Zero-shot cloning yes/no stated; weights-licence stated with current posture | Jurisdiction/commercial-use implications called out |
| Trade-off table | 15 % | Table aligned to exercise-01 dimension set; cells populated | Each architecture has a measured cell for at least one dimension |
| Production-slot fit | 10 % | "Where it fits" / "Doesn't fit" sentences; name the slot | Flip-condition for when the slot assignment would change (e.g., flow-matching successors displacing XTTS) |
| References | 10 % | Each architecture cited to the paper or authoritative model card | Also cites a recent (≤ 12 months) survey or follow-up paper per architecture |
| Honesty about unknowns | 10 % | No invented metrics; `<!-- verify-at-runtime: ... -->` used where a claim can't be verified | Verified licence posture at review time with a dated note |
| Mapping to exercise-01 | 5 % | Self-hosted row of exercise-01 matrix explicitly re-read against the four architectures | Shows the "near future" column and what would move the decision |

Scoring < 70 % is a fail-and-redo. The most common sub-70 failure is conflating "what the architecture is" with "what the shipped product does" — e.g., critiquing XTTS v2 for a Coqui-the-company issue without noticing that the model is still broadly usable.

## 5. Common mistakes

- **"It's a transformer."** All four are built around transformer blocks; the useful distinction is the training objective (VAE + flow + GAN vs flow-matching vs AR over codec tokens).
- **Ignoring the licence.** VoiceBox weights are not public; Bark weights have historically been non-commercial; XTTS v2 is under CPML, not OSI-open. These are first-order facts for production selection.
- **Treating zero-shot cloning as the goal.** It is not the goal for every slot. A single-voice on-device product is a *better* fit for Piper than for XTTS v2 — smaller, stabler, cheaper.
- **Comparing to commercial providers on the wrong axis.** Open-weight models rarely beat commercial providers on TTFB at low-latency telephony scale. Compare on the axes where open weights win: cost at high utilisation, cloning freedom, provenance control, on-prem deployability.
- **Hiding the sampling story.** Flow-matching and AR-over-tokens are sampling-based; stability matters for regression testing. Call it out.
- **Citing marketing rather than the paper.** The paper is the source of truth for the architecture; the model card is the source for the licence and known-limitations.
- **Forgetting that VoiceBox isn't a product you can ship.** This architectural tour ships *next to* a production exercise. Be honest that VoiceBox's role in the tour is didactic + "watch its successors."
- **Collapsing "self-hosted" into one row.** Piper, XTTS v2, Bark, and a flow-matching successor are four very different self-hosted answers.
- **Over-generalising from one measurement.** Measured TTFB for XTTS v2 on your GPU is XTTS v2 on *your* GPU. Don't report it as "XTTS TTFB" without the serving-stack context.
- **Skipping the mapping back to exercise-01.** The point of this tour is to make the self-hosted row of the provider matrix defensible. If you don't connect the two, you've done architecture-for-architecture's-sake.

## 6. References

Architecture papers and model cards (verify URLs at exercise time; `<!-- verify-at-runtime -->` markers call out claims that could not be re-verified at write time):

- **VITS** — Kim, Kong, Son (2021), "Conditional Variational Autoencoder with Adversarial Learning for End-to-End Text-to-Speech." arXiv `https://arxiv.org/abs/2106.06103`. Reference code `https://github.com/jaywalnut310/vits`.
- **StyleTTS 2** (VITS-lineage style-conditioned successor) — Li et al. (2023), "StyleTTS 2: Towards Human-Level Text-to-Speech through Style Diffusion and Adversarial Training with Large Speech Language Models." arXiv `https://arxiv.org/abs/2306.07691`. Repo `https://github.com/yl4579/StyleTTS2`.
- **Piper** (shippable VITS-family) — `https://github.com/rhasspy/piper`.
- **VoiceBox** — Le et al. (2023), "Voicebox: Text-Guided Multilingual Universal Speech Generation at Scale." Research page `https://voicebox.metademolab.com/`; paper on arXiv `https://arxiv.org/abs/2306.15687`. Weights not publicly released by Meta at write time. <!-- verify-at-runtime: confirm Meta's current release posture at exercise-run time -->
- **Flow-matching TTS successors** — "F5-TTS" (Chen et al., 2024) `https://arxiv.org/abs/2410.06885`; "Matcha-TTS" (Mehta et al., 2024) `https://arxiv.org/abs/2309.03199`; "NaturalSpeech 2" (Shen et al., 2023) `https://arxiv.org/abs/2304.09116`. These are the open-weight shippable analogues to VoiceBox's architectural shape.
- **XTTS v2** — Coqui model card `https://huggingface.co/coqui/XTTS-v2`; Casanova et al. (2024), "XTTS: a Massively Multilingual Zero-Shot Text-to-Speech Model" `https://arxiv.org/abs/2406.04904`. Repo `https://github.com/coqui-ai/TTS`. Licence: Coqui Public Model License (CPML) — read the licence file in the repo.
- **Bark** — Suno (2023), repo `https://github.com/suno-ai/bark`. Hugging Face model card `https://huggingface.co/suno/bark`. Historical weights licence: CC-BY-NC-4.0 — not commercial-use permissive. <!-- verify-at-runtime: confirm current licence at exercise-run time -->

Related curriculum anchors:

- Provider matrix that this tour feeds the self-hosted row of — `mod-104 exercise-01`
- Streaming-playout implementation that any of these would plug into — `mod-104 exercise-02`
- SSML / lexicon control surface — `mod-104 exercise-03`
- Consent + provenance that attaches to self-hosted cloned voices — `mod-104 exercise-05`
- Safety / compliance layer — `mod-108-safety-compliance-voice`

Standards referenced indirectly:

- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — frames the "weights-licence + training-corpus-provenance as first-order production facts" posture in §2.3–§2.5. *Official NIST standard.*
- OWASP Machine Learning Security Top 10 — `https://owasp.org/www-project-machine-learning-security-top-10/` — ML06 (Supply Chain) is the lens through which "what's in the training corpus" and "how were the weights distributed" become production concerns. *Official OWASP project.*
- MITRE ATLAS — `https://atlas.mitre.org/` — threat-model vocabulary for voice-impersonation / deepfake scenarios that open-weight cloning amplifies. *Official MITRE project.*

Source policy note. Each architecture is cited to its originating paper (arXiv) and to its reference model card where one exists. VeriSwarm implementations are explicitly not used as sources here — the paper and model card are the authoritative references. Where a current licence or release posture could not be re-verified at write time (notably VoiceBox weights-release status and Bark's current licence), a `<!-- verify-at-runtime: ... -->` marker is present rather than an invented citation.
