# exercise-06 — Walk a Neural Audio Codec (EnCodec or Mimi) — Reference Solution

<!-- Source policy: the lesson file lives in the paired `voice-agent-engineer-learning` repo and is not reachable from this sandbox. The deliverables below are reconstructed from the exercise title plus the official references cited in §6. Any specific numeric claim is anchored to the paper or project it was taken from in the row where it appears. -->

## 1. Solution overview

This is a **reading / walkthrough exercise**: the learner picks one neural audio codec — **EnCodec** (Meta, 2022) or **Mimi** (Kyutai, 2024, the codec underneath the Moshi realtime speech-to-speech model) — and walks through its architecture end to end:

1. What the **encoder** does (raw waveform → latent sequence).
2. How **residual vector quantisation (RVQ)** turns continuous latents into a small number of discrete tokens per frame.
3. What the **decoder** does (tokens → reconstructed waveform).
4. How the tokens-per-second rate and bitrate are computed from the config.
5. **Why a voice-agent engineer cares**: these codecs are the input/output surface for modern speech-to-speech models (mod-105). They are not realtime wire codecs today the way Opus is; they are *representation* codecs for the model.

The deliverable is a short written walkthrough (≤ 3 pages) plus optional working code that encodes / decodes a single utterance with the real model. There is no single correct narrative; a defensible one grounds every number in the paper (EnCodec: arXiv:2210.13438 — Défossez et al., "High Fidelity Neural Audio Compression"; Mimi: arXiv:2410.00037 — Défossez et al., "Moshi: a speech-text foundation model for real-time dialogue", which introduces Mimi in §3).

This reference solution is a worked walkthrough of **EnCodec**, with the equivalent-for-Mimi notes called out so the learner who chose Mimi can map through.

## 2. Worked implementation — EnCodec walkthrough

### 2.1 The architecture at the altitude an engineer uses it

```text
Waveform (24 kHz mono, float32)
   │
   ▼                 1-D convolutional encoder
┌──────────────────┐ (streaming-capable, strided to downsample)
│  Encoder E(x)    │  → latent sequence z ∈ R^(T', D)     T' = T / stride, D = 128
└──────────────────┘
   │
   ▼                 Residual Vector Quantiser (RVQ)
┌──────────────────┐ N_q codebooks of size K each
│  RVQ Q(z)        │  → N_q streams of discrete tokens per frame
└──────────────────┘    (each token indexes one codebook entry)
   │
   ▼                 1-D transposed-conv decoder
┌──────────────────┐ mirror of the encoder
│  Decoder D(q)    │  → reconstructed waveform x̂
└──────────────────┘
```

The three numbers a voice-agent engineer needs to know (EnCodec 24 kHz model, from the paper):

- **Sample rate**: 24 kHz mono input (there is also a 48 kHz stereo variant).
- **Frame rate (`fs_frame`)**: ~**75 frames per second** at 24 kHz input (320× stride end-to-end).
- **Codebook bits**: each codebook has 1024 entries → **10 bits per token**.
- **Number of codebooks (`N_q`)**: configurable at inference — the paper trains for `N_q ∈ {1, 2, 4, 8, 16, 32}`.

### 2.2 Bitrate math (do this yourself before quoting it)

Bitrate = `fs_frame × N_q × bits_per_codebook_entry`.

For EnCodec at 24 kHz with `N_q = 8` and 1024-entry codebooks:

- `bitrate = 75 frames/s × 8 codebooks × 10 bits = 6 000 bits/s = 6 kbps`.

The paper reports operating points at 1.5 / 3 / 6 / 12 / 24 kbps at 24 kHz, obtained by varying `N_q` ∈ {2, 4, 8, 16, 32}. The learner should recompute at least one of these from the formula above, to show the derivation lands.

(Mimi equivalent, per the Moshi paper: **12.5 Hz frame rate** on 24 kHz input (via a larger stride + a Transformer at the bottleneck); **8 codebooks** at inference for Moshi's RQ-Transformer; nominal bitrate ~1.1 kbps for pure speech. The 12.5 Hz frame rate is stated in the Moshi paper abstract; the codebook count and bits-per-codebook are described in §3 of Défossez et al. 2024 — cite directly to that section when quoting the exact numbers. The dramatic drop vs. EnCodec is the point of Mimi — fewer tokens per second of audio means the downstream language model has fewer tokens to generate.)

### 2.3 What each stage actually does

**Encoder.** A stack of 1-D convolutions with strides that multiply to the overall stride (320× for EnCodec @ 24 kHz). Streaming-capable means the encoder uses causal convolutions (each output sample depends only on past inputs) so you can push 10 ms at a time and get the latent at the end of that 10 ms. This is *the* property that distinguishes modern neural audio codecs from older VAE-based ones: streaming is a design goal, not an afterthought.

**Residual Vector Quantiser (RVQ).** A pyramid of codebooks. Quantise the latent against the first codebook, subtract the chosen codeword from the latent to get the first residual, quantise the residual against the second codebook, and so on for `N_q` stages. The output is `N_q` discrete token indices per frame. Two engineering consequences:

- **`N_q` is a dial at inference.** Drop the last `k` codebooks and you drop the bitrate linearly; quality degrades gracefully. This is what gives EnCodec its 1.5 / 3 / 6 / 12 / 24 kbps grid.
- **Each codebook sees a different distribution.** The first codebook captures coarse structure; later codebooks capture fine residuals. A downstream model (e.g., MusicGen, AudioGen, Moshi) can model codebooks in a staged way: coarse codebook autoregressively in time, fine codebooks conditioned on the coarse ones.

**Decoder.** A mirror of the encoder — transposed convolutions that upsample back to the waveform. Trained with a combination of (a) a reconstruction loss (time + multi-scale STFT), (b) an adversarial loss from multi-scale STFT discriminators, and (c) codebook commitment losses (standard VQ-VAE training). For the engineer: the decoder is a function; you call it with the tokens you have and you get audio back.

### 2.4 Working Python walk (optional)

```python
# encodec_walk.py
# Reference walk for mod-102 exercise-06, EnCodec branch.
# Requires: pip install encodec==0.1.1 torch torchaudio

import torchaudio
from encodec import EncodecModel
from encodec.utils import convert_audio

# 1. Load the 24 kHz EnCodec checkpoint and pick an operating bitrate.
model = EncodecModel.encodec_model_24khz()
model.set_target_bandwidth(6.0)          # 6 kbps → N_q = 8 at 75 Hz × 10 bits

# 2. Load an arbitrary wav and normalise it to the model's input format.
wav, sr = torchaudio.load("example.wav")
wav = convert_audio(wav, sr, model.sample_rate, model.channels)   # (1, T) @ 24 kHz
wav = wav.unsqueeze(0)                                            # (B=1, 1, T)

# 3. Encode: get a list of (codes, scale) frames. For streaming use, there is
#    one frame per segment; for bulk use, the model emits one segment covering
#    the whole clip.
with torch.no_grad():
    encoded_frames = model.encode(wav)

# 4. Inspect what the encoder produced.
codes, _ = encoded_frames[0]              # codes: (B, N_q, T')
print("codes shape:", codes.shape)        # e.g. torch.Size([1, 8, 75 * seconds])
print("unique codebook entries per stream:",
      [int(codes[:, q].unique().numel()) for q in range(codes.shape[1])])

# 5. Decode and compute reconstruction error.
with torch.no_grad():
    decoded = model.decode(encoded_frames)   # (B, 1, T̂)

err = (decoded[..., : wav.shape[-1]] - wav).abs().mean().item()
print(f"mean |x̂ - x| = {err:.5f}  (float32 waveform in [-1, 1])")
```

Three things to observe from the output:

1. The `codes` tensor shape is `(B, N_q, T')` with `T' ≈ 75 × seconds` and `N_q = 8` at 6 kbps. This confirms the §2.2 bitrate math.
2. The number of unique entries per codebook is well under 1024 for a single short utterance — codebooks are used sparsely per sample but densely across a dataset.
3. Reconstruction is lossy. `mean |x̂ - x|` is small but non-zero; perceptually the decode usually sounds close to the input at 6 kbps on clean speech.

### 2.5 Why a voice-agent engineer cares

Neural audio codecs matter to voice-agent engineers for three reasons, in increasing order of importance:

1. **Representation for S2S models.** Mimi is the input/output surface for Moshi; SNAC is the surface for several Kyutai and open S2S models; EnCodec is the surface for MusicGen / AudioGen. When you integrate an S2S model (mod-105), you are not passing waveforms around — you are passing codec tokens. Latency is dominated by tokens-per-second, and the codec frame rate (`fs_frame`) sets a floor on how fast the downstream LM must generate.
2. **Token-rate budget for latency.** At 75 Hz × `N_q` codebooks (EnCodec), a 10-second response needs 750 × `N_q` tokens; at 12.5 Hz × `N_q` (Mimi), it needs 125 × `N_q`. The lower frame rate is exactly why Mimi was built — it cuts the LM's generation load by ~6×. This is a first-class latency-budget line item (mod-109).
3. **Not a wire codec today.** Despite the name "codec", EnCodec and Mimi are not what you put between a browser and a media server — the industry has not standardised transport over them, and the ecosystem (SIP, WebRTC) is Opus + G.711. They sit *inside* the model's input/output boundary, after the Opus decoder on the receive side and before the Opus encoder on the send side.

### 2.6 EnCodec vs Mimi — the engineer's compare-table

| Dimension | EnCodec (2022) | Mimi (2024) |
|---|---|---|
| Native input rate | 24 kHz mono / 48 kHz stereo | 24 kHz mono |
| Frame rate | ~75 Hz | ~12.5 Hz |
| Typical codebooks at inference | 2 – 32 (grid of bitrates) | 8 (Moshi config) |
| Typical bitrate | 1.5 – 24 kbps | ~1.1 kbps nominal |
| Latency floor (algorithmic) | Streaming-capable, short frame delay | Streaming-capable, short frame delay — Mimi's larger stride is implemented with causal convs + a Transformer bottleneck |
| Target use | General audio (music + speech) compression and as representation for MusicGen / AudioGen | Speech-tuned; representation for Moshi realtime S2S |
| Licence (upstream) | MIT (Meta `facebookresearch/encodec`) | Apache-2.0-style, as part of the Moshi release from Kyutai (`kyutai-labs/moshi`); check the Mimi model-artefact licence in the Moshi repo before redistributing. |

A voice-agent engineer writing today should expect to see **Mimi** (via Moshi or follow-ups) as the codec-at-the-model-boundary for open S2S systems, and **EnCodec**-family representations for general audio generation.

## 3. Validation steps

A finished walkthrough should pass all of the following:

1. **Architecture named end to end.** Encoder → RVQ → decoder, each stage described in a sentence or two. Not just a diagram.
2. **Frame rate and codebook bits cited from the paper.** EnCodec: 75 Hz / 1024-entry codebooks / 10 bits per token at 24 kHz. Mimi: 12.5 Hz on 24 kHz input.
3. **Bitrate math re-derived.** At least one operating point (e.g., `N_q = 8` → 6 kbps for EnCodec) is recomputed from `fs_frame × N_q × bits`.
4. **RVQ dial-down explained.** The learner states that `N_q` is a dial at inference and that dropping codebooks lowers bitrate with graceful quality loss.
5. **Why it matters for voice-agent engineers.** At least the three points in §2.5: representation for S2S, latency / token-rate implications, and the non-wire-codec caveat.
6. **Working code or declared skip.** Either a run of `encodec` or a declared "I did not run the code because X" — fabricated output numbers are not acceptable.
7. **No invented metrics.** PESQ / MOS claims are either from the paper (and cited to the paper's Table) or omitted.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Architecture coverage | 20 % | Encoder / RVQ / decoder each described | Each stage grounded to a specific section of the paper (e.g., EnCodec §3.1 encoder) |
| Numbers from the paper | 20 % | Frame rate, codebook size, bits/token cited correctly | Also cites the discriminator / loss recipe from the paper |
| Bitrate math | 15 % | One operating point re-derived | Full operating grid re-derived and matched to the paper's Table |
| RVQ dial-down | 10 % | `N_q` is explicitly described as a bitrate / quality dial at inference | Walks what the coarse vs fine codebooks capture and what that implies for staged modelling (MusicGen / Moshi) |
| Engineer relevance | 15 % | All three points in §2.5 covered | Links codec frame rate to the mod-109 latency budget with numbers |
| Working code or declared skip | 10 % | Script runs or absence is declared honestly | Script also runs at two bitrates and compares reconstruction error |
| Cross-codec compare | 10 % | EnCodec vs Mimi table (§2.6) filled in | Includes a third codec (SoundStream or DAC) with the same dimensions |

Submissions under 70 % usually fail on one of two places: fabricating the frame rate ("it's 50 Hz because that sounded right") or claiming S2S models consume waveforms directly.

## 5. Common mistakes

- **Treating EnCodec / Mimi as a wire codec.** They are not what goes between browser and SFU; Opus is. These are representation codecs that live *inside* the model's input/output.
- **Confusing `N_q` (codebooks) with `K` (codebook size).** `N_q` is the number of stages in the RVQ pyramid (typically small, 2–32). `K` is how many entries each codebook has (typically 1024). Bitrate scales with `N_q × log2(K)`.
- **Quoting the wrong frame rate.** EnCodec 24 kHz is ~75 Hz; Mimi is 12.5 Hz. Mixing them up changes every downstream latency number by ~6×.
- **Assuming non-streaming training → non-streaming inference.** EnCodec and Mimi are trained offline but designed for streaming with causal convolutions; the paper sections on causality are worth reading before claiming otherwise.
- **Forgetting the decoder.** Walking only the encoder + RVQ is half the story. The decoder is a non-trivial piece of the training recipe (multi-scale STFT discriminators) and is where reconstruction quality lives.
- **Picking a codec because the README said "high fidelity".** Pick the codec to match what the downstream S2S model expects. Mimi for Moshi; SNAC for its target models; EnCodec for MusicGen / AudioGen. The model tells you the codec, not the other way around.
- **Inventing MOS numbers.** Cite Table X of the paper or omit. Vendor / blog MOS numbers don't count.
- **Skipping the "why it matters" section.** Walking an architecture without landing why a voice-agent engineer cares is a reading exercise, not an engineering one.

## 6. References

Primary — papers and official project pages:

- **EnCodec paper** — A. Défossez, J. Copet, G. Synnaeve, Y. Adi, "High Fidelity Neural Audio Compression" (Meta AI, 2022). arXiv: https://arxiv.org/abs/2210.13438 . Architecture / RVQ / loss recipe.
- **EnCodec code** — Meta Research reference implementation: https://github.com/facebookresearch/encodec .
- **Moshi / Mimi paper** — Défossez, Mazaré, Orsini, Royer, Pérez, Jégou, Grave, Zeghidour, "Moshi: a speech-text foundation model for real-time dialogue" (Kyutai, 2024). arXiv: https://arxiv.org/abs/2410.00037 . Mimi codec: §3. Moshi RQ-Transformer: §4.
- **Moshi / Mimi code** — Kyutai reference implementation: https://github.com/kyutai-labs/moshi .
- **SoundStream** — predecessor neural codec (Google, 2021), useful context for EnCodec. arXiv: https://arxiv.org/abs/2107.03312 .
- **Descript Audio Codec (DAC)** — https://github.com/descriptinc/descript-audio-codec (open-source practitioner codec, useful compare point).

Local curriculum anchors:

- `mod-105-realtime-speech-to-speech` — the module where Mimi-style codecs are the input/output surface of the S2S model.
- `mod-109-latency-budget-architecture` — how the codec frame rate flows into the end-to-end budget.
- `mod-110-multimodal-input-design` — where learned front-ends replace handcrafted features.

Source policy note. OWASP ML Top 10, MITRE ATLAS, and NIST AI RMF are not load-bearing for a codec walkthrough. They become relevant when neural codecs are used as the trust boundary for user audio (mod-108), e.g., the question "what cleartext speech content is reconstructable from stored codec tokens?" — which is a mod-108 topic, not a mod-102 one.
