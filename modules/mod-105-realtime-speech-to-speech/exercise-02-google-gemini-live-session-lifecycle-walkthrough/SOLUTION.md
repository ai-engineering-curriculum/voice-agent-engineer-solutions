# exercise-02 — Google Gemini Live: Session Lifecycle Walkthrough — Reference Solution

## 1. Solution overview

Google's **Gemini Live API** (the `BidiGenerateContent` WebSocket method, exposed through the `google-genai` SDK as `client.aio.live.connect(...)`) is Google's speech-native counterpart to OpenAI's Realtime API. One bidirectional WebSocket carries setup, streamed audio-in, streamed audio-out, text, tool calls, and server-side VAD events. The shape is the same as OpenAI Realtime; the dialect is different. The lifecycle you should be able to narrate:

1. **Pick a Live-capable model** (e.g., `gemini-2.0-flash-live-001` on the Gemini API, or `gemini-2.0-flash-live-preview-04-09` / similar on Vertex AI — model IDs evolve rapidly).
2. **Open the WebSocket** (Gemini API uses an API-key query param; Vertex AI uses Google auth with a service identity).
3. **Send `setup`** (model, generation config, system instruction, tools, voice/response-modalities, VAD config). The server replies `setupComplete`.
4. **Stream audio in** via `realtimeInput` messages (base64 PCM, 16 kHz mono by default). Server-side VAD decides end-of-utterance, or you flip to `activityStart` / `activityEnd` for manual control.
5. **Receive output** as `serverContent` messages (interleaved audio parts and text parts), ending each model turn with `turnComplete`.
6. **Handle barge-in / interruption** via the `interrupted` flag in `serverContent` — the server detects user speech during assistant playout and signals it.
7. **Call tools** via `toolCall` messages → run the function → return `toolResponse`. Model continues producing audio.
8. **Teardown** — close the socket; capture `usageMetadata` and any `goAway` (the server telling you to reconnect soon).

This solution walks the lifecycle, gives a minimal Python reproducer using the official `google-genai` SDK, contrasts the protocol against OpenAI Realtime from exercise-01, and gives rubric + common-mistakes. All model IDs, field names, and limits carry `<!-- verify-at-runtime: ... -->` markers because the Live API is actively iterating.

<!-- verify-at-runtime: all field names, model IDs, limits, and sample rates in this document should be re-pulled from https://ai.google.dev/gemini-api/docs/live and https://cloud.google.com/vertex-ai/generative-ai/docs/live-api before quoting externally. -->

## 2. Implementation

### 2.1 Where Live lives — two surfaces

| Surface | SDK path | Auth | When to pick |
|---|---|---|---|
| **Gemini Developer API** (`generativelanguage.googleapis.com`) | `google.genai.Client(api_key=...)` → `client.aio.live.connect(...)` | API key (query param on the WebSocket) | Prototypes, individual-developer apps, lightweight prod. |
| **Vertex AI** (`<region>-aiplatform.googleapis.com`) | `google.genai.Client(vertexai=True, project=..., location=...)` → same `.aio.live.connect(...)` | Google Cloud auth (ADC, service account, workload identity) | Enterprise: IAM, VPC-SC, regional residency, logging to Cloud Logging. |

The protocol over the wire is the same `BidiGenerateContent` method; the SDK hides the auth difference. Most production voice agents pick Vertex AI once they need IAM boundaries; rapid-iteration code lives on the Developer API.

<!-- verify-at-runtime: both endpoints and Live model availability per region. -->

### 2.2 The eight-step lifecycle

#### Step 1 — Pick a model

Not every Gemini model supports Live. The Live-capable "flash" family is the voice-agent default because of latency and cost; "live" model names change (e.g., `gemini-2.0-flash-live-001`, `gemini-2.5-flash-preview-native-audio-dialog`). Native-audio variants perform the audio output directly; half-cascade variants ASR→LLM→TTS internally. Native audio is lower-latency and captures affect; half-cascade is more predictable for text-heavy use cases. <!-- verify-at-runtime: current model catalog and the native-vs-cascade distinction -->

#### Step 2 — Open the WebSocket

With the SDK:

```python
from google import genai
from google.genai import types

client = genai.Client(api_key=os.environ["GEMINI_API_KEY"])  # or vertexai=True, project=..., location=...
config = types.LiveConnectConfig( ... )                       # see Step 3
async with client.aio.live.connect(model="gemini-2.0-flash-live-001", config=config) as session:
    ...
```

Under the hood the SDK opens `wss://generativelanguage.googleapis.com/ws/google.ai.generativelanguage.v1beta.GenerativeService.BidiGenerateContent?key=<API_KEY>` (Developer API) or the equivalent Vertex URL, then sends the first `setup` frame for you based on the `LiveConnectConfig`. Raw-socket clients can skip the SDK and send the `setup` message themselves.

<!-- verify-at-runtime: exact WebSocket URL path -->

#### Step 3 — `setup` (the single config call)

Everything that shapes the session lives in `setup`. The minimal production-shaped config:

```python
config = types.LiveConnectConfig(
    response_modalities=["AUDIO"],  # or ["TEXT"]; most Live models accept exactly one modality per connection
    system_instruction=types.Content(
        parts=[types.Part(text="You are a terse voice agent. Reply in <= 2 sentences.")]
    ),
    speech_config=types.SpeechConfig(
        voice_config=types.VoiceConfig(
            prebuilt_voice_config=types.PrebuiltVoiceConfig(voice_name="Puck"),
        ),
        language_code="en-US",
    ),
    generation_config=types.GenerationConfig(temperature=0.7, max_output_tokens=400),
    realtime_input_config=types.RealtimeInputConfig(
        automatic_activity_detection=types.AutomaticActivityDetection(
            disabled=False,
            start_of_speech_sensitivity=types.StartSensitivity.START_SENSITIVITY_LOW,
            end_of_speech_sensitivity=types.EndSensitivity.END_SENSITIVITY_LOW,
            prefix_padding_ms=20,
            silence_duration_ms=500,
        ),
        activity_handling=types.ActivityHandling.START_OF_ACTIVITY_INTERRUPTS,
        turn_coverage=types.TurnCoverage.TURN_INCLUDES_ONLY_ACTIVITY,
    ),
    tools=[types.Tool(function_declarations=[types.FunctionDeclaration(
        name="book_appointment",
        description="Book an appointment at the given ISO time.",
        parameters=types.Schema(
            type=types.Type.OBJECT,
            properties={
                "iso_time": types.Schema(type=types.Type.STRING),
                "service":  types.Schema(type=types.Type.STRING),
            },
            required=["iso_time"],
        ),
    )])],
)
```

Field-by-field:

- **`response_modalities`** — Live models typically accept **exactly one** output modality per connection. Pick `"AUDIO"` for a voice agent; pick `"TEXT"` for a captioning-only session. Trying to request both on a single connection is a common first-time error. <!-- verify-at-runtime -->
- **`system_instruction`** — model-level persona; applied to every turn in this session.
- **`speech_config`** — selects the output voice (`Puck`, `Charon`, `Kore`, `Fenrir`, `Aoede`, etc. — catalog changes) and the language. <!-- verify-at-runtime: current voice list -->
- **`generation_config`** — temperature, max output tokens, etc. Max tokens caps *assistant audio length* indirectly; tune it.
- **`realtime_input_config.automatic_activity_detection`** — server-side VAD. Setting `disabled=True` flips to manual: you then send `activity_start` / `activity_end` yourself.
- **`activity_handling`** — `START_OF_ACTIVITY_INTERRUPTS` means user speech while the model is speaking aborts the model turn (barge-in). `NO_INTERRUPTION` disables that.
- **`turn_coverage`** — whether the model sees non-activity audio as part of its input. `TURN_INCLUDES_ONLY_ACTIVITY` is the voice-agent default.
- **`tools`** — function declarations (same JSON-schema shape as `generateContent`). Also: Google Search, code execution — Live supports those as built-in tools.

The server responds with a `setupComplete` message. If `setup` is invalid, the socket closes with an error status. Capture `setupComplete` as your "we're live" signal; do not start streaming mic audio before it.

#### Step 4 — Stream audio in

Gemini Live expects **16-bit PCM, 16 kHz, mono, little-endian** as the default input format. You send it in `realtimeInput` messages:

```python
await session.send_realtime_input(
    audio=types.Blob(data=pcm16_bytes, mime_type="audio/pcm;rate=16000")
)
```

With VAD on, the server detects start/end and will emit `serverContent` events with the user's transcript (if you asked for input transcription) and start producing the model turn. With VAD off, you wrap each turn in `send_realtime_input(activity_start=...)` and `send_realtime_input(activity_end=...)`.

#### Step 5 — Receive output

Each assistant turn is a stream of `serverContent` messages. In the SDK, iterate the `session.receive()` async generator:

```python
async for msg in session.receive():
    if msg.server_content:
        # audio parts
        if msg.server_content.model_turn:
            for part in msg.server_content.model_turn.parts:
                if part.inline_data:
                    play_pcm(part.inline_data.data)  # 24 kHz PCM16 out
                elif part.text:
                    log_caption(part.text)
        # user-side transcription (if enabled)
        if msg.server_content.input_transcription:
            log_user(msg.server_content.input_transcription.text)
        # model-side transcription
        if msg.server_content.output_transcription:
            log_assistant(msg.server_content.output_transcription.text)
        # interrupted flag (barge-in)
        if msg.server_content.interrupted:
            flush_local_audio_buffer()
        # end of turn
        if msg.server_content.turn_complete:
            on_turn_complete()
    if msg.tool_call:
        handle_tool_call(msg.tool_call)
    if msg.go_away:
        # server is asking you to reconnect soon
        schedule_reconnect(msg.go_away.time_left)
    if msg.usage_metadata:
        log_usage(msg.usage_metadata)
```

Two important facts about the output audio:

- The default Live model **outputs 24 kHz PCM16 mono**, not the 16 kHz you send in. Your speaker / resampler must handle that. <!-- verify-at-runtime -->
- Audio is split across many `inline_data` parts; concatenate them for each turn or stream them straight to playout as they arrive.

#### Step 6 — Barge-in / interruption

When the server's VAD detects user speech mid-assistant-turn (and `activity_handling` is `START_OF_ACTIVITY_INTERRUPTS`), you will see:

- a `serverContent` with `interrupted=True`,
- no further audio parts for the interrupted turn,
- a new turn beginning once the user stops speaking.

Your client must: (a) flush local playout immediately on `interrupted`, (b) treat the new user activity as a fresh turn. Unlike OpenAI Realtime, Gemini Live **does not require a separate "tell the server what the user actually heard" call** — the server stops producing audio immediately and the model's conversation state reflects only what it emitted up to the interrupt. If your local jitter buffer held more audio than the server actually sent, your logs may show the user "hearing" ms of audio that the model does not know it emitted; log the delta as a diagnostic.

<!-- verify-at-runtime: whether/when an explicit "truncate" equivalent is exposed on the Live protocol -->

#### Step 7 — Tool calling

```
Server → Client
  toolCall
    functionCalls: [ { name, args, id } ]

Client → Server (after running the function)
  toolResponse
    functionResponses: [ { name, id, response: {...} } ]
```

The SDK:

```python
await session.send_tool_response(function_responses=[
    types.FunctionResponse(id=call.id, name=call.name, response={"status": "ok"})
])
```

The model continues the turn — you do not send a `response.create` equivalent; the Live session drives the response loop itself once a `toolResponse` arrives. Long tool latency is user-visible as dead air; return something fast or stream a filler acknowledgement before the real result.

Note: the Live API also supports server-side tools (Google Search, code execution). Those are returned to the client as `toolCallCancellation` and `toolCall` records when their state changes; your client does not have to run them.

#### Step 8 — Teardown

Each turn's `serverContent.turn_complete=True` ends a turn — the session stays open. The session only ends when:

- You close the `async with` context,
- The server emits `goAway` (signals the connection will be cut, with `time_left` so you can reconnect gracefully), or
- A session-length limit is hit. Live sessions are not infinite — audio-only sessions are typically capped on the order of 15 minutes, with context-window and token limits also applying. Reconnect strategy and context-resume are the responsibility of the client. <!-- verify-at-runtime: current session-length limit and whether `sessionResumption` is GA on your model/region -->

`usageMetadata` carries the per-turn or per-session token counts; log them.

### 2.3 Minimal Python client (SDK, asyncio)

```python
# python 3.11+;  pip install google-genai
import asyncio, os, sys
from google import genai
from google.genai import types

MODEL = "gemini-2.0-flash-live-001"   # verify-at-runtime

config = types.LiveConnectConfig(
    response_modalities=["AUDIO"],
    system_instruction=types.Content(parts=[types.Part(text=
        "You are a terse voice assistant. Reply in <= 2 sentences.")]),
    speech_config=types.SpeechConfig(
        voice_config=types.VoiceConfig(
            prebuilt_voice_config=types.PrebuiltVoiceConfig(voice_name="Puck"))),
    realtime_input_config=types.RealtimeInputConfig(
        automatic_activity_detection=types.AutomaticActivityDetection(
            silence_duration_ms=500, prefix_padding_ms=20),
        activity_handling=types.ActivityHandling.START_OF_ACTIVITY_INTERRUPTS),
)

async def pump_mic(session, reader):
    # mic chunks: 16kHz mono PCM16, ~40ms → 1280 bytes
    while True:
        chunk = await reader.readexactly(1280)
        if not chunk:
            break
        await session.send_realtime_input(
            audio=types.Blob(data=chunk, mime_type="audio/pcm;rate=16000"))

async def pump_events(session):
    async for msg in session.receive():
        sc = getattr(msg, "server_content", None)
        if sc and sc.model_turn:
            for part in sc.model_turn.parts:
                if part.inline_data and part.inline_data.data:
                    sys.stdout.buffer.write(part.inline_data.data); sys.stdout.buffer.flush()
                if part.text:
                    sys.stderr.write(part.text); sys.stderr.flush()
        if sc and sc.interrupted:
            sys.stderr.write("\n[barge-in]\n")
        if sc and sc.turn_complete:
            sys.stderr.write("\n")
        if msg.tool_call:
            # run tools, then:
            # await session.send_tool_response(function_responses=[...])
            pass
        if msg.go_away:
            sys.stderr.write(f"[goAway in {msg.go_away.time_left}s]\n")
        if msg.usage_metadata:
            sys.stderr.write(f"[usage] {msg.usage_metadata}\n")

async def main():
    client = genai.Client(api_key=os.environ["GEMINI_API_KEY"])
    loop = asyncio.get_running_loop()
    reader = asyncio.StreamReader()
    await loop.connect_read_pipe(lambda: asyncio.StreamReaderProtocol(reader), sys.stdin.buffer)
    async with client.aio.live.connect(model=MODEL, config=config) as session:
        await asyncio.gather(pump_mic(session, reader), pump_events(session))

asyncio.run(main())
```

What a graded submission should name as missing from this scaffold:

- **Session resume** on `goAway`. The Live API exposes `session_resumption` tokens on some models; a client that does not resume loses conversation context on every reconnect.
- **Input transcription** (`input_audio_transcription` on `setup`) — without it, you cannot log the user's words as text.
- **Resampling** if the source mic isn't 16 kHz. Output is 24 kHz PCM16 — your playout must handle that.
- **Backpressure** — `send_realtime_input` is fire-and-forget at the SDK level; a slow network can accumulate queued frames.

### 2.4 Protocol comparison with OpenAI Realtime (ex-01)

Side-by-side at the event-shape level. Use this table to convert prototypes between the two protocols.

| Concept | OpenAI Realtime | Gemini Live |
|---|---|---|
| Transport | WebSocket or WebRTC | WebSocket (`BidiGenerateContent`) |
| Auth | `Authorization: Bearer` + `OpenAI-Beta: realtime=v1`; ephemeral tokens for WebRTC | `?key=<API_KEY>` (Gemini API) or Google ADC (Vertex) |
| Config call | `session.update` | `setup` |
| Mic audio frame | `input_audio_buffer.append` (base64) | `realtimeInput.audio` (`Blob` mime `audio/pcm;rate=16000`) |
| VAD mode | `turn_detection: { type: "server_vad" \| "semantic_vad" \| null }` | `realtime_input_config.automatic_activity_detection` |
| Manual turn boundary | `input_audio_buffer.commit` + `response.create` | `activity_start` / `activity_end` on `realtimeInput` |
| Assistant audio chunk | `response.audio.delta` (base64) | `serverContent.model_turn.parts[*].inline_data` |
| Assistant transcript | `response.audio_transcript.delta/done` | `serverContent.output_transcription` (when enabled) |
| User transcript | `conversation.item.input_audio_transcription.completed` | `serverContent.input_transcription` (when enabled) |
| Barge-in signal | `input_audio_buffer.speech_started` | `serverContent.interrupted=true` |
| Barge-in "truncate" | explicit `conversation.item.truncate` with `audio_end_ms` | implicit — server stops emitting; no explicit truncate |
| Tool call | `response.function_call_arguments.delta/done` + `conversation.item.create` (function_call_output) + `response.create` | `toolCall` → `toolResponse` (no explicit re-trigger) |
| End-of-turn marker | `response.done` | `serverContent.turn_complete=true` |
| Per-session usage | `response.done.usage` (per response) | `usageMetadata` (per message / session) |
| Server-initiated hints | `rate_limits.updated` | `goAway` |
| Session length | Not explicitly bounded; rate limited | Bounded (audio-only sessions cap on the order of minutes); use resumption |

<!-- verify-at-runtime: session length cap and resumption are the two fields most likely to change. -->

The *shape* is the same — setup, stream in, stream out, VAD events, tool-call loop, teardown — but the names and barge-in semantics differ. A production voice-agent engineer should be able to translate between the two blind. Exercises 3 and 4 then draw the comparison to the *pipeline* approach.

## 3. Validation steps

A finished submission should pass each check.

1. **Lifecycle narration.** All eight steps named with the client message and the corresponding server message(s).
2. **Surface decision.** Submission says *Developer API* or *Vertex AI* and names the reason tied to auth / IAM / residency — not just "the one I found first."
3. **Model choice.** A Live-capable model is named; if it is a native-audio variant, the submission says *why* (expressiveness / latency / paralinguistics). If half-cascade, same test.
4. **`setup` correctness.** `response_modalities` is a single-entry list; `speech_config` picks a voice; `realtime_input_config` picks VAD on or off with reasoning; `tools` (if present) are valid function declarations.
5. **Sample-rate hygiene.** Submission explicitly states: input 16 kHz PCM16, output 24 kHz PCM16; resampling path named for non-16 kHz mics.
6. **Barge-in correctly handled.** Submission flushes local playout on `interrupted` and documents that no explicit truncate is needed; any local-vs-server audio-delta risk is at least acknowledged.
7. **Tool-call loop correct.** `toolCall` → run → `toolResponse`; note that no `response.create` equivalent is required.
8. **Reconnect / resumption.** Submission names `goAway` and either uses `sessionResumption` or explicitly says why it doesn't (short-session product, acceptable context loss).
9. **Reproducer.** A minimal `google-genai` client exists that at least connects, sends `setup`, pushes one PCM blob, and prints a `serverContent` stream.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Lifecycle coverage | 20 % | All 8 steps named with their client/server messages | Each step annotated with its failure mode and log signature |
| Surface decision | 5 % | Developer vs Vertex AI chosen with reason | IAM, VPC-SC, and regional-residency implications spelled out |
| Model choice | 10 % | Live-capable model named with native-vs-cascade framing | Trade-off against Latin-alphabet vs multilingual coverage explicitly addressed |
| `setup` correctness | 15 % | Required `response_modalities`, `speech_config`, `realtime_input_config`, `tools` set correctly | Each field annotated with voice-agent rationale and the failure mode of a wrong value |
| Sample-rate hygiene | 10 % | Input 16 kHz / output 24 kHz stated; resampling path mentioned | Rate-mismatch failure modes enumerated (why 48 kHz mic sounds "slow-motion" if not resampled) |
| Barge-in | 10 % | Local flush on `interrupted`; absence of explicit truncate noted | Audio-delta between local playout and server-emitted audio logged for debugging |
| Tool-call loop | 10 % | `toolCall` → `toolResponse` named; no spurious `response.create` | Slow-tool-filler UX pattern called out |
| Session continuity | 10 % | `goAway` handled; reconnect plan named | `sessionResumption` tokens used (if GA on chosen model) |
| Reproducer | 10 % | Minimal client connects, configures, streams one audio response | Reproducer handles `goAway`, resume, and barge-in flush |

Scoring < 70 % is a fail-and-redo. The most common sub-70 failure: treating Live like the REST API — opening a new WebSocket per turn and losing context — or getting the 16 kHz-in / 24 kHz-out sample-rate asymmetry wrong.

## 5. Common mistakes

- **Requesting multiple response modalities on one connection.** Live models accept one modality per connection. Pick `"AUDIO"` or `"TEXT"`, not both.
- **Assuming output is 16 kHz.** Output is 24 kHz PCM16 mono by default. 16 kHz-only playout will sound pitched-down.
- **Opening a new WebSocket per turn.** The Live session carries conversation state; reconnecting discards it unless you use `sessionResumption`.
- **Hard-coding a model ID that is in preview.** Preview Live models are renamed/retired at short notice. Pin the ID in config, not in code; monitor the Live model-release page.
- **Not reading `goAway`.** The server tells you in advance that it is about to drop the connection and how much time is left. Ignoring it means your voice agent cuts out mid-sentence.
- **Running the session key in the browser on the Developer API.** The API key becomes a public credential; voice-agent engineers on consumer-facing products should use Vertex AI with ephemeral tokens or a server relay (see exercise-03's relay pattern, which also solves this).
- **Confusing `activity_handling` and `turn_coverage`.** Different knobs: `activity_handling` is whether user speech aborts the model; `turn_coverage` is whether non-VAD audio counts as input. Teams routinely cargo-cult both from a sample and get surprised when the model starts transcribing coughs.
- **Treating the `interrupted` flag as a terminal event.** It is a mid-stream marker; more messages follow (the next user turn). Don't tear down the session on it.
- **Expecting an OpenAI-shaped `conversation.item.truncate`.** Gemini Live does not have one; the server truncates server-side state automatically on interrupt. Attempting to "ack what was heard" on Live is a category error.
- **Forgetting input/output transcription.** Without `input_audio_transcription` and `output_audio_transcription`, your logs are audio-only — operationally useless at scale.
- **Treating native-audio and half-cascade as interchangeable.** Native-audio models capture affect / paralinguistics; half-cascade models route through text and lose them. If your product needs the user to be able to shout angrily and have the model notice, you want native audio; if your product needs deterministic, replay-friendly text transcripts, half-cascade is easier.

## 6. References

Primary — Google documentation (verify at runtime; the Live API is still iterating):

- Gemini Live API guide — `https://ai.google.dev/gemini-api/docs/live`
- Gemini Live API capabilities — `https://ai.google.dev/gemini-api/docs/live-api-capabilities`
- Gemini Live session management — `https://ai.google.dev/gemini-api/docs/live-session`
- Gemini Live tools — `https://ai.google.dev/gemini-api/docs/live-tools`
- Vertex AI Live API — `https://cloud.google.com/vertex-ai/generative-ai/docs/live-api`
- `google-genai` Python SDK — `https://github.com/googleapis/python-genai`
- `BidiGenerateContent` method reference — `https://ai.google.dev/api/live` <!-- verify-at-runtime: URL may have been restructured -->
- Gemini Live cookbook examples — `https://github.com/google-gemini/cookbook`

Related exercises inside this module:

- exercise-01 (OpenAI Realtime) — the protocol twin; the comparison table in §2.4 is the hinge.
- exercise-03 (Claude relay) — the pipeline path for models without a native Live surface.
- exercise-04 (pipeline vs native) — decision drill that uses this lifecycle plus exercise-01 as input.
- exercise-05 (half vs full duplex) — explains why Live barge-in is still half-duplex at today's model tier.

Standards referenced indirectly (used by the compliance / observability framing):

- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — the "measure, govern, document" discipline behind `usageMetadata` logging, and the input/output-transcription auditability stance.
- OWASP Machine Learning Security Top 10 — `https://owasp.org/www-project-machine-learning-security-top-10/` — relevant when the Live channel is deliberately attacked (prompt injection via audio, forced tool-call exfiltration).
- MITRE ATLAS — `https://atlas.mitre.org/` — threat-model reference for audio-adversarial scenarios on bidirectional voice channels.

Source policy note. Model IDs, field names, voice names, and session-length limits in this document carry `<!-- verify-at-runtime: ... -->` because the Live API is actively iterating. Do not quote specific IDs or limits downstream without re-verifying against the current `ai.google.dev/gemini-api/docs/live` and `cloud.google.com/vertex-ai/generative-ai/docs/live-api` pages.
