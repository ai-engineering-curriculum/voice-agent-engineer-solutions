# exercise-04 — Opus and G.711 Codec Decision Drill — Reference Solution

<!-- Source policy: the lesson file lives in the paired `voice-agent-engineer-learning` repo and is not reachable from this sandbox. The deliverables below are reconstructed from the exercise title plus the official references cited in §6. Any specific numeric claim is anchored to the spec it was taken from in the row where it appears. -->

## 1. Solution overview

This is a **design / decision exercise.** The learner is given a small set of voice-agent deployment scenarios and asked to pick Opus or G.711 for each leg of each pipeline, defend the choice from the spec (RFC 6716 for Opus, ITU-T G.711 for μ-law / A-law), and call out the transcode boundaries where both codecs appear on the same call.

There is no single correct table: the two codecs cover different problem spaces, and a defensible submission names the trade — **where** on the network a given leg runs — not the codec itself. This reference solution contains:

- A worked scenarios table and the decision rationale.
- The two-page quick-reference: Opus vs. G.711 side-by-side, every row anchored to the spec.
- The rubric and the common mistakes.

## 2. Worked implementation

### 2.1 The two codecs side-by-side

| Dimension | G.711 μ-law / A-law | Opus |
|---|---|---|
| Spec | ITU-T Rec. G.711 (1988) | IETF RFC 6716 (2012) |
| Sample rate | 8 kHz only | 8 / 12 / 16 / 24 / 48 kHz (RFC 6716 §2.1.1 — NB/MB/WB/SWB/FB) |
| Bitrate | Fixed 64 kbps (8 bits × 8 kHz) | 6 – 510 kbps, variable (RFC 6716 §2.1.1) |
| Frame size | 20 ms conventional (packetisation), codec itself is sample-wise | 2.5 / 5 / 10 / 20 / 40 / 60 ms (RFC 6716 §2.1.4) |
| Algorithmic delay | ~0 (companding is instantaneous) | ~22.5 ms at 20 ms frame + 2.5 ms lookahead (SILK/Hybrid); CELT-only mode omits the SILK lookahead. See RFC 6716 §1.1 and §2 for the mode-by-mode narrative. |
| FEC / PLC | External (RTP PLC in RFC 3551-style profiles) | Built-in in-band FEC (LBRR) and PLC (RFC 6716 §2.1.7) |
| DTX / CN | External (comfort noise) | Built-in DTX (RFC 6716 §2.1.9) |
| Complexity (encode) | Near-zero (table lookup) | Non-trivial (SILK + CELT); tunable via `complexity` 0–10 |
| Where it lives | PSTN, SIP trunks, interop with TDM carriers | WebRTC, streaming SDK transports, modern real-time pipelines |
| Patent / royalty status | ITU-T recommendation, patents expired decades ago; universally free to implement | Royalty-free (RFC 6716 preface, IETF IPR disclosures) |
| Interop requirement | Mandatory for any PSTN / SIP termination | Mandatory for WebRTC (RFC 7874) |
| What it compresses | Dynamic range, via logarithmic companding | Spectrum, via SILK (speech) and/or CELT (music) with mode selection |

### 2.2 Scenarios and the defensible answer for each

Below are the scenarios a voice-agent engineer sees most often. For each, the "defensible" answer names the leg + the codec; the reason cites the spec or the transport.

**Scenario A — Inbound PSTN call → voice agent → GPT-4o Realtime.**
*Carrier leg*: **G.711** (μ-law in NA, A-law in Europe). Non-negotiable: the carrier gives you 8 kHz companded PCM. *Agent-to-model leg*: whatever the model's SDK uses — for OpenAI Realtime that is 24 kHz PCM under the hood, often tunnelled as Opus-in-WebRTC. The pipeline has **two codec boundaries**: a G.711 ↔ 16 kHz linear PCM transcode at the CPaaS edge, and a 16 kHz PCM ↔ Opus transcode at the WebRTC leg to the model.

**Scenario B — Browser-based web widget → voice agent → GPT-4o Realtime.**
*Capture leg*: **Opus** at 48 kHz (browsers require it, RFC 7874). *Agent-to-model leg*: Opus continues through LiveKit / OpenAI Realtime; no transcode is needed if you stay on WebRTC end-to-end. The simplest pipeline is the Opus-everywhere pipeline; G.711 appears only if you also need to answer PSTN calls.

**Scenario C — Mobile app with its own SDK (iOS / Android, cellular + Wi-Fi mixed).**
*Capture leg*: **Opus** (every modern SDK supports it; RFC 6716 is implementable on both platforms). Variable network means you also want Opus's in-band FEC and VBR on, because the codec's PLC + FEC is the *only* thing between you and gaps on a dropping cellular link.

**Scenario D — Outbound call via SIP trunk (Twilio / Vonage / Telnyx) to a human.**
*Carrier leg*: **G.711**. Opus-over-SIP exists in spec (RFC 7587), but most TDM-adjacent carriers and most consumer SIP endpoints don't speak it; defaulting to G.711 is the interop-safe choice.

**Scenario E — Internal agent-to-agent RPC where both sides are in your control (e.g., voice-agent-to-voice-agent handoff).**
Not a codec decision — this is **linear PCM** over a reliable transport. You only pay the codec cost when the audio crosses a network link that charges by bandwidth or a boundary that requires a standard payload.

**Scenario F — Recording audio for later analytics / eval.**
Not a realtime codec choice at all. Record the raw 16 kHz / 24 kHz / 48 kHz linear PCM (or lossless FLAC) if storage allows; both Opus and G.711 are lossy and will remove information you may later want to score ASR / eval against (mod-112).

### 2.3 Scenario table (reviewer-facing artefact)

The reviewer-facing artefact is a single table like this; this is the shape a submission should land:

| # | Scenario | Capture leg codec | Transport codec on the model-facing leg | Transcode points | Why (spec / interop anchor) |
|---|---|---|---|---|---|
| A | Inbound PSTN → voice agent → Realtime model | G.711 μ-law (NA) / A-law (EU) @ 8 kHz | Opus @ 48 kHz (WebRTC to model) | Twilio media server (μ-law↔PCM), model-SDK edge (PCM↔Opus) | G.711: ITU-T G.711. Opus mandatory on WebRTC: RFC 7874. |
| B | Browser → voice agent → Realtime model | Opus @ 48 kHz | Opus @ 48 kHz | None if end-to-end WebRTC | Both ends speak the WebRTC baseline codec (RFC 7874). |
| C | Mobile SDK (cellular + Wi-Fi) → agent | Opus @ 24–48 kHz, VBR, in-band FEC on | Opus | None if SDK speaks Opus directly | Lossy-network FEC / PLC: RFC 6716 §2.1.7. |
| D | Outbound SIP call to a human | N/A (agent produces) | G.711 μ-law / A-law | PCM → G.711 at SIP edge | SIP carrier interop: ITU-T G.711. |
| E | Agent ↔ Agent internal handoff | Linear PCM | Linear PCM | None | No standard codec needed over a reliable internal transport. |
| F | Recording for later ASR / eval | Raw 16 kHz linear PCM or FLAC | — | — | Avoid lossy artefacts in eval ground truth (see mod-112). |

### 2.4 Decision rationale — three rules that cover 95 % of the calls

1. **Carrier boundary → G.711.** If your audio touches a PSTN/SIP trunk, that leg is G.711. Everything else is an optimisation on top of that fact.
2. **WebRTC boundary → Opus.** If your audio touches a browser or any WebRTC endpoint, that leg is Opus. RFC 7874 mandates it, and there is no "fall back to G.711" that is faster or simpler in that leg.
3. **Internal links → linear PCM.** Inside your own process / container boundary, don't codec at all; pay the resample cost only once at each external edge. Codec round-trip inside the pipeline is pure quality loss with no bandwidth payoff.

### 2.5 Call-out: the three places Opus can bite you

- **Frame-size churn.** RFC 6716 §2.1.4 allows a packet to contain multiple frames at 2.5 / 5 / 10 / 20 / 40 / 60 ms. If your stack renegotiates PTime mid-stream (rare but it happens with network adaptation), your ring buffer and jitter buffer both have to cope. Default to 20 ms and keep it fixed.
- **DTX.** Opus DTX emits comfort-noise packets during silence. Voice-agent turn-taking logic that counts "no packet" as "end-of-turn" will break if DTX is on. Default to DTX off for voice agents; use VAD on raw PCM for turn-taking (mod-106).
- **FEC.** In-band FEC (LBRR) adds ~1 packet of extra latency on the receive side. On a stable link this is free quality insurance; on a very low-latency path you may want to turn it off. This is a budget call, not a correctness call.

### 2.6 Call-out: the one place G.711 can bite you

- **Bandwidth cap.** G.711 at 8 kHz only reproduces audio up to ~3.4 kHz (anti-alias filter below Nyquist). Any wideband speech content — accents, sibilance, multilingual — is silently clipped before the codec runs. You cannot recover wideband quality by transcoding G.711 up; the information is gone at capture. This is why PSTN voice agents score worse on ASR than WebRTC voice agents even when the model is identical.

## 3. Validation steps

A finished drill should pass all of the following:

1. **At least four scenarios** covered, including one PSTN (G.711 forced) and one WebRTC (Opus forced). "All Opus, everywhere" is a failing answer because it ignores the PSTN reality.
2. **Spec anchoring.** Every codec choice cites either RFC 6716 or ITU-T G.711 / RFC 7874 for the mandate. "Opus is faster" is not an anchor.
3. **Transcode-point audit.** Every scenario names where the pipeline transcodes, or declares explicitly that no transcode is needed.
4. **Opus knobs declared.** The Opus rows name at least frame size, FEC, and DTX — the three knobs that decide whether Opus plays well with a voice-agent pipeline.
5. **G.711 flavour declared.** μ-law vs A-law is called out (not just "G.711"), with the region the choice implies.
6. **No MOS marketing.** No scenario uses "better perceived quality" as a decision point without a MUSHRA / ITU-T P.863 anchor.
7. **Recording decision.** At least one scenario names raw PCM / FLAC as the right choice for recording; lossy codecs are not acceptable for eval ground truth.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Scenario coverage | 20 % | ≥ 4 scenarios, including 1 PSTN + 1 WebRTC + 1 mobile | All six scenarios from §2.3 plus one novel scenario the learner proposes |
| Spec anchoring | 25 % | Every codec row cites RFC 6716 / RFC 7874 / ITU-T G.711 by document (not just "the Opus RFC") | Opus rows cite the specific RFC 6716 section (§2.1.4, §2.1.7, §2.1.9) |
| Transcode-point audit | 20 % | Every scenario names where the pipeline transcodes or declares "no transcode" | Latency cost of each transcode is estimated and sourced |
| Opus knobs | 15 % | Frame size, FEC, DTX each declared with a default for voice agents | VBR / complexity / channels also declared and justified |
| G.711 flavour | 10 % | μ-law vs A-law declared per region | Also names the specific RTP payload type / Twilio stream codec name |
| Common-pitfalls callout | 10 % | At least two of §2.5 or §2.6 called out in the writeup | Learner argues against "Opus everywhere" as a design mistake with a concrete scenario |

Submissions under 70 % typically fail on either spec anchoring (citing marketing sites instead of the RFC) or on the PSTN leg (leaving it as "we'll use Opus" when the carrier doesn't support it).

## 5. Common mistakes

- **"We'll use Opus everywhere."** Breaks at the PSTN boundary. RFC 7587 defines Opus-over-SIP, but most carrier interop still only guarantees G.711. A pipeline that assumes end-to-end Opus will silently fail on inbound calls from a landline.
- **"G.711 is legacy, we can skip it."** G.711 is the live baseline for every carrier-originated call. There is no "dropping G.711 support" for a voice agent that answers phone calls.
- **Forgetting μ-law vs A-law.** They are both G.711, but they are **not** bit-compatible. Shipping a μ-law decoder into a European carrier edge will produce near-silence or clipped artefacts. Each region needs the right flavour.
- **Leaving Opus DTX on.** Default-on DTX (silence-suppression) makes "no packet" ambiguous: did the user stop speaking, or did the network drop? Voice-agent turn-taking depends on reliable signalling of speech-vs-silence (mod-106); do turn-taking on decoded PCM via VAD, not on packet arrival.
- **Transcoding Opus → G.711 → Opus.** A tandem through a lossy codec twice is a quality disaster. If your pipeline needs both legs, keep the Opus leg Opus-end-to-end and let only the PSTN leg be G.711 — do not transcode Opus down to G.711 and back up just to stay inside one media server.
- **Using a lossy codec for recording / eval.** Opus at any bitrate removes information. If you intend to re-score the audio against an improved ASR later, recording Opus bakes in a one-time quality ceiling. Store raw PCM or FLAC for eval (mod-112).
- **Claiming MOS numbers.** "Opus at 32 kbps beats G.711" is only meaningful with a test method (MUSHRA, ITU-T P.863) and a dataset. Vendor marketing MOS without method is not an official source.
- **Picking frame size without thinking about jitter.** Opus at 60 ms frames costs 3× the algorithmic delay of 20 ms and is harder on the jitter buffer; 10 ms is halfway to 2× the packet rate on the wire for a small quality delta. 20 ms is the safe default (RFC 7587 recommends 20 ms PTime).

## 6. References

Primary — official specs:

- **IETF RFC 6716** — "Definition of the Opus Audio Codec." — https://datatracker.ietf.org/doc/html/rfc6716 . Sample rates: §2.1.1. Frame sizes: §2.1.4. In-band FEC / PLC: §2.1.7. DTX / CN: §2.1.9.
- **IETF RFC 7587** — "RTP Payload Format for the Opus Speech and Audio Codec." — https://datatracker.ietf.org/doc/html/rfc7587 . 20 ms PTime default, SDP parameters.
- **IETF RFC 7874** — "WebRTC Audio Codec and Processing Requirements." — https://datatracker.ietf.org/doc/html/rfc7874 . Mandates Opus + G.711 μ-law / A-law support.
- **ITU-T Recommendation G.711** — "Pulse Code Modulation (PCM) of Voice Frequencies" (1988). — https://www.itu.int/rec/T-REC-G.711/ . Defines μ-law (NA / JP) and A-law (EU) companding at 8 kHz / 64 kbps.
- **IETF RFC 3551** — "RTP Profile for Audio and Video Conferences with Minimal Control." — https://datatracker.ietf.org/doc/html/rfc3551 . G.711 payload types 0 (PCMU) and 8 (PCMA).

Practitioner references (treat as implementation examples only, per source policy):

- **Twilio Media Streams docs** — https://www.twilio.com/docs/voice/twiml/stream . Confirms 8 kHz μ-law on carrier leg.
- **LiveKit Agents / OpenAI Realtime** — realtime pipeline docs that illustrate Opus-at-the-model-edge in production.
- **VeriSwarm practitioner examples** — implementation reference only.

Local curriculum anchors:

- `mod-107-telephony-webrtc-transport` — details the transcode boundary at the media-server edge.
- `mod-106-vad-turn-taking-barge-in` — why DTX-off is the voice-agent default.
- `mod-109-latency-budget-architecture` — where each transcode costs in the end-to-end budget.
- `mod-112-voice-multimodal-eval-and-observability` — why lossy-codec recording poisons eval.

Source policy note. OWASP ML Top 10, MITRE ATLAS, and NIST AI RMF are not load-bearing for codec selection; this is a transport / interop decision anchored in IETF and ITU-T specs.
