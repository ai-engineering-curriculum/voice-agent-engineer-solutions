# exercise-05 — Half vs Full Duplex Architecture Read: Moshi and Advanced Voice — Reference Solution

## 1. Solution overview

"Realtime" and "full-duplex" are not the same thing. The major commercial speech-native systems you have seen in exercises 1–3 are **half-duplex with low-latency interruption**: the user and the assistant share a single logical channel; a VAD or interruption signal switches who holds it; the two parties do not simultaneously *generate* speech. **Full-duplex** is a stronger property: both sides can speak and listen at the same time, both model streams run in parallel, and the assistant can produce backchannels ("mm-hm", "right", "I see") *while* the user is still talking.

This exercise is an architecture-reading drill. You read two system descriptions — Kyutai's **Moshi** (open-weights, published full-duplex) and OpenAI's **Advanced Voice Mode** (production, surface exposed via the Realtime API) — and produce:

- A precise half-duplex vs full-duplex definition that distinguishes them.
- A system-diagram-level description of each, including token streams, VAD, inner monologue (for Moshi), and the speech-native backbone (for Advanced Voice).
- A head-to-head table on the properties that matter in production (latency, barge-in, backchannels, controllability, deployability).
- A rubric and a common-mistakes list.

All claims about Moshi are drawn from the Kyutai paper (`https://kyutai.org/moshi.pdf`) and the Moshi repository (`https://github.com/kyutai-labs/moshi`). Claims about Advanced Voice Mode come from OpenAI's public documentation and the Realtime API docs; where OpenAI has not disclosed internal architecture, the solution says so explicitly rather than guessing.

<!-- verify-at-runtime: all paper-sourced numbers (parameter count, frame rate, latency, token-stream count for Moshi) should be re-checked against kyutai.org/moshi.pdf and the Moshi GitHub README. OpenAI internal-architecture claims should carry source-unverified markers unless OpenAI has disclosed them. -->

## 2. Implementation

### 2.1 Duplex, precisely

| Property | Half-duplex (turn-taking) | Full-duplex |
|---|---|---|
| Who may speak at a given instant | One at a time; a VAD/interrupt-event decides. | Either or both; no exclusivity. |
| Model's speech-generation stream when the user is talking | Paused / cancelled on barge-in. | Continues running; may emit short backchannels or stay silent. |
| Backchannels ("mm-hm", "right") | Only if simulated by scripting short TTS utterances between user turns. | Natural; the model emits them from its own generative stream. |
| Reaction time to user | Bounded by VAD → cancel → next-turn cycle (typically 200–600 ms). | Bounded by frame rate (12.5 Hz in Moshi ⇒ 80 ms per step), not by a turn event. |
| Typical architecture | Two bidirectional audio paths (mic + speaker), one assistant generation stream, event-driven turn boundary. | At least two **parallel generation streams** (one per speaker), both conditioned on a shared past. |

Barge-in in a half-duplex system is a *special case* of switching turns fast. In a full-duplex system there are no turns to switch; the model is continuously predicting what to emit on its channel conditioned on what the other channel has emitted so far.

### 2.2 Moshi — architecture read

Moshi is Kyutai's open-weights full-duplex speech-native model, published 2024. Key design choices visible in the paper:

- **Mimi**: a neural audio codec. Encodes 24 kHz mono audio into **discrete tokens at 12.5 Hz** with **8 codebooks per frame** (RVQ — residual vector quantisation). Mimi separates a "semantic" codebook (used to predict content) from "acoustic" codebooks (used to reconstruct the waveform). 12.5 Hz ⇒ one Mimi frame is 80 ms of audio. <!-- verify-at-runtime: Mimi frame rate and codebook count from the Moshi paper, section on Mimi. -->
- **Two token streams**: Moshi predicts, at every frame, the next tokens for *two* speakers simultaneously — the "user" stream and the "assistant" stream — plus a **text transcript stream** for the assistant ("inner monologue"). Each stream is a sequence of Mimi codebooks over time.
- **Inner monologue**: the text stream is generated **ahead of** the assistant audio stream. The model writes a word first, then produces the Mimi tokens that pronounce it. The paper argues this improves both coherence and pronunciation accuracy and lets you read the model's "thought" as text while it speaks.
- **Backbone**: an autoregressive Transformer (7B parameters, "Helium" backbone). Decodes multiple streams in a single forward pass via the "RQ-Transformer" multi-stream pattern.
- **Full-duplex behaviour**: the model is trained on two-stream dialog data, so it learns what to put on its own stream (mostly silence tokens when the user is speaking, with occasional backchannels) conditioned on the other stream. There is no VAD and no explicit "turn" boundary.
- **End-to-end latency**: ~ 200 ms theoretical floor from one frame of input audio to the next frame of output audio on a modern GPU, per the paper. <!-- verify-at-runtime -->
- **Deployment**: open weights + inference code on GitHub; runs locally on a GPU (CUDA and Candle variants); also available via `moshi-server`. <!-- verify-at-runtime: current repo state -->

Block diagram, approximately:

```
      Mic audio (24 kHz)
             │
             ▼
         [ Mimi encoder ]  ── 8 codebook streams @ 12.5 Hz ──┐
                                                             │
    (previous frames of both streams + text stream) ─────► [ Helium 7B RQ-Transformer ]
                                                             │
                                                             ├──► text token (inner monologue, lead-time)
                                                             ├──► user-stream tokens (next frame)
                                                             └──► assistant-stream tokens (next frame)
                                                                                           │
                                                                                           ▼
                                                                                     [ Mimi decoder ]
                                                                                           │
                                                                                           ▼
                                                                                  Speaker audio (24 kHz)
```

What you can read off this diagram:

- The assistant audio is *generated at every frame*, not just during assistant turns. Silence is a valid output.
- The user stream is not a free mic feed — it is *predicted* from the Mimi-encoded mic. Training lets the model anticipate the user's audio, which is why it can emit backchannels timed to natural pauses.
- The inner-monologue text stream is a **deliberate controllability feature** of the architecture, not a logging add-on. It is the thing that lets you steer and audit a full-duplex model's content layer.
- There is no barge-in / turn-detection module. The "cancel on interrupt" problem does not exist in this design because the assistant's stream is already free to go silent whenever its probability distribution says to.

Where Moshi is weak, in production terms:

- **Content quality** is bounded by the backbone size (7B); it is not the equal of frontier text LLMs. Tool calling is not a first-class feature.
- **Controllability** is weaker than a text-first pipeline: changing the "personality" of the assistant means fine-tuning Mimi+Helium or careful prompting, not just editing a system prompt.
- **Scaling** a two-stream decoder has higher per-frame compute than one-stream speech-native, and the codec is lower-rate than a classic 48 kHz TTS.

### 2.3 OpenAI Advanced Voice Mode — architecture read

Advanced Voice Mode (AVM) is the voice surface of GPT-4o available in the ChatGPT app and, as a developer surface, through the OpenAI **Realtime API** (exercise-01). What OpenAI has disclosed publicly, framed as facts; where there is no disclosure, this document flags it.

What is public:

- AVM is **built on GPT-4o**, a natively multimodal model trained on text, images, and **audio** (input and output). The Realtime API exposes GPT-4o-class voice through the event protocol you walked in ex-01. (`https://openai.com/index/hello-gpt-4o/`)
- Audio-in and audio-out are **first-class modalities** — not an STT→LLM→TTS cascade behind the scenes. Affect-in (whispering, shouting, laughing) and affect-out (singing, laughing, modulating tone) are demonstrated behaviours. <!-- verify-at-runtime per model version -->
- The Realtime API **exposes server-side VAD** (`turn_detection.server_vad`) and an explicit **barge-in / truncate** protocol (`response.cancel`, `conversation.item.truncate`). This is the signature of a **half-duplex** surface: user audio and assistant audio share one logical channel, with the VAD switching who holds it.
- OpenAI has publicly discussed voice-cloning risk and voice-filter safeguards; the model will refuse to imitate unapproved voices. (`https://openai.com/index/hello-gpt-4o/`)

What is not public (and should be marked):

- Whether the internal generation path is a **single multimodal transformer** emitting audio tokens directly, or a **half-cascade** (LLM text → fast TTS) at inference time for some response types. <!-- source-unverified: OpenAI has not publicly disclosed the full inference-time architecture of Advanced Voice Mode as of 2026-11; treat any such claim as unverified unless you can point at an OpenAI post that confirms it. -->
- The exact audio tokenizer / rate used internally.
- Whether and how AVM uses an "inner monologue" style text transcript to steer its audio output.

Block diagram (external contract, not internal architecture):

```
     Mic audio  ──► [ WebSocket / WebRTC ]  ──►  [ GPT-4o-class realtime endpoint ]
                                                        │
                                                        │   (one logical channel: user OR assistant,
                                                        │    gated by server-side VAD + barge-in events)
                                                        │
     Speaker  ◄── [ WebSocket / WebRTC ]  ◄──────────────┘

     Server events the client sees:
        - input_audio_buffer.speech_started / .speech_stopped      (VAD)
        - response.audio.delta / .audio.done                       (assistant audio chunks)
        - response.audio_transcript.delta / .done                  (assistant transcript)
        - conversation.item.input_audio_transcription.completed    (user transcript)
        - response.cancel + conversation.item.truncate             (barge-in)
```

What you can read off this contract:

- It is half-duplex *by surface*. The `turn_detection` module and the `response.cancel + truncate` dance only make sense if the user and assistant are not simultaneously generating.
- Barge-in latency is bounded by VAD decision time + cancel round-trip + local-playout flush — low, but not zero and not frame-rate-bounded.
- No native backchannel generation: the assistant does not emit "mm-hm" while the user is talking. Any such behaviour would need to be scripted client-side with a short TTS pre-roll.
- Affect-in and affect-out are features of the **generation**, not of the **duplex structure**. AVM preserves and generates affect *within* the current turn, but it still owns one turn at a time.

### 2.4 Head-to-head

| Property | Moshi (full-duplex) | Advanced Voice Mode / Realtime API (half-duplex low-latency) |
|---|---|---|
| Duplex model | Full-duplex, two parallel generation streams + text stream. | Half-duplex, one logical channel, server VAD + barge-in. |
| Theoretical floor on reaction latency | ~ one frame (80 ms). <!-- verify-at-runtime --> | Bounded by VAD + cancel + playout (~ 200–600 ms). |
| Backchannels | Native; the model's own stream emits them. | Not native; would need client-side scripting. |
| Interruption handling | Degenerate (no concept of "turn"; the assistant can just stop emitting). | Explicit `response.cancel` + `conversation.item.truncate` protocol. |
| Text controllability | Inner-monologue text stream — can be read and (with care) steered. | System prompt via `session.update.instructions`; transcript is a side-channel on the assistant's audio. |
| Content model quality (reasoning, tool use) | 7B backbone; tool use is not a first-class feature. | Frontier-class (GPT-4o lineage) with native tool-calling via Realtime function tools. |
| Deployability | Open weights; runs on a single GPU; can be self-hosted. | API only; closed weights. |
| Affect preservation | Yes (native audio tokens both directions). | Yes (native audio modality). <!-- verify-at-runtime per model version --> |
| Compliance posture | You own it (self-hosted). | Inherits OpenAI's; HIPAA-eligible plans etc. verify per contract. |
| Current voice-agent product fit | Research / experimentation; special-purpose builds (e.g., companion agents where backchannels matter). | Production mainstream; the "it sounds like a person" low-latency assistant surface. |

Takeaway for the engineer: **half-duplex-with-fast-barge-in covers ~95% of what a voice-agent product calls "feels like a real conversation,"** and full-duplex closes the remaining gap on backchannels and on-latency reaction — at the cost of a much narrower content capability today. The trajectory is that frontier labs integrate the duplex idea into frontier-content-quality models; Moshi is the open-weights proof that the shape works.

### 2.5 Implications for your voice agent

If you are shipping a voice product today, the architectural implications fall out this way:

- **You are picking half-duplex with fast barge-in.** The design decisions that pay off are VAD tuning, local-playout flush discipline, and `truncate` correctness — not pretending you have full-duplex.
- **You cannot naively fake backchannels.** Dropping canned "mm-hm" TTS clips between user turns is uncanny. If backchannels matter to your product, note it as an architecture requirement and plan for the next generation of models rather than hacking it on top of today's surface.
- **Treat "simultaneous speech" as a product-level feature flag.** If you ever change out the content model (e.g., move from Realtime API to a Moshi-derivative), the UX has to be redesigned; it is not a drop-in migration.
- **When reading a model's spec sheet, ask "is this duplex or barge-in?"** Vendors sometimes advertise "full-duplex" when they mean "server-VAD with low-latency cancel." The test: does the model ever generate audio *while* the user is still talking?

## 3. Validation steps

A finished submission should pass each check.

1. **Precise definition.** The submission defines half-duplex and full-duplex with the "generation during listening" test; "allows interruption" alone is not enough.
2. **Moshi diagram.** The submission draws Moshi with Mimi + two generation streams + text inner-monologue stream, and names the 12.5 Hz / 80 ms frame rate (or cites the paper's current number with a verify marker).
3. **Advanced Voice diagram.** The submission draws AVM as a single-channel half-duplex surface with VAD + barge-in events. Any internal-architecture claim beyond what OpenAI has publicly disclosed is explicitly marked `<!-- source-unverified: ... -->`.
4. **Head-to-head table.** A side-by-side table that at minimum covers duplex model, backchannels, interruption, content quality, deployability, and one latency column.
5. **Takeaway on current production fit.** Submission states that today's production voice agents are half-duplex with fast barge-in and justifies that against product needs — not against vendor framing.
6. **Trajectory call.** Submission names the shape of the next generation (frontier content models with integrated duplex) and what would mark the crossover point.
7. **Sources separated.** Moshi-specific claims cite the Moshi paper / repo. AVM-specific public claims cite OpenAI's own posts. Non-disclosed internals are marked, not invented.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Duplex definition | 15 % | "Generation during listening" test; backchannel awareness | Definition anchored to the two-stream RQ-Transformer design |
| Moshi architecture | 20 % | Mimi + two streams + inner monologue; cites paper / repo | Block diagram with per-frame token flow; frame-rate reasoning |
| AVM architecture | 15 % | Half-duplex surface with VAD + barge-in; internals marked if not disclosed | Explicit distinction between what OpenAI has disclosed and what remains speculative |
| Head-to-head | 15 % | ≥ 6 rows covering the dimensions in §2.4 | Latency floor numerics with verify markers; compliance row |
| Production implications | 15 % | Current voice agents are half-duplex-with-fast-barge-in; call-out of backchannel limitation | Product feature-flag framing for duplex changes |
| Trajectory | 10 % | Names what crossover would look like | Mentions specific signals (open-weights frontier models with integrated duplex, closed-weights disclosures) |
| Source hygiene | 10 % | Moshi claims cite paper; non-disclosed AVM internals marked | Pull dates on paper / repo; `<!-- verify-at-runtime: ... -->` and `<!-- source-unverified: ... -->` used appropriately |

Scoring < 70 % is a fail-and-redo. The most common sub-70 failure is conflating low-latency barge-in with full-duplex, or guessing at AVM's internal architecture without flagging the guess.

## 5. Common mistakes

- **Calling Realtime API "full-duplex."** It is low-latency half-duplex. Full-duplex has a specific technical meaning (simultaneous generation on both sides) that Moshi meets and that current Realtime / Live surfaces do not. Marketing language is not a reliable guide here.
- **Confusing Moshi with a drop-in GPT-4o replacement.** Moshi is a research-grade 7B speech-native model. Its voice quality and duplex behaviour are striking; its content quality is not frontier.
- **Guessing at AVM's internals.** OpenAI has not publicly disclosed whether AVM is one multimodal transformer end-to-end, a half-cascade, or a mix per response type. Saying "it's a half-cascade" or "it's a single transformer" as a fact is a research-integrity miss. Use `<!-- source-unverified: ... -->`.
- **Reading barge-in latency without naming its floor.** Barge-in in a half-duplex system is bounded by VAD + cancel + playout flush. A submission that doesn't acknowledge this is treating latency as unbounded-good instead of VAD-bounded.
- **Ignoring the inner-monologue stream.** The inner-monologue text stream in Moshi is not an incidental logging feature — it is a controllability and auditability mechanism that *does not exist* in a pure audio-in/audio-out architecture. A submission that calls Moshi "audio-only" misreads the paper.
- **Thinking "full-duplex" means you no longer need a UI concept of turns.** Even with Moshi, products still carry a UI/UX concept of whose turn it is for accessibility and for logging. Full-duplex removes the architectural turn; the product turn is still a design choice.
- **Treating backchannels as cosmetic.** Backchannels are a strong user-experience signal. Users talking to a backchannel-capable agent self-report more "listened to" in Kyutai's qualitative evaluations. Dismissing them as "noise" misreads why Moshi matters.
- **No open-weights-vs-closed note.** Deployability is a decision variable. A submission that ignores whether weights are available misses one of the two most-important columns in the head-to-head.

## 6. References

Primary — architecture papers and documentation:

- Kyutai Moshi (paper, "Moshi: a speech-text foundation model for real-time dialogue") — `https://kyutai.org/moshi.pdf`
- Kyutai Moshi repository — `https://github.com/kyutai-labs/moshi`
- Kyutai site — `https://kyutai.org/`
- OpenAI "Hello GPT-4o" announcement (Advanced Voice context) — `https://openai.com/index/hello-gpt-4o/`
- OpenAI Realtime API — `https://platform.openai.com/docs/guides/realtime`

Supporting — the audio-codec and multi-stream Transformer lineage:

- EnCodec (Meta, neural audio codec that the Mimi line builds on) — `https://arxiv.org/abs/2210.13438`
- SoundStream (Google, earlier neural audio codec) — `https://arxiv.org/abs/2107.03312`
- AudioLM (hierarchical audio generation) — `https://google-research.github.io/seanet/audiolm/examples/`

Related exercises inside this module:

- exercise-01 (OpenAI Realtime) — the half-duplex surface whose protocol makes the duplex distinction concrete.
- exercise-02 (Gemini Live) — same, different vendor.
- exercise-04 (pipeline vs native) — this exercise supplies the "but how 'native' is native, really?" nuance.
- exercise-06 (open-weights tour) — Moshi, Qwen2-Audio, SeamlessM4T as a broader open-weights landscape.

Standards referenced indirectly (used by the compliance / safety framing of AVM):

- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — for the "disclosed vs undisclosed internals" research-integrity stance in §2.3.
- OWASP Machine Learning Security Top 10 — `https://owasp.org/www-project-machine-learning-security-top-10/` — relevant to adversarial-audio and voice-cloning risks called out in the AVM safety context.
- MITRE ATLAS — `https://atlas.mitre.org/` — threat-model reference; voice-cloning adversarial scenarios belong here.

Source policy note. All Moshi-architecture claims (12.5 Hz frame rate, 8 codebooks, two token streams, inner monologue, 7B Helium backbone, ~200 ms reaction floor) are drawn from the Kyutai Moshi paper and repository and should be re-verified on each read because the paper has been revised. Internal-architecture claims about Advanced Voice Mode are only included when OpenAI has publicly disclosed them; anything beyond that carries `<!-- source-unverified: ... -->`.
