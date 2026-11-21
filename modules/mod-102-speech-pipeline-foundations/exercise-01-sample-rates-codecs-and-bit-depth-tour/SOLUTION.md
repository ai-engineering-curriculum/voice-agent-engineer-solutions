# exercise-01 — Sample Rates, Codecs, and Bit Depth Tour — Reference Solution

<!-- Source policy: the lesson file lives in the paired `voice-agent-engineer-learning` repo and is not reachable from this sandbox. The deliverables below are reconstructed from the exercise title plus the official references cited in §6. Any specific numeric claim is anchored to the spec it was taken from in the row where it appears. -->

## 1. Solution overview

This is a **tour / design exercise**: the learner walks through the three knobs that govern raw-audio quality and bandwidth before any codec runs — **sample rate**, **bit depth**, and **channel count** — and then overlays the four codec families a voice-agent engineer actually ships against (uncompressed PCM, companded PCM via G.711, low-bitrate speech-tuned Opus, and general-purpose AAC). There is no single correct table; a defensible submission is one where every row is grounded in the Nyquist-Shannon sampling theorem for the physical quantities and in an official codec spec (RFC 6716, ITU-T G.711) for the codec-specific claims.

This reference solution contains:

- A worked comparison table across the four codecs at the four sample rates most voice-agent engineers see (8 / 16 / 24 / 48 kHz).
- The bandwidth-math worked for one row so a reviewer can re-derive any of the others.
- The Nyquist / anti-aliasing rationale and the decision rubric a reviewer should look for.
- The rubric and common mistakes.

## 2. Worked implementation

### 2.1 The three physical knobs

| Knob | Units | Why it matters for voice | Typical voice-agent values |
|---|---|---|---|
| Sample rate `fs` | Hz | By Nyquist-Shannon, the usable audio bandwidth is at most `fs/2`. Telephony speech is band-limited to ~3.4 kHz and so can be carried at `fs = 8 kHz`; "wideband" and "superwideband" speech need 16 / 24 kHz; full-band music / general audio needs 44.1 or 48 kHz. | 8, 16, 24, 48 kHz |
| Bit depth | bits/sample | Sets the quantisation noise floor. 16-bit linear PCM gives ≈ 96 dB of dynamic range, which is well in excess of what speech requires. 8-bit *linear* PCM is audibly noisy; 8-bit *companded* PCM (μ-law / A-law in G.711) gets ~13-bit equivalent dynamic range at 8 bits/sample. | 16-bit linear PCM internally; 8-bit μ-law / A-law on PSTN. |
| Channels | count | Voice agents carry mono end-to-end for the ASR / TTS path. Stereo is only useful at capture (e.g., noise-cancelling microphone arrays) or playback (music / ambience), never through the codec path. | 1 (mono) |

**Nyquist-Shannon** (classical, see Oppenheim & Schafer; summarised on standard references): a continuous signal band-limited to `B` Hz is perfectly recoverable from samples taken at `fs > 2B`. In practice the anti-alias filter at capture is not a brick wall, so engineers budget ~10 % headroom; e.g., to carry 7 kHz of speech bandwidth, sample at 16 kHz, not 14 kHz.

### 2.2 Raw PCM bandwidth math

Uncompressed PCM bitrate is `fs × bit_depth × channels`. Worked for one row so a reviewer can re-derive any other:

- 16 kHz × 16-bit × 1 channel = **256 kbps** = 32 kB/s = 1.92 MB/min.

The other common values follow directly:

| Config | Raw PCM bitrate | 1-minute size |
|---|---|---|
| 8 kHz / 16-bit / mono | 128 kbps | 960 kB |
| 8 kHz / 8-bit μ-law / mono (G.711) | **64 kbps** | 480 kB |
| 16 kHz / 16-bit / mono | 256 kbps | 1.92 MB |
| 24 kHz / 16-bit / mono | 384 kbps | 2.88 MB |
| 48 kHz / 16-bit / mono | 768 kbps | 5.76 MB |
| 48 kHz / 16-bit / stereo | 1 536 kbps (1.5 Mbps) | 11.52 MB |

### 2.3 Codec overlay

The table below is the artefact a reviewer should see in a finished submission. Every row is grounded in the official spec for that codec; the Opus rows cite RFC 6716 §2.1.1, which lists supported internal sample rates (NB / MB / WB / SWB / FB at 8/12/16/24/48 kHz) and the operating-bitrate range of 6 – 510 kbps.

| Codec | Spec anchor | Sample rates it natively supports | Typical bitrate for 1-channel speech | Best-fit voice-agent surface | Trade-off a reviewer should flag |
|---|---|---|---|---|---|
| **Linear PCM (internal)** | — (no codec; representation only) | Anything you choose (16/24/48 kHz typical inside the agent) | Full raw rate (see §2.2) | In-process audio, SDK buffers, before/after codec boundary | Zero compression → only use between in-process stages, never on the wire |
| **G.711 μ-law** (PCMU) | ITU-T Rec. G.711 (1988) | 8 kHz only | 64 kbps | PSTN / SIP / Twilio `audio/PCMU` payload | Narrowband only; speech above ~3.4 kHz is discarded by the anti-alias filter |
| **G.711 A-law** (PCMA) | ITU-T Rec. G.711 (1988) | 8 kHz only | 64 kbps | European PSTN; Twilio `audio/PCMA` | Same narrowband trade; prevalent in Europe, μ-law in NA/JP |
| **Opus (SILK / Hybrid / CELT)** | RFC 6716 | 8 / 12 / 16 / 24 / 48 kHz (NB/MB/WB/SWB/FB) | 12 – 32 kbps speech-only (SILK/Hybrid); 32 – 96 kbps full-band music | WebRTC default (RFC 7874 mandates Opus support); LiveKit / Pipecat / OpenAI Realtime transport | Variable latency: frame sizes 2.5 / 5 / 10 / 20 / 40 / 60 ms (RFC 6716 §2.1.4). 20 ms is the WebRTC default. |
| **AAC-LC / HE-AAC** | ISO/IEC 14496-3 | 8 – 96 kHz | 32 – 128 kbps for mono voice at good quality | HLS / MP4 recording & playback; **not** a realtime voice-agent wire codec | Much higher algorithmic delay than Opus; not optimised for streaming turn-taking |

Two engineering notes a reviewer should look for:

1. **Opus internally resamples.** Even if your SDK says "48 kHz Opus", the codec may route through the SILK branch at 16 kHz for pure-speech content and the CELT branch at 48 kHz for music — RFC 6716 §2 describes the internal mode selection. You can't infer "what sample rate was transported" from the container; you infer it from the Opus `TOC` byte of each packet.
2. **G.711 is still the baseline on the carrier side.** Any voice agent that terminates a PSTN call via a CPaaS (Twilio, Vonage, Telnyx) will see μ-law at 8 kHz on the carrier-facing leg, regardless of what the agent-facing leg speaks. The resample-and-transcode at the media-server boundary is a first-class latency line item.

### 2.4 Decision rationale — picking a config

For a voice-agent engineer, the live decision reduces to three lines:

- **Internal audio graph**: 16-bit linear PCM at 16 kHz mono by default. This matches the native rate of most speech-tuned ASR / TTS models (Whisper, Deepgram Nova wideband mode, ElevenLabs 24 kHz output resampled down). Higher rates only pay off when the downstream model consumes them.
- **Browser / WebRTC transport**: Opus at 48 kHz, 20 ms frames, mono, VBR on, FEC on, DTX off (DTX off because voice-agent turn-taking depends on reliable signalling of silence — see mod-106). Opus is the wire codec every real browser understands (WebRTC Audio Codec Requirements, RFC 7874).
- **PSTN leg**: G.711 μ-law (NA) or A-law (Europe) at 8 kHz, non-negotiable because that's what the carrier gives you. Plan a resample + transcode step in the latency budget.

### 2.5 One worked scenario

*"A customer-support voice agent accepts both inbound PSTN calls (Twilio) and inbound WebRTC calls (web widget). ASR is Deepgram Nova (wideband, 16 kHz). TTS is ElevenLabs (24 kHz). What sample rates and codecs are on each leg, and where does the agent transcode?"*

- **PSTN leg (Twilio):** Opus not available on the carrier side. Twilio Media Streams emits 8 kHz μ-law frames. Agent receives 8 kHz μ-law → upsamples to 16 kHz linear PCM → feeds Deepgram.
- **WebRTC leg (web widget):** Browser sends Opus at 48 kHz. Agent SDK decodes Opus → downsamples to 16 kHz linear PCM → feeds Deepgram.
- **TTS back-path (PSTN):** ElevenLabs returns 24 kHz PCM (or encoded MP3, depending on stream mode) → downsample to 8 kHz → encode to μ-law → hand back to Twilio.
- **TTS back-path (WebRTC):** 24 kHz PCM → resample to 48 kHz → encode Opus → send over DataChannel / media track.

The **two resample boundaries per direction** are the design cost a reviewer should spot; each adds a frame or two of algorithmic delay and is a canonical place for sample-rate-mismatch bugs.

## 3. Validation steps

A finished tour should pass all of the following:

1. **Nyquist sanity.** For every sample rate in the table, the learner states the maximum usable audio bandwidth (`fs/2` minus filter transition band) and does not claim coverage above it.
2. **Raw-PCM bandwidth derivation.** One worked example of `fs × bit_depth × channels` with units carried through, not just a copied number.
3. **Codec anchor.** Every codec row points to its spec: RFC 6716 for Opus, ITU-T G.711 for μ-law / A-law, ISO/IEC 14496-3 for AAC. "I read it on Wikipedia" is not an anchor.
4. **Opus internal-mode awareness.** The learner notes that Opus has distinct speech (SILK) and music (CELT) branches and that the codec may operate at an internal rate different from the container's sample rate.
5. **Companded vs linear bit-depth distinction.** 8-bit μ-law is *not* 8-bit linear; the learner states that companding yields ~13-bit-equivalent dynamic range at 8 bits/sample.
6. **End-to-end worked scenario.** At least one scenario (PSTN + WebRTC + wideband ASR) is traced through the pipeline with explicit resample points.
7. **No unverified metrics.** Any claim of "Opus is X % better than G.711 on MOS" is either cited from the ITU-T MUSHRA / P.863 literature or omitted. Marketing MOS numbers are not accepted.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Sample-rate / bit-depth table | 15 % | 4 sample rates × 2+ bit depths covered with correct bitrates | Adds 44.1 kHz + stereo rows and the 3.4 kHz-vs-7 kHz bandwidth context |
| Codec coverage | 20 % | PCM, G.711 (μ-law **and** A-law), Opus all covered | AAC covered as "not a realtime voice codec" with the reason |
| Spec anchoring | 20 % | Every codec row cites RFC 6716 / G.711 / 14496-3 | Opus row cites the specific RFC 6716 section for the claim (e.g., §2.1.4 for frame sizes) |
| Bandwidth math | 15 % | One worked `fs × depth × ch` derivation with units | Table of all rows recomputed from the formula, not copy-pasted |
| Decision rationale | 15 % | Three-line "internal / WebRTC / PSTN" default declared with reason | Worked end-to-end scenario with every resample point named |
| Pitfalls called out | 15 % | 8-bit linear vs 8-bit μ-law distinction; Opus internal-mode awareness | Twilio media-server transcode called out as a latency line item (links to mod-109) |

A submission scoring under 70 % is a *fail-and-redo* — it usually means the learner has copied a codec comparison table without working the Nyquist math.

## 5. Common mistakes

- **"8-bit audio is low-fidelity."** Only for *linear* PCM. 8-bit μ-law via G.711 is the toll-quality telephony baseline and sits above the audible-artefact threshold for speech. The lesson wants you to make this distinction explicitly.
- **Picking 44.1 kHz for a voice agent.** 44.1 kHz is a CD / music artefact; 48 kHz is the WebRTC / modern pipeline default. Voice agents should default to multiples of 8 kHz (8/16/24/48) end-to-end so resampling is cheap.
- **Confusing "sample rate" with "Opus internal rate".** Opus ingests 48 kHz by convention but may run at 16 kHz internally for pure speech. The packet rate (one Opus packet per 20 ms frame) is independent of the internal rate.
- **Reporting bitrate without channels.** `256 kbps` is not meaningful without stating mono / stereo; `256 kbps mono` is.
- **Claiming a MOS delta between codecs.** Subjective quality claims need a MUSHRA / ITU-T P.863 test behind them. Vendor marketing MOS is not an official source.
- **Forgetting the PSTN leg exists.** A pure-WebRTC design is incomplete if the product also takes inbound calls; the submission should carry G.711 on at least one leg of the worked scenario.

## 6. References

Primary — official specs:

- **RFC 6716** — "Definition of the Opus Audio Codec" (IETF). — https://datatracker.ietf.org/doc/html/rfc6716 . Internal sample rates: §2.1.1. Frame sizes 2.5 / 5 / 10 / 20 / 40 / 60 ms: §2.1.4. Bitrate range 6 – 510 kbps: §2.1.1.
- **RFC 7874** — "WebRTC Audio Codec and Processing Requirements" (IETF). — https://datatracker.ietf.org/doc/html/rfc7874 . Mandates Opus (with specific operating constraints) + G.711 μ-law / A-law support in WebRTC endpoints.
- **ITU-T Recommendation G.711** — "Pulse Code Modulation (PCM) of Voice Frequencies" (1988). — https://www.itu.int/rec/T-REC-G.711/ . Defines μ-law (North America / Japan) and A-law (Europe) companding at 8 kHz / 64 kbps.
- **ISO/IEC 14496-3** — "Information technology — Coding of audio-visual objects — Part 3: Audio" (AAC family). Catalogue entry: https://www.iso.org/standard/76383.html .

Project documentation named by the title:

- **librosa** — https://librosa.org/doc/latest/ (resampling, `librosa.load` default sample-rate behaviour).
- **torchaudio** — https://pytorch.org/audio/stable/ (I/O backends; sample-rate normalisation).

Local curriculum anchors (adjacent mods this exercise hands off to):

- `mod-104-tts-production-engineering` — TTS output sample rates and the resample-to-wire step.
- `mod-107-telephony-webrtc-transport` — G.711 ↔ Opus transcode at the media-server boundary.
- `mod-109-latency-budget-architecture` — where each resample costs on the end-to-end budget.

Source policy note. OWASP ML Top 10, MITRE ATLAS, and NIST AI RMF (the three standards named in the packet) are not load-bearing for this exercise; audio-rate selection is a DSP / standards question, not an ML-security question. They are referenced in later modules (mod-108 safety / compliance) where they do apply.
