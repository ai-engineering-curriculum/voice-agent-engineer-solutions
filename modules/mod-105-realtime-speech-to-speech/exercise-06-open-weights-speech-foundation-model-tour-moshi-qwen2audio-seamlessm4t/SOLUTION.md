# exercise-06 — Open-Weights Speech Foundation Model Tour: Moshi, Qwen2-Audio, SeamlessM4T — Reference Solution

## 1. Solution overview

The three open-weights systems in this tour each solve a *different* piece of the speech-foundation puzzle. A voice-agent engineer should be able to:

- Place each one on a *capability* axis (what inputs does it accept, what outputs does it emit).
- Place each one on an *architecture* axis (how is audio represented, what is the backbone, is it autoregressive, is it duplex).
- Know when to pick one vs. the others for a production voice-agent build vs. research vs. a research-adjacent product.

This solution gives, for each model, a crisp architecture read, a "what it is good for / what it is bad at" summary, and a code-sketch of how you would run it. It ends with a comparison table and a decision tree. All model-size, sampling-rate, and language-count numbers come from the respective model cards / papers and carry `<!-- verify-at-runtime: ... -->` markers.

<!-- verify-at-runtime: parameter counts, language counts, frame rates, release dates, and license terms can and do change as new model versions ship. Re-verify against each model's HuggingFace page, GitHub README, and paper before quoting externally. -->

## 2. Implementation

### 2.1 Moshi (Kyutai)

**One-sentence pitch:** Open-weights **full-duplex** speech-native model — it can speak and listen simultaneously, with an inner-monologue text stream for controllability.

**Architecture summary.** (Draws from `https://kyutai.org/moshi.pdf`.)

- **Audio tokenizer: Mimi.** Neural audio codec. **24 kHz mono audio → discrete tokens at 12.5 Hz with 8 codebooks per frame** (residual vector quantisation; a semantic codebook plus acoustic codebooks). One Mimi frame ≈ 80 ms. <!-- verify-at-runtime -->
- **Backbone: Helium 7B.** Autoregressive Transformer. Decodes multiple streams per forward via the "RQ-Transformer" multi-stream pattern.
- **Streams at inference:** two speaker audio streams (user + assistant, each in Mimi tokens) plus one **text stream** (inner monologue) for the assistant. The text stream leads the audio stream by a small offset.
- **Full-duplex:** trained on two-speaker overlap-tolerant dialog; the assistant stream can emit silence tokens or backchannels while the user is speaking.
- **Reaction latency:** on the order of a frame (~80 ms framerate floor, ~ 200 ms practical on a modern GPU). <!-- verify-at-runtime -->
- **Deployability:** open weights + inference code (`kyutai-labs/moshi` on GitHub). Candle (Rust) and PyTorch versions; `moshi-server` for serving. Single-GPU inference is practical.

**Good for.**
- Research into duplex behaviour, backchannels, overlap handling.
- Companion-agent products where "feels like a person is listening" (backchannels, rapid micro-reactions) is a measurable UX win.
- Self-hosted, no-API-key voice deployments.

**Bad at.**
- Frontier content capability. 7B backbone trails frontier LLMs on reasoning, tool use, and instruction following.
- Tool use is not a first-class feature as of the referenced release. <!-- verify-at-runtime: check for tool-use updates -->
- Multilinguality is narrow at release — English-focused. <!-- verify-at-runtime -->

**Running it (sketch).**
```bash
# Approximate; verify against current README.
pip install moshi
# or clone the kyutai-labs/moshi repo for the Candle (Rust) version
python -m moshi.run_inference  # see current README for the exact CLI entry point
```
The reference client in the repo is a WebSocket-based mic/speaker loop; most product integrations wrap `moshi-server` and drive it from the usual LiveKit/Pipecat voice-agent harnesses.

### 2.2 Qwen2-Audio (Alibaba)

**One-sentence pitch:** Open-weights audio-and-text **understanding** model — accepts speech / sound / music as input, produces **text** output. Not a TTS; not a duplex model.

**Architecture summary.** (Draws from the Qwen2-Audio technical report and HuggingFace model card: `https://github.com/QwenLM/Qwen2-Audio`, `https://huggingface.co/Qwen/Qwen2-Audio-7B-Instruct`.)

- **Audio encoder: Whisper-lineage.** The audio is encoded into a dense representation by a Whisper-style encoder, then projected into the LLM's embedding space. <!-- verify-at-runtime -->
- **Backbone:** Qwen2 LLM (7B in the publicly released variant). Autoregressive text decoder.
- **Modalities:** audio-in + text-in → text-out. Can do:
  - Automatic speech recognition (ASR) — audio to text in many languages.
  - Audio captioning — "there is a dog barking and a glass breaking in the background."
  - Spoken-dialog chat — accepts a spoken turn as the user message, replies in text.
  - Audio question-answering about ambient sound.
- **Tokens / rate:** the audio is not tokenised at fixed granularity for *output*; output is text tokens. Input is dense features. <!-- verify-at-runtime -->
- **Two released variants (at reference time):** a base pretrained Qwen2-Audio and `Qwen2-Audio-7B-Instruct` that is fine-tuned for dialog. <!-- verify-at-runtime -->
- **Deployability:** open weights on HuggingFace under the Qwen license; runs with `transformers`; multi-GPU not required for 7B.

**Good for.**
- Serving as the **STT + understanding** stage of a pipeline voice agent when you want affect-and-ambient awareness beyond vanilla ASR. "The user said 'can you help me?' and sounded upset" is a thing Qwen2-Audio can produce; vanilla Whisper cannot.
- Audio classification / tagging workflows (identifying coughs, music, environment).
- Accessibility assistants that have to describe the ambient audio world to the user.

**Bad at.**
- Being a voice agent on its own. You still need a TTS to speak back.
- Low-latency streaming — Qwen2-Audio is not designed for incremental audio streaming; it operates on whole-utterance inputs. <!-- verify-at-runtime: check for streaming variants -->
- Head-to-head frontier LLM reasoning (7B backbone).

**Running it (sketch).**
```python
# pip install transformers soundfile
from transformers import Qwen2AudioForConditionalGeneration, AutoProcessor
import soundfile as sf

MODEL = "Qwen/Qwen2-Audio-7B-Instruct"  # verify-at-runtime
processor = AutoProcessor.from_pretrained(MODEL)
model = Qwen2AudioForConditionalGeneration.from_pretrained(MODEL, device_map="auto")

audio, sr = sf.read("user_turn.wav")
conversation = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": [
        {"type": "audio", "audio_url": "user_turn.wav"},
        {"type": "text", "text": "What did the user say and what did they sound like?"},
    ]},
]
inputs = processor.apply_chat_template(
    conversation, audios=[audio], sampling_rate=sr, return_tensors="pt"
).to(model.device)
out = model.generate(**inputs, max_new_tokens=256)
print(processor.decode(out[0], skip_special_tokens=True))
```
Exact API shape changes between `transformers` versions; verify against the current Qwen2-Audio model card.

### 2.3 SeamlessM4T (Meta)

**One-sentence pitch:** Open-weights **multilingual** speech-and-text foundation model covering ASR, S2T (speech-to-text translation), T2T, T2S (text-to-speech), **and S2S (speech-to-speech translation)** across ~100 input / ~ 35 output languages. Not a duplex model; not a dialog model; a **translation / understanding** workhorse.

**Architecture summary.** (Draws from Meta's SeamlessM4T paper and HuggingFace model cards: `https://ai.meta.com/research/seamless-communication/`, `https://arxiv.org/abs/2308.11596`, `https://huggingface.co/facebook/seamless-m4t-v2-large`.)

- **Two generations released:** SeamlessM4T v1 (initial 2023 release), SeamlessM4T v2 ("v2-large", incorporating the **UnitY2** architecture; improves streaming and translation quality). <!-- verify-at-runtime: current release state, including Seamless Expressive / Seamless Streaming extensions -->
- **Unified task head.** Takes speech or text in, produces speech or text out, in any pairing (S2ST, S2TT, T2ST, T2TT, ASR).
- **Audio representation.** Uses a wav2vec 2.0-style audio encoder; text is tokenised with SentencePiece; audio output goes through a unit-based vocoder (HiFi-GAN variant). <!-- verify-at-runtime -->
- **Languages:** v2-large supports ~100 input languages for ASR and S2TT; ~100 for text translation; ~35 for speech output (S2ST / T2ST). <!-- verify-at-runtime: exact counts and the per-language task matrix -->
- **Variants:** base and large sizes; v2 adds **Seamless Streaming** (low-latency streaming S2TT/S2ST, under a different model name `SeamlessStreaming`) and **Seamless Expressive** (preserves speaker prosody and voice style across translation) released alongside v2. <!-- verify-at-runtime -->
- **License:** Meta custom license (non-commercial on some variants; check per-model-card). <!-- verify-at-runtime -->

**Good for.**
- Multilingual voice-agent routes where you need to accept dozens of input languages. Use as the STT + translator stage.
- Live translation products (streaming variant).
- Cross-lingual dubbing or voice-over workflows (expressive variant).

**Bad at.**
- Being a general voice-agent LLM. SeamlessM4T is a translation / recognition model, not a reasoning model. Chain it with an LLM for dialog tasks.
- Low-latency *dialog*. The streaming variant is for low-latency *translation*, which is a different latency budget (a few hundred ms to first partial translated chunk).
- Commercial use in some variants without additional licensing. Check the Seamless license for your use case before building on top. <!-- verify-at-runtime -->

**Running it (sketch).**
```python
# pip install transformers torchaudio
from transformers import AutoProcessor, SeamlessM4Tv2Model  # verify-at-runtime: class name
import torchaudio

MODEL = "facebook/seamless-m4t-v2-large"  # verify-at-runtime
processor = AutoProcessor.from_pretrained(MODEL)
model = SeamlessM4Tv2Model.from_pretrained(MODEL, device_map="auto")

audio, sr = torchaudio.load("input_english.wav")
# speech-to-speech translation: English → Spanish
inputs = processor(audios=audio, sampling_rate=sr, return_tensors="pt").to(model.device)
out = model.generate(**inputs, tgt_lang="spa", generate_speech=True)
audio_out = out.waveform.cpu().numpy()
# audio_out can be written to a WAV at the model's sample rate (16 kHz by default). Verify.
```
Class names and sample rates change across `transformers` versions; verify against the current model card.

### 2.4 Comparison at a glance

| Property | **Moshi** | **Qwen2-Audio** | **SeamlessM4T (v2)** |
|---|---|---|---|
| Purpose | Full-duplex **dialog** speech model | **Audio understanding** + dialog text | **Multilingual translation / ASR / S2S** |
| Input modalities | Audio (self + user streams) | Audio + text | Audio or text |
| Output modalities | Audio + text (inner monologue) | Text | Text or audio, depending on task |
| Duplex | **Full-duplex** | Non-duplex | Non-duplex; streaming variant is low-latency but still turn-based |
| Parameter scale (public) | ~ 7B (Helium) <!-- verify-at-runtime --> | 7B (Qwen2 LLM) <!-- verify-at-runtime --> | Several hundred M to ~ 2B+ depending on variant <!-- verify-at-runtime --> |
| Audio sample rate / tokens | 24 kHz / Mimi @ 12.5 Hz | Input: audio features, Output: text | Input: 16 kHz; output speech ~ 16 kHz vocoder <!-- verify-at-runtime --> |
| Language coverage | English-focused <!-- verify-at-runtime --> | Multilingual ASR + broad spoken-dialog <!-- verify-at-runtime --> | ~ 100 in, ~ 35 for speech out <!-- verify-at-runtime --> |
| Tool use | Not a first-class feature | Not a first-class feature | Not a first-class feature |
| Backchannels / affect-emitting | Yes (native) | No (text output) | No (translation) |
| Backchannels / affect-sensing | Yes | Yes (describes affect in output text) | Partial (expressive variant preserves prosody across translation) |
| License (at reference time) | Open; check Kyutai license. <!-- verify-at-runtime --> | Qwen license; check commercial terms. <!-- verify-at-runtime --> | Meta Seamless license; some variants non-commercial. <!-- verify-at-runtime --> |
| Deployment | Single GPU; `moshi-server` | HuggingFace `transformers` | HuggingFace `transformers`; Seamless repo |

### 2.5 Decision tree — which open-weights model slots in where

```
What role are you trying to fill in the voice stack?

── "I need something that speaks and listens simultaneously and sounds natural with backchannels"
   └─► Moshi. (Also pass the production-shape caveats from ex-05: content model is 7B.)

── "I need something that turns audio + ambient context into text (STT with affect/ambient awareness)"
   └─► Qwen2-Audio as the understanding stage of a pipeline; pair with a TTS (and, if dialog, an LLM) downstream.

── "I need to translate speech across many languages, possibly producing speech out"
   └─► SeamlessM4T v2 (or SeamlessStreaming for live, or SeamlessExpressive for voice-preserving dubbing).

── "I need a general-purpose dialog voice agent for production"
   └─► None of the three alone. Build a pipeline (ex-03) or use a speech-native API (ex-01/02).
      Use the open-weights models above for one stage of the pipeline where their capability fits.
```

The decision tree is the point of the tour: these three are not competitors. They occupy different capability niches, and a mature voice-agent engineer can look at a product requirement and know which, if any, applies.

### 2.6 A worked composition — "multilingual companion agent"

Example of combining the three in a single product, to illustrate how the niches compose. A graded submission is not required to build this, but should be able to sketch a comparable composition.

- User audio comes in → **SeamlessStreaming** detects language and streams a translation to English if needed. Store both the original-language transcript and the English pivot.
- English pivot + the raw audio → **Qwen2-Audio** for affect-and-ambient tagging ("user sounded upset; dog barking in background").
- Affect-tagged English pivot → text LLM (any — frontier LLM, Claude, Gemini, GPT-4o) for the actual dialog policy.
- LLM output → **SeamlessM4T** (or a conventional TTS) to produce output in the user's language.

The pipeline is longer than any single speech-native endpoint; the trade is multilingual breadth + affect-awareness + LLM choice freedom that no single current speech-native API covers.

## 3. Validation steps

A finished submission should pass each check.

1. **Three architecture reads.** Each model gets an architecture paragraph that names its audio representation, backbone, output modality, and one distinguishing property (Moshi: full-duplex / Mimi; Qwen2-Audio: audio-to-text understanding; SeamlessM4T: multilingual any-to-any).
2. **Capability niche named.** The submission identifies the niche each model occupies and contrasts it against the other two — not just lists them.
3. **Comparison table.** A side-by-side comparison with ≥ 8 rows covering purpose, modalities, duplex, scale, language coverage, license.
4. **Decision tree.** A terminal-in-finite-steps decision tree (or equivalent) that routes a product requirement to one of the three, or to "none of the above."
5. **Runnable sketch per model.** A code sketch (3–10 lines) that shows the practical invocation — not necessarily a running application, but concrete enough that a reader can tell the model class / method / arguments shape.
6. **License / commercial-use awareness.** Submission names the license of each and flags which are non-commercial-by-default. A submission that treats all three as unrestricted misreads the Seamless terms.
7. **"Not a voice agent alone" caveat.** Submission states clearly that none of the three is a drop-in general-purpose voice agent and that composition with other components is required.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Moshi read | 20 % | Mimi + duplex streams + inner monologue + Helium 7B | Frame-rate (12.5 Hz) numeric; reaction-latency floor cited |
| Qwen2-Audio read | 20 % | Audio encoder + Qwen2 LLM; audio-in → text-out | Affect/ambient-awareness distinction from vanilla ASR called out |
| SeamlessM4T read | 20 % | Any-to-any task head; ~ 100 in / ~ 35 out speech languages | v1 vs v2, Streaming vs Expressive variants distinguished |
| Comparison table | 10 % | ≥ 8 rows across the dimensions in §2.4 | Licenses, deployment footprint, language coverage all included with verify markers |
| Decision tree | 10 % | Finite, terminal, leads to one model or "none" per branch | Branches include the "none — compose or use a native API" branch |
| Composition example | 10 % | A worked composition that uses ≥ 2 of the three together | Composition includes the LLM choice and the affect-tagging path |
| License awareness | 5 % | Each license named with a verify marker | Commercial-use caveat for Seamless flagged |
| Source hygiene | 5 % | Every architecture claim cites the paper or model card URL | Pull dates + `<!-- verify-at-runtime: ... -->` on all numerics |

Scoring < 70 % is a fail-and-redo. The most common sub-70 failure: treating the three as competitors (they occupy different niches) or treating them as drop-in production-ready voice agents.

## 5. Common mistakes

- **Comparing Moshi's content quality against GPT-4o.** It is a 7B open-weights research model. The point of Moshi is the *duplex architecture*, not frontier reasoning. Benchmarks that score content quality miss the point.
- **Treating Qwen2-Audio as a TTS.** Qwen2-Audio outputs text. It is a *listener*, not a speaker. If you need to speak back, chain a TTS.
- **Treating SeamlessM4T as a chat model.** It translates. It does not hold dialog, reason, or follow instructions in the LLM sense.
- **Ignoring the Seamless commercial-license caveat.** Some Seamless variants are non-commercial. Shipping a product on one without re-licensing is an incident waiting to happen.
- **Assuming any of the three streams natively.** Moshi does; SeamlessStreaming does; Qwen2-Audio and the non-streaming SeamlessM4T do not. If your product needs streaming, name which variant you are using.
- **Pretending open-weights means "run it anywhere free."** GPU memory and inference cost are real. A 7B dense model takes roughly 14 GB in fp16 for weights alone; serving it is not free.
- **Writing one paragraph covering "open-weights speech models" without differentiating.** The whole exercise is about *niches*. A submission that says "they are all audio foundation models" fails.
- **Trusting parameter counts from third-party summaries.** Model cards get updated; sizes shift between v1 and v2 lines; the HF page is authoritative, not a blog post.

## 6. References

Primary — model papers, repositories, and model cards:

Moshi (Kyutai)
- Paper — `https://kyutai.org/moshi.pdf`
- Site — `https://kyutai.org/`
- Repository — `https://github.com/kyutai-labs/moshi`
- Model cards on HuggingFace — `https://huggingface.co/kyutai`

Qwen2-Audio (Alibaba)
- Repository — `https://github.com/QwenLM/Qwen2-Audio`
- Model card (Instruct) — `https://huggingface.co/Qwen/Qwen2-Audio-7B-Instruct`
- Technical report — `https://arxiv.org/abs/2407.10759`

SeamlessM4T (Meta)
- Research page — `https://ai.meta.com/research/seamless-communication/`
- v1 paper — `https://arxiv.org/abs/2308.11596`
- v2 / Seamless suite paper — `https://arxiv.org/abs/2312.05187`
- Repository — `https://github.com/facebookresearch/seamless_communication`
- v2-large model card — `https://huggingface.co/facebook/seamless-m4t-v2-large`
- SeamlessStreaming model card — `https://huggingface.co/facebook/seamless-streaming` <!-- verify-at-runtime -->
- SeamlessExpressive model card — `https://huggingface.co/facebook/seamless-expressive` <!-- verify-at-runtime -->

Supporting — underlying codec / encoder lineage:

- EnCodec — `https://arxiv.org/abs/2210.13438`
- SoundStream — `https://arxiv.org/abs/2107.03312`
- Whisper (audio encoder lineage behind Qwen2-Audio) — `https://github.com/openai/whisper`
- wav2vec 2.0 (encoder lineage behind SeamlessM4T) — `https://arxiv.org/abs/2006.11477`

Related exercises inside this module:

- exercise-05 (half vs full duplex) — supplies the duplex-definition that frames Moshi's "full-duplex" property.
- exercise-03 (Claude relay) — the pipeline shape into which Qwen2-Audio and SeamlessM4T slot naturally.
- exercise-04 (pipeline vs native) — the decision drill that the open-weights tour can turn into a measurable: "if I pick pipeline, which open-weights stages fit?"

Standards referenced indirectly:

- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — for the "disclose license and training-data lineage" research-integrity stance in §2.
- OWASP Machine Learning Security Top 10 — `https://owasp.org/www-project-machine-learning-security-top-10/` — relevant to open-weights supply-chain risks (weight tampering, pre-trained-model backdoors).
- MITRE ATLAS — `https://atlas.mitre.org/` — threat-model reference for adversarial-audio attacks against the audio encoder stages of each of these models.

Source policy note. All architecture claims (frame rate, codebook count, parameter counts, language counts, license terms) are drawn from the respective papers, repositories, or HuggingFace model cards listed above. Every quantitative claim carries `<!-- verify-at-runtime: ... -->` because the open-weights ecosystem updates frequently — new model variants and relicenses land faster than this document can; re-verify against the current model card before quoting externally.
