# exercise-01 — OpenAI Realtime API: Session Lifecycle Walkthrough — Reference Solution

## 1. Solution overview

The OpenAI Realtime API exposes a **speech-native** model over WebSocket (server-to-server) or WebRTC (browser/mobile). Instead of chaining STT → LLM → TTS, one process handles audio-in → reasoning → audio-out with a single bidirectional connection and an event-based protocol. The lifecycle you are expected to be able to narrate in production terms:

1. **Authenticate + connect** (`wss://api.openai.com/v1/realtime?model=…` with `OpenAI-Beta: realtime=v1`, or exchange an ephemeral token for WebRTC).
2. **Configure session** (`session.update`): modalities, voice, turn-detection mode, input/output audio formats, tools, instructions.
3. **Stream audio in** (`input_audio_buffer.append`) and let server-side VAD (or manual `commit`) detect end-of-turn.
4. **Receive model output** as a stream of `response.*` events — interleaved text transcript deltas and audio deltas — ending with `response.done`.
5. **Handle interruption / barge-in** — when the user speaks while the assistant is talking, cancel the in-flight response and truncate the already-played audio on the server's conversation state.
6. **Call tools** via `response.function_call_arguments.delta` / `.done`, respond with `conversation.item.create` (type `function_call_output`), then `response.create` to continue.
7. **Teardown**: close the socket; capture token-usage and any `error` events for logging.

This solution gives a worked walkthrough — event-by-event — plus a minimal Python asyncio reproducer, rubric, and common-mistakes list. All API-specific values (event names, parameter names, model IDs, sample rates) come from the current OpenAI Realtime documentation and carry `<!-- verify-at-runtime: ... -->` markers where OpenAI is still iterating on the surface.

<!-- verify-at-runtime: all event-name, model-ID, and parameter-name claims below should be re-checked against https://platform.openai.com/docs/guides/realtime and https://platform.openai.com/docs/api-reference/realtime before quoting externally; the beta protocol has changed shape several times. -->

## 2. Implementation

### 2.1 Transport choice — WebSocket vs WebRTC

| Transport | When it fits | How it is authenticated |
|---|---|---|
| **WebSocket** (server-to-server) | Your server is the audio owner (telephony, LiveKit worker, Pipecat worker). You already have the raw PCM and want direct control over the protocol. | Standard API key in `Authorization: Bearer …` header, plus `OpenAI-Beta: realtime=v1`. Never ship this to a browser. |
| **WebRTC** (browser/mobile) | The browser or mobile device captures the mic and plays the speaker directly; your server only mints the session. | Server calls the `/v1/realtime/sessions` endpoint to mint an **ephemeral** client token; browser connects WebRTC with that ephemeral token in the SDP offer flow. The permanent API key never leaves your server. |

Both transports speak the same **event protocol**. Choice of transport affects audio-path latency (WebRTC adds its own jitter buffer but removes a custom media path your service has to maintain) and where the first audio frame is encoded. The *lifecycle below is identical* for both; only the first/last step differ (TCP WebSocket handshake vs WebRTC SDP offer/answer).

<!-- verify-at-runtime: ephemeral-token endpoint path and the exact handshake for WebRTC; both have evolved during the beta. -->

### 2.2 The seven-step lifecycle

#### Step 1 — Connect and authenticate (WebSocket path)

```
Request:
  URL:    wss://api.openai.com/v1/realtime?model=gpt-4o-realtime-preview-2024-12-17
  Header: Authorization: Bearer <OPENAI_API_KEY>
  Header: OpenAI-Beta: realtime=v1
```

The server replies with a `session.created` event when the socket is up. This event carries the session `id`, the `model`, and the **current defaults** (voice, turn-detection, audio formats, etc.) — capture them, they are your baseline before `session.update`.

#### Step 2 — Configure the session

Send one `session.update` event. This is the single most load-bearing call. A minimal production-shaped config:

```json
{
  "type": "session.update",
  "session": {
    "modalities": ["audio", "text"],
    "voice": "alloy",
    "instructions": "You are a helpful voice agent for Acme appointment booking. Keep replies under 2 sentences unless asked for detail.",
    "input_audio_format": "pcm16",
    "output_audio_format": "pcm16",
    "input_audio_transcription": { "model": "whisper-1" },
    "turn_detection": {
      "type": "server_vad",
      "threshold": 0.5,
      "prefix_padding_ms": 300,
      "silence_duration_ms": 500,
      "create_response": true
    },
    "tools": [
      {
        "type": "function",
        "name": "book_appointment",
        "description": "Book an appointment at the given time.",
        "parameters": {
          "type": "object",
          "properties": {
            "iso_time": { "type": "string" },
            "service": { "type": "string" }
          },
          "required": ["iso_time"]
        }
      }
    ],
    "tool_choice": "auto",
    "temperature": 0.7,
    "max_response_output_tokens": 400
  }
}
```

Field-by-field, in the order a voice-agent engineer should think about them:

- **`modalities`** — `["audio", "text"]` gives you both streamed audio and a synchronous text transcript of the assistant's own output. Useful for logging, redaction, and dual-sinking to both a speaker and a UI caption track. Omitting `"audio"` makes the model text-only.
- **`voice`** — one of the model-supported voices (e.g., `alloy`, `verse`, `aria` — check the current voice catalogue). Cannot be changed mid-response for the current response. <!-- verify-at-runtime -->
- **`instructions`** — system prompt. In a voice loop, keep it tight: voice models *follow length instructions more literally than text models* and tend to produce long, chatty audio if the prompt is long.
- **`input_audio_format` / `output_audio_format`** — `pcm16` (16 kHz, 16-bit, little-endian, mono), `g711_ulaw`, or `g711_alaw`. Telephony integrations go µ-law/A-law 8 kHz; LiveKit / WebRTC paths usually go `pcm16`. <!-- verify-at-runtime: current list -->
- **`input_audio_transcription`** — turns on a separate Whisper-based transcription of the *user*'s audio, delivered as `conversation.item.input_audio_transcription.completed` events. Required if you want to log what the user said; the model's own reasoning does not surface a verbatim user transcript to you by default.
- **`turn_detection`** — server-side VAD. `server_vad` is the default voice-agent choice: the server decides when the user stopped talking and triggers a response automatically (`create_response: true`). Set `turn_detection: null` to go manual — you then emit `input_audio_buffer.commit` and `response.create` yourself. The `semantic_vad` option (where available) uses the model itself to decide end-of-turn and is better for longer thinking pauses. <!-- verify-at-runtime: semantic_vad availability -->
- **`tools` / `tool_choice`** — same JSON-schema shape as the Chat Completions API. In a voice loop, tool-call latency is a user-visible thing; keep argument schemas small.
- **`temperature`**, **`max_response_output_tokens`** — cap the model's chattiness. Short replies are the cheapest latency win in voice.

The server responds with `session.updated`. If a field is invalid (bad voice name, malformed tool schema) you get an `error` event instead — do not assume `session.updated` on blind faith.

#### Step 3 — Stream audio in

Mic audio is base64-encoded and appended in frames. A typical frame size is 20–100 ms of audio at your chosen sample rate; pick a size that matches your mic buffer size.

```
{ "type": "input_audio_buffer.append", "audio": "<base64 pcm16>" }   ← repeat continuously
```

With `server_vad`, the server emits:

- `input_audio_buffer.speech_started` — the server-side VAD saw the user start speaking.
- `input_audio_buffer.speech_stopped` — VAD saw the user stop.
- `input_audio_buffer.committed` — buffer is closed and queued for the model.
- (Automatic) `response.create` fires and the response begins.

With manual turn detection, your code is responsible for `input_audio_buffer.commit` and `response.create`.

#### Step 4 — Receive and render the response

An assistant turn is a stream of events; a production client needs to handle at minimum:

| Event | Purpose |
|---|---|
| `response.created` | Response begins. Capture `response.id`. |
| `response.output_item.added` | A new output item (message, function_call). |
| `response.content_part.added` | A content part inside that item (audio, text). |
| `response.audio.delta` | Base64-encoded audio chunk — decode and queue for playback. |
| `response.audio_transcript.delta` | Running text transcript of the assistant's own audio. |
| `response.audio.done` | No more audio for this content part. |
| `response.audio_transcript.done` | Final transcript text for this content part. |
| `response.function_call_arguments.delta` | Streamed JSON args for a tool call. |
| `response.function_call_arguments.done` | Final JSON args. |
| `response.done` | Entire response is complete. Includes `usage`. |
| `error` | Something failed. Response may still be partially usable. |

For a voice client the hot path is: receive `response.audio.delta` → decode base64 → append to your output audio ring buffer / WebRTC track. `response.audio_transcript.*` goes to your caption/log sink.

#### Step 5 — Handle interruption / barge-in

If the user starts speaking while the assistant is still producing audio, the correct sequence (regardless of transport):

1. Immediately stop *local* playback of the assistant's audio. This is what the user hears as the barge-in; do not wait for the server to catch up.
2. Send `response.cancel` with the active `response.id`. The server stops producing further `response.*` events for that response.
3. Send `conversation.item.truncate` to tell the server **how much** of the assistant audio the user actually heard. The server stores the full assistant turn in the conversation; if you do not truncate, the model thinks the user heard its entire reply and will refer back to things the user never heard.

```json
{
  "type": "conversation.item.truncate",
  "item_id": "<assistant_item_id>",
  "content_index": 0,
  "audio_end_ms": 1240
}
```

`audio_end_ms` is the number of milliseconds of assistant audio actually played out to the user — measure this at your playout, not at the server. With `server_vad`, the server itself may truncate on detecting the barge-in, but relying on that silently is fragile; truncate explicitly.

#### Step 6 — Tool calling

When the model decides to call a function, you see:

```
response.output_item.added           (item with type=function_call, name, call_id)
response.function_call_arguments.delta   (streamed JSON args, N times)
response.function_call_arguments.done    (final args string)
response.done                        (the "response" ends here, from the model's POV)
```

Your code then: (a) runs the function, (b) sends the result as a new conversation item, (c) requests the next response.

```json
{
  "type": "conversation.item.create",
  "item": {
    "type": "function_call_output",
    "call_id": "<call_id>",
    "output": "{\"status\":\"ok\",\"booking_id\":\"b_123\"}"
  }
}
```

```json
{ "type": "response.create" }
```

The second response is the one that speaks the tool result to the user. Latency here is **user-visible**: a slow tool makes the voice loop feel dead. Return something fast, even if it is "one moment, I'm checking that for you," and continue the lookup in parallel if your product shape allows.

#### Step 7 — Teardown

Close the WebSocket (or disconnect WebRTC). `response.done` events carry a `usage` block with `input_tokens`, `output_tokens`, and audio-token counts — log these per-session for cost tracking. If the session died on an `error`, capture the `code` and `message`; the most common operational errors are rate-limit, invalid `session.update`, and `input_audio_buffer_commit_empty` (you tried to commit silence).

### 2.3 Minimal Python client (WebSocket, asyncio)

The code below is a working *shape*. It reads 16 kHz mono PCM16 from stdin, streams it to the API, prints transcript deltas, and writes assistant audio to stdout. It is not production hardened — it is the scaffold to attach a real mic/speaker path to.

```python
# python 3.11+;  pip install websockets
import asyncio, base64, json, os, sys
import websockets

URL = "wss://api.openai.com/v1/realtime?model=gpt-4o-realtime-preview-2024-12-17"
HEADERS = [
    ("Authorization", f"Bearer {os.environ['OPENAI_API_KEY']}"),
    ("OpenAI-Beta", "realtime=v1"),
]

SESSION_UPDATE = {
    "type": "session.update",
    "session": {
        "modalities": ["audio", "text"],
        "voice": "alloy",
        "instructions": "You are a terse voice assistant. Reply in <= 2 sentences.",
        "input_audio_format": "pcm16",
        "output_audio_format": "pcm16",
        "input_audio_transcription": {"model": "whisper-1"},
        "turn_detection": {
            "type": "server_vad",
            "threshold": 0.5,
            "prefix_padding_ms": 300,
            "silence_duration_ms": 500,
            "create_response": True,
        },
    },
}

async def pump_mic(ws):
    # read raw 16kHz PCM16 mono from stdin in ~40ms chunks (16000*2*0.04 = 1280 bytes)
    loop = asyncio.get_running_loop()
    reader = asyncio.StreamReader()
    protocol = asyncio.StreamReaderProtocol(reader)
    await loop.connect_read_pipe(lambda: protocol, sys.stdin.buffer)
    while True:
        chunk = await reader.readexactly(1280)
        if not chunk:
            break
        await ws.send(json.dumps({
            "type": "input_audio_buffer.append",
            "audio": base64.b64encode(chunk).decode(),
        }))

async def pump_events(ws):
    async for raw in ws:
        evt = json.loads(raw)
        t = evt.get("type")
        if t == "session.created":
            await ws.send(json.dumps(SESSION_UPDATE))
        elif t == "response.audio.delta":
            sys.stdout.buffer.write(base64.b64decode(evt["delta"]))
            sys.stdout.buffer.flush()
        elif t == "response.audio_transcript.delta":
            sys.stderr.write(evt["delta"]); sys.stderr.flush()
        elif t == "response.audio_transcript.done":
            sys.stderr.write("\n")
        elif t == "conversation.item.input_audio_transcription.completed":
            sys.stderr.write(f"[user] {evt['transcript']}\n")
        elif t == "input_audio_buffer.speech_started":
            # barge-in hook: your local audio sink should flush here
            pass
        elif t == "error":
            sys.stderr.write(f"[error] {evt}\n")

async def main():
    async with websockets.connect(URL, additional_headers=HEADERS) as ws:
        await asyncio.gather(pump_mic(ws), pump_events(ws))

asyncio.run(main())
```

What's deliberately missing (and what a graded submission should name explicitly):

- **Barge-in**: on `input_audio_buffer.speech_started`, cancel the current response and `conversation.item.truncate` with the measured playback position. This scaffold does neither.
- **Reconnect**: a dropped WebSocket does not restore the conversation. In production, catch the disconnect, reconnect, and (optionally) replay the last few conversation items with `conversation.item.create` to rebuild context.
- **Backpressure**: if your mic is faster than the network, `send` queues grow; cap them.
- **Audio format mismatches**: if your mic hardware delivers 44.1 kHz or 48 kHz, resample to 16 kHz *before* `append`. The server does not resample for you on this format.

### 2.4 Event-shape cheat sheet

A voice-agent engineer should be able to recite the five hot-path event groups without looking them up:

```
Client → Server (what you send)
  session.update                      — change modalities, voice, VAD, tools, …
  input_audio_buffer.append           — mic audio frames
  input_audio_buffer.commit           — (manual VAD) close the current turn
  response.create                     — (manual VAD) ask the model to reply
  response.cancel                     — cancel the in-flight response (barge-in)
  conversation.item.create            — inject a message or tool-call result
  conversation.item.truncate          — tell server what was actually heard
  conversation.item.delete            — remove a prior item (uncommon)

Server → Client (what you receive)
  session.created / session.updated   — session state
  input_audio_buffer.speech_started   — VAD onset (barge-in trigger)
  input_audio_buffer.speech_stopped   — VAD offset
  input_audio_buffer.committed        — buffer closed, response queued
  conversation.item.created           — new item in conversation state
  conversation.item.input_audio_transcription.completed — user transcript
  response.created                    — response begins
  response.output_item.added          — new output item (message, function_call)
  response.content_part.added         — content part inside item
  response.audio.delta                — audio chunk (base64)
  response.audio_transcript.delta     — assistant transcript chunk
  response.audio.done                 — audio finished
  response.audio_transcript.done      — transcript finished
  response.function_call_arguments.delta  — tool arg stream
  response.function_call_arguments.done   — tool args complete
  response.done                       — response finished (usage inside)
  rate_limits.updated                 — current rate-limit headroom
  error                               — something broke
```

<!-- verify-at-runtime: event-name list; the Realtime API is beta and has added/renamed events several times. Confirm against platform.openai.com before using the names externally. -->

## 3. Validation steps

A finished submission should pass each check.

1. **Lifecycle narration.** The seven steps in §2.2 are each named with the triggering client event and the expected server event(s). A reviewer can read the submission end-to-end and see each transition.
2. **Transport decision.** The submission says *which* transport it chose (WebSocket vs WebRTC) and *why*, tied to where the audio actually lives (server-side telephony vs browser mic).
3. **Session config correctness.** `session.update` sets at least `modalities`, `voice`, `input_audio_format`, `output_audio_format`, `turn_detection`, and `instructions`. If `tools` is present, the JSON schemas are valid.
4. **Turn-detection mode justified.** The submission picks `server_vad`, manual, or `semantic_vad` and names the user-facing reason (telephony-shape silences, long-thinking pauses, programmatic push-to-talk). Not just "default."
5. **Barge-in correctly named.** The submission explicitly describes: (a) local playout stop, (b) `response.cancel`, (c) `conversation.item.truncate` with measured `audio_end_ms`. A submission that stops only local audio but does not truncate server-side state is incomplete.
6. **Tool-call loop correctly named.** The function_call → function_call_output → `response.create` chain is explicit, including which events to watch and the fact that the tool-result turn is a *new* response.
7. **Teardown / observability.** The submission names where `usage` is captured and how `error` events are logged. Silent `error` handling fails this check.
8. **Reproducer.** A minimal client exists (a modified version of §2.3 is acceptable) that at least sends `session.update` and prints a `response.audio_transcript.*` stream for a canned audio file. A submission without *any* running code fails this check.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Lifecycle coverage | 20 % | All 7 steps named, each with client-event → server-event pair | Each step includes its common failure mode and the log signature |
| Transport decision | 10 % | WebSocket vs WebRTC chosen, with reason tied to audio ownership | Both transports described; ephemeral-token flow for WebRTC is drawn |
| Session config | 15 % | `session.update` sets the six required fields; JSON is well-formed | Each field annotated with its voice-agent rationale and the failure mode of a wrong value |
| Turn detection | 10 % | Chosen mode justified against product shape | `server_vad` parameters (threshold, padding, silence) tuned with reasoning |
| Barge-in | 15 % | Local stop + `response.cancel` + `conversation.item.truncate` with measured `audio_end_ms` | Barge-in discipline tested against a scripted interruption and `audio_end_ms` validated against local playout position |
| Tool-call loop | 10 % | function_call → function_call_output → `response.create` chain documented | Long-running-tool pattern ("one moment while I check") called out as a UX choice |
| Reproducer | 10 % | Minimal client that connects, updates session, and streams one audio response | Reproducer handles reconnect, backpressure, and barge-in |
| Observability + teardown | 10 % | `usage` logged, `error` events logged | Per-session structured log includes session id, response ids, barge-in count, and tool-call count |

Scoring < 70 % is a fail-and-redo. The most common sub-70 failures are: (a) no `conversation.item.truncate` on barge-in, and (b) no `input_audio_transcription` so the submission has no record of what the user said.

## 5. Common mistakes

- **Treating `response.cancel` as sufficient for barge-in.** Without `conversation.item.truncate`, the server still believes the user heard the full assistant reply and will reference content the user never heard. The voice agent then sounds schizophrenic ("as I mentioned, your appointment is at 4:15" when the user cut in at "your appointment is at…").
- **Playing out all buffered audio on barge-in.** The local jitter buffer may hold hundreds of ms of audio. The user perception of a responsive barge-in requires flushing *local* buffers first; only then do the server-side cancel + truncate matter.
- **Shipping the permanent API key to the browser.** The WebRTC path specifically exists so you don't do this; use the ephemeral-token endpoint.
- **Missing `OpenAI-Beta: realtime=v1` header.** The connection rejects without it. The error is often opaque to the client.
- **Assuming `modalities: ["audio"]` is enough for logging.** Without `"text"` *and* without `input_audio_transcription`, you cannot log either side of the conversation as text — only as audio, which is operationally useless.
- **Sending 8 kHz audio with `input_audio_format: "pcm16"`.** `pcm16` means 16 kHz. Telephony audio (8 kHz µ-law) needs `g711_ulaw` or a resample step.
- **Letting the server VAD decide for long-form reasoning.** Default `silence_duration_ms` (~500 ms) cuts users off mid-thought. Either lengthen it, switch to `semantic_vad` where available, or move to manual turn detection.
- **Not handling `response.function_call_arguments.delta` as a stream.** Some clients wait for `.done` and block on the full args. For slow network / long args this costs latency; you can start preparing the tool call on partial args when the schema allows.
- **Ignoring `rate_limits.updated`.** This event tells you how close you are to hitting a cap mid-session. Teams that ignore it find out at 03:00.
- **Opening a new WebSocket per turn.** The Realtime connection is session-scoped, not request-scoped — the model's conversation state lives on the server for the life of the socket. Reconnecting discards it (unless you replay conversation items yourself).
- **Trusting `session.updated` as success acknowledgement without reading its body.** The server may have silently coerced an unknown voice name to the default; always diff the echoed session state.

## 6. References

Primary — OpenAI documentation (verify at runtime; this surface is still beta):

- OpenAI Realtime API guide — `https://platform.openai.com/docs/guides/realtime`
- OpenAI Realtime API reference — `https://platform.openai.com/docs/api-reference/realtime`
- OpenAI Realtime WebRTC guide — `https://platform.openai.com/docs/guides/realtime-webrtc` <!-- verify-at-runtime: URL may have been restructured -->
- OpenAI model index (Realtime-capable models) — `https://platform.openai.com/docs/models`
- OpenAI Realtime console (example client source) — `https://github.com/openai/openai-realtime-console`
- OpenAI Realtime agents / twilio reference — `https://github.com/openai/openai-realtime-agents` <!-- verify-at-runtime: repo name/URL -->

Related exercises inside this module:

- exercise-02 (Gemini Live) — same lifecycle shape with a different protocol dialect; the comparison is the point.
- exercise-03 (Claude via realtime-relay) — the pipeline-side counterpart when the LLM lacks a native speech-in/speech-out surface.
- exercise-04 (pipeline vs speech-native) — the decision this walkthrough fuels.
- exercise-05 (half vs full duplex) — explains why "barge-in" in OpenAI Realtime is still half-duplex, not what Moshi does.

Standards referenced indirectly (used by the compliance / observability framing):

- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — the "measure, govern, document" discipline behind the `usage` and `error` logging stance.
- OWASP Machine Learning Security Top 10 — `https://owasp.org/www-project-machine-learning-security-top-10/` — relevant when the Realtime channel is deliberately attacked (prompt-injection via spoken audio, forced tool-call exfiltration).
- MITRE ATLAS — `https://atlas.mitre.org/` — audio-adversarial threat model reference for the barge-in/interruption channel.

Source policy note. All event names, endpoint paths, and model IDs in this document reflect the OpenAI Realtime documentation at the time the solution was written and carry `<!-- verify-at-runtime: ... -->` markers because the surface is still beta. Do not quote event names or model IDs downstream without re-verifying against the current `platform.openai.com/docs` pages.
