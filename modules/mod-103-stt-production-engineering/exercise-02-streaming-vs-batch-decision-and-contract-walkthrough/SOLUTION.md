# exercise-02 — Streaming vs. Batch Decision + Streaming-Contract Walkthrough — Reference Solution

## 1. Solution overview

This is a two-part exercise:

- **Part A — Decision.** Decide, per route, whether the ASR path should be streaming or batch, and defend the decision against both the latency budget and the quality budget.
- **Part B — Streaming-contract walkthrough.** Walk through the event shape a streaming ASR endpoint actually emits: partials, finals, speech-start / speech-end, endpointing, confidence, word-timings. Pick one provider and show exactly how your pipeline subscribes to and reacts to each event type.

A mature submission does *not* answer "streaming is always better." It commits, per route, and names the trigger event ("end of final transcript" vs "VAD end-of-speech + 300 ms stability window") that starts the LLM.

The worked example below uses **Deepgram's streaming API** as the walkthrough provider because its event shape is public and well-documented, and because it has the richest event surface of the current commercial streaming ASRs. The same pattern applies to AssemblyAI Universal-Streaming, OpenAI Realtime, Google Streaming Recognize, and Azure continuous-recognition — the field names differ, the *shape* doesn't.

Any vendor-specific numeric claim below is marked `<!-- verify-at-runtime: confirm at exercise-run time -->` when it's the kind of number that moves.

## 2. Implementation

### 2.1 Part A — streaming vs batch decision

The decision is made against four forces: latency, accuracy, cost, and operational complexity.

| Force | What pushes toward streaming | What pushes toward batch |
|---|---|---|
| Latency | Any route where "the user is on the line waiting" — voice agent, live captioning, live coaching | Any route where results are consumed asynchronously — post-call summarisation, call-recording analytics, transcription-for-search |
| Accuracy | Streaming models are *generally* on-par or slightly behind batch on WER, because batch gets more right-context. In most provider roadmaps the gap is small on English but non-trivial on multilingual. <!-- verify-at-runtime: confirm current WER-gap claims per provider --> | Batch wins when the audio is rich (podcast, multi-speaker long-form), when you can tolerate waiting, and when you want model-level features (diarisation refinement, speaker recognition) that are higher-quality in batch |
| Cost | Streaming is *usually* priced at a premium to batch (per-second billing, lower utilisation of the vendor's GPU pool). <!-- verify-at-runtime --> Short utterances inflate per-call cost. | Batch typically cheaper per minute; fewer connection-level costs; big-audio economies. |
| Operational complexity | Streaming carries a WebSocket (or bidi-gRPC) lifecycle: connect, keepalive, mid-stream errors, mid-stream reconnects, half-close semantics. Your code grows an audio buffer, a reconnect strategy, and a state machine. | Batch is one HTTP call. Operationally boring; retry is trivially idempotent. |

**Decision rule.** Streaming if *and only if* either:

1. The route is in the user-waiting path (voice agent, live caption, live coaching), **or**
2. The product explicitly promises "live" transcript to a human who will act on it inside the conversation.

Otherwise, batch. "We might want live someday" is not a reason to pay the streaming tax today.

**Worked routes.**

| Route | Decision | Why |
|---|---|---|
| Inbound customer-service voice agent | **Streaming** | User-waiting. The LLM-trigger event is the final transcript of the user's utterance; VAD + endpointing is the backup if finals are slow. |
| Outbound appointment reminder | **Streaming** | User-waiting. Same reasoning as inbound. |
| Post-call CSAT summary and action-item extraction | **Batch** | Not user-waiting; runs once when the call ends. Batch gives the richer transcript the LLM will summarise more cleanly. |
| Trust-and-safety queue of flagged calls for human review | **Batch** | Analyst reads the transcript afterwards. Batch + diarisation + redaction, which is a feature set batch APIs tend to lead on. |
| Live coaching whisper to a human sales rep during the call | **Streaming** | Rep-waiting. Insertion-latency budget is tight; the whisper is useless if it arrives after the moment passed. |
| Meeting notes for an internal meeting-notes product | **Depends** | Streaming if live-captions are the primary feature; batch if post-meeting summary is primary. Split the product's feature set across the two routes rather than forcing one. |

Note that voice agents sometimes run batch *alongside* streaming — the streaming path drives the live loop, and a batch pass runs at call-end against the recording to produce a higher-fidelity transcript for post-call analytics and eval. This is a common pattern; a mature answer calls it out.

### 2.2 Part B — streaming-contract walkthrough (Deepgram as the worked provider)

Deepgram's streaming API transmits audio frames over a WebSocket (`wss://api.deepgram.com/v1/listen?...`) and emits JSON messages back. A representative message surface:

```json
{
  "type": "Results",
  "channel_index": [0, 1],
  "duration": 1.02,
  "start": 3.14,
  "is_final": false,
  "speech_final": false,
  "channel": {
    "alternatives": [
      {
        "transcript": "i'd like to change my appointment",
        "confidence": 0.86,
        "words": [
          {"word": "i'd",         "start": 3.14, "end": 3.26, "confidence": 0.93},
          {"word": "like",        "start": 3.26, "end": 3.41, "confidence": 0.91},
          {"word": "to",          "start": 3.41, "end": 3.50, "confidence": 0.88},
          {"word": "change",      "start": 3.50, "end": 3.74, "confidence": 0.84},
          {"word": "my",          "start": 3.74, "end": 3.86, "confidence": 0.89},
          {"word": "appointment", "start": 3.86, "end": 4.16, "confidence": 0.82}
        ]
      }
    ]
  }
}
```

(Shape above is the documented Deepgram message; see `https://developers.deepgram.com/docs/live-streaming-audio` and `https://developers.deepgram.com/reference/listen-live` for the current field list. Confirm exact fields at exercise-run time.)

The four event classes you care about in the pipeline:

1. **Interim partial** — `is_final: false`, `speech_final: false`. The best current hypothesis; expected to be revised. **Rule:** display to the user (if you are showing captions), *do not* send to the LLM.
2. **Final** — `is_final: true`, `speech_final: false`. The ASR has committed to the words so far within the current speech segment. **Rule:** safe to show; can be used to accumulate what will become the user turn.
3. **Speech-final** — `is_final: true`, `speech_final: true`. The ASR believes the user has finished speaking (endpointing). **Rule:** the canonical "start the LLM" trigger for most voice agents.
4. **Connection events** — `Metadata`, `UtteranceEnd`, errors, close frames. **Rule:** route to your logging and your reconnect state machine; do not swallow.

**LLM-trigger pattern (the decision at the heart of the exercise):**

```python
# Worked pipeline — illustrative; real code will use the SDK's event hooks
accumulated_turn = []

async def on_message(msg: dict) -> None:
    results = msg.get("channel", {}).get("alternatives", [])
    if not results:
        return
    transcript = results[0].get("transcript", "")
    confidence = results[0].get("confidence")

    if msg.get("speech_final"):
        accumulated_turn.append(transcript)
        user_turn = " ".join(t for t in accumulated_turn if t).strip()
        accumulated_turn.clear()
        await start_llm_turn(user_turn, asr_confidence=confidence)
    elif msg.get("is_final"):
        accumulated_turn.append(transcript)
        # No LLM yet — we're accumulating within a speech segment.
    else:
        # Interim — update the UI if you have one, do not call the LLM.
        await update_live_caption(transcript)
```

**Why the trigger is `speech_final` and not "the first final"**: a single user utterance frequently produces multiple `is_final` messages — the ASR commits to chunks as the user keeps speaking. If you start the LLM on the first `is_final`, you'll fire the LLM mid-utterance, which (a) wastes a round-trip because the user will keep talking, (b) risks starting the TTS before the user has finished, and (c) makes barge-in ambiguous. `speech_final` collapses the ambiguity.

**Why you still need VAD-based backup endpointing**: if the ASR's endpointer is slow (or the user says "um" and then the long silence), you can hit the user-tolerance ceiling waiting for `speech_final`. Belt-and-braces: fire the LLM when **either** `speech_final` arrives **or** the VAD has seen > N ms of silence (typical: 400–800 ms, tuned per route) *and* the ASR has emitted at least one final. If both triggers fire, the `speech_final` one wins (richer signal).

**Mid-stream error handling.**

| Event | Correct reaction |
|---|---|
| WebSocket closes unexpectedly mid-utterance | Open a new socket immediately; replay the last ~300–500 ms of audio (your ring buffer, see mod-102). The ASR will relabel; dedup against the last final you kept. |
| Keepalive ping missed / server close with code 1011 | Treat as transient; reconnect; log. If three reconnects in 30 s, escalate (page or degrade to batch per call). |
| Server returns a final with confidence < threshold (e.g., 0.4) | Keep the hypothesis but mark it as low-confidence in the turn envelope passed to the LLM. See exercise-03 for the full confidence-informed handoff. |
| Audio-side clock drift / jitter | Your sending loop should pace at wallclock, not at `audio_length / sample_rate` — otherwise you'll either outrun or lag the vendor's buffers. |

**Backpressure.**

Streaming ASRs can and will slow down under load. Your sending loop MUST NOT block waiting for the server to catch up — it must either drop old audio (bad for a voice agent, good enough for live captions) or buffer with a bounded queue (good for a voice agent, requires a backpressure signal to the upstream). Pick explicitly; don't let Python's socket defaults pick for you.

### 2.3 How other providers map to the same contract

The same four event classes exist across providers under different names. If you port the pipeline, you port the mapping, not the logic.

| Event class | Deepgram | AssemblyAI Universal-Streaming | Google Streaming Recognize | Azure continuous recognition | OpenAI Realtime |
|---|---|---|---|---|---|
| Interim partial | `is_final:false` | `PartialTranscript` (non-final) | `is_final:false` | `recognizing` event | `conversation.item.input_audio_transcription.delta` |
| Final within turn | `is_final:true, speech_final:false` | `FinalTranscript` | `is_final:true` | `recognized` event | `conversation.item.input_audio_transcription.completed` (per audio input) |
| End of turn / speech-final | `speech_final:true` / `UtteranceEnd` | `SessionTerminated` / endpointing event | `single_utterance:true` behaviour | `recognized` + silence timeout | `input_audio_buffer.speech_stopped` |
| Error / close | WebSocket close code | Session closed | gRPC trailer | Cancellation event | WebSocket error / response.error |

<!-- verify-at-runtime: confirm each provider's exact event names at exercise-run time; the shapes above are drawn from docs at the time this reference was written and move. -->

### 2.4 Validation measurements to collect

For each shortlisted provider on each shortlisted route, collect four numbers on the same test clip set (20+ clips, mixed clean and noisy, mixed short and long utterances):

1. **Partial-latency P50 / P95** — from audio-frame-sent to first partial containing a non-empty transcript.
2. **Final-latency P50 / P95** — from end-of-audio for an utterance to the first `is_final` containing that utterance.
3. **Speech-final-latency P50 / P95** — from end-of-audio to the provider's end-of-utterance signal.
4. **Reconnect-penalty** — forcibly close a socket mid-utterance; measure how long until the new socket re-emits the lost words.

Report numbers with a sample size and a confidence interval, not as point values.

## 3. Validation steps

A finished submission should pass each of the following checks.

1. **Per-route decision table.** At least four routes, each decided (streaming / batch / split), each with the decision tied to a specific force from §2.1.
2. **Walkthrough for one provider.** The Deepgram-shape walkthrough (or the equivalent for your chosen provider) names all four event classes, with a worked pipeline snippet showing which event fires the LLM.
3. **Trigger argument.** The submission explicitly defends `speech_final` (or the provider's equivalent) over "the first final" as the LLM trigger — the reviewer should see the reason, not just the code.
4. **VAD-backup trigger.** The submission has a belt-and-braces condition for the case where `speech_final` is slow or missing.
5. **Mid-stream error table.** All four failure modes (unexpected close, server error, low-confidence final, jitter) are addressed.
6. **Backpressure stance.** The submission picks drop-vs-buffer explicitly, and justifies the pick against the route.
7. **Measurements table.** For each shortlisted provider, partial / final / speech-final latencies and reconnect penalty are reported with sample size.
8. **Cross-provider mapping.** At least one other provider has its event-shape mapped against the walkthrough provider's event shape, so the pipeline can port.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Decision per route | 15 % | 4+ routes, each with a reason | Each reason names the user-waiting / not-user-waiting test explicitly, and the split-route case is handled |
| Event-class coverage | 20 % | All four event classes named and distinguished | Each event class mapped to its purpose in the pipeline (UI update / accumulation / LLM-trigger / log-only) |
| LLM-trigger argument | 15 % | `speech_final`-style trigger chosen with a defensible reason | The "why not the first final" argument is given and tested against a worked multi-final utterance |
| VAD-backup trigger | 10 % | Backup trigger present | Threshold values named and tied to the user-tolerance ceiling for the route |
| Error / backpressure handling | 15 % | All four failure modes handled; backpressure stance explicit | Reconnect strategy is idempotent against the last final; backpressure stance measured, not asserted |
| Cross-provider mapping | 10 % | At least two providers mapped against the walkthrough | All five shortlisted providers mapped, with the event-shape differences called out |
| Measurements | 15 % | Partial / final / speech-final latencies for ≥ 2 providers | Measurements posted with sample size, P50/P95/P99, and reconnect-penalty |

Scoring < 70 % is a fail-and-redo. The most common sub-70 failure: an answer that says "streaming is better" without ever naming the trigger event.

## 5. Common mistakes

- **Firing the LLM on the first `is_final`.** Produces mid-utterance LLM triggers; see §2.2 for the correction.
- **Forgetting VAD backup.** Relying solely on the ASR endpointer leaves you blocked when it stalls.
- **Treating partials as if they were finals.** Showing partials in captions is fine; sending partials to the LLM wastes tokens and, worse, confuses the model because the partials will be revised. Don't.
- **Streaming on batch routes.** Paying the streaming premium on a post-call summary pipeline is pure cost burn. Batch the batch routes.
- **Reconnect that replays the whole call.** On reconnect, replay the last 300–500 ms of audio, not the whole audio buffer. Replaying more than that causes duplicate finals.
- **Backpressure that silently drops.** Dropping audio frames is sometimes correct (live captions), but silent drops in a voice-agent path produce "the bot missed what I said" complaints that the pipeline can't explain. Log drops, and bias toward bounded buffering with escalation.
- **No reconnect-penalty measurement.** The first real production incident will be a mid-call disconnect. Measure the penalty before you ship.
- **Per-vendor event names baked in.** If your pipeline hard-codes `speech_final`, porting to AssemblyAI / Google / Azure / OpenAI requires a rewrite, not a plugin swap. Keep the event-class abstraction (§2.3) between your pipeline and the SDK.
- **Streaming to log back-end on the hot path.** Wire your event-shape logging to an async non-blocking sink. A slow log sink will stall the audio-send loop and tank TTFT.

## 6. References

Primary — provider streaming docs (verify URLs still resolve at exercise-run time):

- Deepgram live streaming — `https://developers.deepgram.com/docs/live-streaming-audio`
- Deepgram listen-live reference — `https://developers.deepgram.com/reference/listen-live`
- Deepgram endpointing & interim-results behaviour — `https://developers.deepgram.com/docs/understanding-endpointing-interim-results`
- AssemblyAI Streaming (Universal-Streaming) — `https://www.assemblyai.com/docs/speech-to-text/streaming`
- Google Cloud Speech-to-Text streaming recognition — `https://cloud.google.com/speech-to-text/docs/basics#streaming_recognition`
- Azure Speech Service streaming / continuous recognition — `https://learn.microsoft.com/azure/ai-services/speech-service/get-started-speech-to-text`
- OpenAI Realtime API — `https://platform.openai.com/docs/guides/realtime`
- WebSocket RFC 6455 (behaviour on close codes, keepalives) — `https://www.rfc-editor.org/rfc/rfc6455`

Open-source orchestration frameworks (where the event-class abstraction is implemented in the real world):

- LiveKit Agents — `https://docs.livekit.io/agents/`
- Pipecat — `https://github.com/pipecat-ai/pipecat`
- Vocode — `https://docs.vocode.dev/`

Related exercises:

- exercise-01 (STT provider decision matrix) — feeds the per-provider shortlist.
- exercise-03 (confidence-informed handoff) — consumes the final + confidence envelope this exercise produces.
- exercise-05 (WER benchmark) — measures the accuracy side of the streaming-vs-batch trade-off on real audio.

Standards referenced indirectly:

- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — relevant to the "log what you dropped" discipline under backpressure.
- OWASP ML Top 10 — `https://owasp.org/www-project-machine-learning-security-top-10/` — relevant when the error-handling table interacts with injected/adversarial audio.
- MITRE ATLAS — `https://atlas.mitre.org/` — audio-adversarial threat-model reference.

Source policy note. The walkthrough above uses Deepgram's documented event shape as the worked example because it is the most publicly documented of the current commercial streaming ASRs. The cross-provider mapping in §2.3 names event-fields drawn from docs at the time this reference was written; vendor-specific event names and shapes change and every row in §2.3 should be re-verified against the vendor's own current docs at exercise-run time.
