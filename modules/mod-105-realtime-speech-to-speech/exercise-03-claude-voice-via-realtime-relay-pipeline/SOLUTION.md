# exercise-03 — Claude Voice via a Realtime-Relay Pipeline — Reference Solution

## 1. Solution overview

Anthropic's Claude does not currently expose a native speech-in / speech-out "Realtime"-style surface. To put Claude in a voice loop you build a **realtime relay**: a thin server that terminates the user's audio transport (WebSocket or WebRTC), runs an **STT → Claude Messages API → TTS** pipeline, and streams audio back — behaving, from the browser or telephony side, like a single speech-native endpoint.

This exercise walks the reference relay architecture, gives a runnable Python scaffold, and sets the rubric for what makes a *good* relay vs. a naive "await STT; await Claude; await TTS" chain. The design decisions the exercise forces you to commit to:

- Where does the WebRTC/WebSocket terminate — your relay, or a managed media SFU (LiveKit, Daily, Twilio)?
- How does the relay enforce **streaming at every stage** (STT partial → first LLM tokens → first TTS audio) so time-to-first-audio is low?
- How does the relay detect **barge-in** and cancel in-flight Claude and TTS calls cleanly?
- How does it preserve **Anthropic prompt caching** across turns so Claude isn't billed and warmed up from scratch every turn?
- How does it handle **tool calls** that happen inside the LLM turn without starving the audio side?
- What does the **observability surface** look like end-to-end?

All Anthropic / provider-specific details come from the current Anthropic docs and the public SDK docs listed in §6; vendor-side details (voice names, model IDs, pricing, specific endpoint paths) carry `<!-- verify-at-runtime: ... -->` markers.

<!-- verify-at-runtime: all Claude model IDs, Anthropic Messages API event names (message_start, content_block_delta, etc.), and TTS provider specifics should be re-pulled from https://docs.anthropic.com and each TTS provider's current docs before quoting externally. -->

## 2. Implementation

### 2.1 Where "realtime relay" came from (vocabulary)

Terminology used here, aligned with how it is used in the voice-agent community:

- **Speech-native endpoint.** A single bidirectional stream whose other end is a model that consumes and emits audio (OpenAI Realtime, Gemini Live, Moshi). ex-01/02.
- **Pipeline (STT→LLM→TTS).** Three sequential models stitched by your code. Classic voice-stack shape (LiveKit voice-pipeline-agent, Pipecat, Vocode).
- **Realtime relay.** A **server** that exposes a speech-native-shaped surface to clients and internally runs a pipeline. From the client's perspective it looks like a Realtime API; from the inside it is a pipeline. This is the shape you ship when the LLM you want is text-only.

The LiveKit "voice-pipeline-agent" pattern, Pipecat's "pipeline" abstraction, and OpenAI's own "openai-realtime-relay" sample all implement variants of this. Which off-the-shelf piece you lean on (LiveKit Agents, Pipecat, Vocode, custom asyncio) is secondary — the reference architecture below is the same.

### 2.2 Reference architecture

```
┌──────────┐    WebRTC / WebSocket / SIP     ┌──────────────────────────────────────────┐
│          │ ──── audio frames  ───────────► │                                          │
│  Client  │                                 │             RELAY (your service)         │
│ browser/ │ ──── events (text, barge-in)─► │ ┌──────────────────────────────────────┐ │
│   phone  │                                 │ │  Audio in → VAD / turn boundary      │ │
│          │ ◄─── audio frames  ───────────── │ │  → STT stream  (Deepgram / AAI /…)   │ │
│          │ ◄─── events (text)   ─────────── │ │  → Claude Messages API (streaming)   │ │
└──────────┘                                 │ │    (prompt cache, tools, barge-in)   │ │
                                             │ │  → TTS stream (Cartesia / ElevenLabs │ │
                                             │ │    / OpenAI TTS / Deepgram Aura)     │ │
                                             │ │  → Audio out                         │ │
                                             │ └──────────────────────────────────────┘ │
                                             │          ▲ Observability bus ▲           │
                                             └──────────────────────────────────────────┘
```

Key architectural moves:

- **Media terminates on the relay (or on a media SFU the relay controls).** For telephony voice agents, use a SIP gateway (Twilio Voice / LiveKit SIP) that bridges to the relay via WebSocket or WebRTC. For browser clients, use a SFU like LiveKit or Daily so the browser does not open a direct socket to Claude's provider. The relay never exposes your Anthropic API key to the client.
- **STT emits partials.** Use a streaming STT (Deepgram, AssemblyAI, OpenAI gpt-4o-transcribe realtime, Google Chirp streaming). Do **not** wait for the full final before starting the LLM for one-shot queries — but do wait until the user has stopped talking (endpointed utterance) before actually calling Claude, because Claude will not be able to retract its answer if the user continues. See exercise-02 in mod-103 for the STT contract.
- **Claude streams.** Use the Anthropic Messages API with `stream=True` so you see `content_block_delta` text tokens as they are produced. Pipe those tokens to the TTS as they arrive.
- **TTS streams.** Use a streaming TTS (Cartesia Sonic streaming, ElevenLabs streaming, OpenAI TTS streaming, Deepgram Aura streaming). The first speakable audio chunk should leave the relay before Claude has finished generating.
- **Prompt caching is on.** Cache the system prompt + tools block + any long static context across turns. This is where a pipeline wins back cost from a speech-native endpoint on heavy-context agents.
- **Barge-in cancels all three.** When the user starts speaking during assistant playout, you cancel the in-flight Claude stream (`client.messages.stream(...)` context exit), cancel the TTS stream, and flush local playout.

### 2.3 Chunking + pacing — the single idea that makes a pipeline feel real-time

The latency budget to defend (mod-109 goes deep on this; summary here):

```
User stops speaking
   └─ STT emits final      (~0.1 – 0.4 s after speech end, provider-dependent)
      └─ Claude first token (~0.2 – 0.8 s after STT final, cached-prompt case)
         └─ TTS first audio chunk  (~0.2 – 0.4 s after first speakable sentence chunk)
            └─ Playback starts on client
Target end-to-end: < 1.0 s "voice-to-voice" for a cached-prompt, short-answer turn.
```

To hit that, the relay's one job is to pipeline-break on **speakable units**, not on the full LLM output:

```python
# pseudo-code — chunk Claude's text output into speakable units and stream each into TTS
async def stream_assistant_turn(relay, user_turn):
    sentence_buffer = ""
    async with anthropic.messages.stream(
        model="claude-sonnet-4-6",          # verify-at-runtime
        system=[{"type":"text","text":SYSTEM_PROMPT,"cache_control":{"type":"ephemeral"}}],
        messages=relay.history + [user_turn],
        tools=relay.tools,
        max_tokens=400,
    ) as stream:
        async for event in stream:
            if event.type == "content_block_delta" and event.delta.type == "text_delta":
                sentence_buffer += event.delta.text
                # flush on sentence-end or clause-end punctuation
                while True:
                    boundary = find_speakable_boundary(sentence_buffer)
                    if boundary is None:
                        break
                    chunk, sentence_buffer = sentence_buffer[:boundary], sentence_buffer[boundary:]
                    await relay.tts.feed(chunk)   # TTS streams audio frames out to client
            if event.type == "content_block_stop":
                if sentence_buffer:
                    await relay.tts.feed(sentence_buffer)
                    sentence_buffer = ""
            if event.type == "message_stop":
                await relay.tts.close_turn()
```

The `find_speakable_boundary` function should split on sentence-ends (`.`, `?`, `!`, `…`) plus mid-sentence clauses (`,`, `;`, `:`) when the buffer grows past ~40–80 characters without a sentence terminator, so you don't block waiting for a full sentence on long answers.

`cache_control: ephemeral` on the system / tools block lets you reuse the cached prefix across turns inside the session — prompt caching is a major cost lever on heavy-context voice agents. See the Anthropic prompt-caching docs.

### 2.4 Reference Python scaffold

```python
# python 3.11+;  pip install anthropic deepgram-sdk cartesia-python livekit-agents
# Minimal reference for illustration; omit ms-level jitter handling for brevity.

import asyncio, os, re
from anthropic import AsyncAnthropic
# STT + TTS provider SDKs imported below are illustrative; swap for your choices.

SYSTEM_PROMPT = (
    "You are a terse voice agent for Acme appointment booking. "
    "Keep replies under 2 sentences unless asked for detail."
)

anth = AsyncAnthropic()
# stt_client = ...   # streaming STT (Deepgram / AssemblyAI / gpt-4o-transcribe-realtime)
# tts_client = ...   # streaming TTS (Cartesia / ElevenLabs / Deepgram Aura / OpenAI TTS)

SENTENCE_RE = re.compile(r"(.*?[\.\?!…])(\s|$)", re.DOTALL)
CLAUSE_RE   = re.compile(r"(.{40,}?[,;:])(\s|$)", re.DOTALL)

def find_speakable_boundary(buf: str) -> int | None:
    m = SENTENCE_RE.match(buf)
    if m: return m.end()
    m = CLAUSE_RE.match(buf)
    if m: return m.end()
    return None


class Relay:
    def __init__(self):
        self.history = []
        self.tools   = []
        self.active_llm = None
        self.active_tts = None

    async def on_user_final(self, text: str):
        # Called when STT emits endpointed-final text for the user's turn.
        if self.active_llm:
            await self.cancel_assistant_turn()  # barge-in handling
        user_turn = {"role": "user", "content": text}
        self.history.append(user_turn)
        self.active_llm = asyncio.create_task(self._assistant_turn())

    async def on_user_speech_start(self):
        # Called by STT/VAD on *start* of user speech — barge-in trigger.
        if self.active_llm:
            await self.cancel_assistant_turn()

    async def cancel_assistant_turn(self):
        if self.active_llm:
            self.active_llm.cancel()
            try: await self.active_llm
            except asyncio.CancelledError: pass
            self.active_llm = None
        if self.active_tts:
            await self.active_tts.cancel()
            self.active_tts = None
        # flush local playout (client-side responsibility, via a control event)
        await self.send_event({"type": "playout.flush"})

    async def _assistant_turn(self):
        buf = ""
        assistant_text = ""
        try:
            async with anth.messages.stream(
                model="claude-sonnet-4-6",   # verify-at-runtime
                system=[{"type": "text", "text": SYSTEM_PROMPT,
                         "cache_control": {"type": "ephemeral"}}],
                messages=self.history,
                tools=self.tools,
                max_tokens=400,
            ) as stream:
                self.active_tts = await self.open_tts_stream()
                async for event in stream:
                    if event.type == "content_block_delta" and event.delta.type == "text_delta":
                        buf += event.delta.text
                        assistant_text += event.delta.text
                        while (b := find_speakable_boundary(buf)) is not None:
                            await self.active_tts.feed(buf[:b])
                            buf = buf[b:]
                    elif event.type == "content_block_stop":
                        if buf:
                            await self.active_tts.feed(buf); buf = ""
                    elif event.type == "message_stop":
                        await self.active_tts.close_turn()
            self.history.append({"role": "assistant", "content": assistant_text})
        finally:
            self.active_tts = None

    async def open_tts_stream(self):
        # Return a provider-specific stream object with .feed(text), .cancel(), .close_turn()
        ...

    async def send_event(self, evt): ...
```

The scaffold is deliberately provider-agnostic on STT/TTS so the engineering shape is visible. Real relays either wire LiveKit Agents' `VoicePipelineAgent` (which gives you STT/TTS plugins, VAD, turn-taking, and barge-in cancellation out of the box with a Claude LLM plugin) or Pipecat's `Pipeline` (same idea, different abstraction).

### 2.5 Tool calls in a relay

A tool call makes the pipeline *conditionally longer*: the model emits a `tool_use` block, your code runs the tool, and the model emits text in a continuation. For latency, this is the worst case. Two mitigations:

1. **Prime a filler.** When you see the first `content_block_start` with `type: "tool_use"`, immediately feed a short filler phrase ("One moment, let me check…") to the TTS, then continue with the real response once the tool completes. Only do this on tools that exceed a latency threshold — not on trivial lookups — otherwise the voice loop sounds hesitant.
2. **Parallel-safe tools.** If the tool call is read-only and the model's *default* answer is often "yes" / "ok", you can speculatively fetch in parallel and discard if the model's tool call ends up different. Dangerous for write-side tools; useful for read-side lookups (e.g., "check calendar availability").

The continuation after the tool result is a **new Messages API call** with the user turn, assistant turn (with the `tool_use` block), and a user turn carrying the `tool_result` block. Prompt caching extends cleanly across these.

### 2.6 Barge-in semantics in a relay

The relay has three cancellation points, and all three must fire:

| Component | Cancel how | Failure if skipped |
|---|---|---|
| Claude stream | Exit the `anthropic.messages.stream` context manager / cancel the task | Keep paying for tokens the user never hears |
| TTS stream | Call provider-specific cancel (e.g., Cartesia `send({"type":"cancel"})`) | Keep paying for synthesis + occupy the socket |
| Local playout | Send a `playout.flush` control event to client | User continues hearing ms of buffered audio and feels the agent "ignored" the barge-in |

Compared to the speech-native endpoints (ex-01/02), the relay has *more* places to cancel and therefore more places to be slow. The speed with which all three cancel is a measurable latency metric — treat it as such and include "barge-in-to-silence" in your dashboards.

### 2.7 Observability — the end-to-end trace

Emit one structured log per turn with the fields below. These are the fields a graded submission is expected to name:

```json
{
  "session_id": "sess_...",
  "turn_id": "turn_...",
  "user_speech_start_ts": "...",
  "user_speech_end_ts": "...",
  "stt_final_ts": "...",
  "llm_first_token_ts": "...",
  "llm_last_token_ts": "...",
  "tts_first_audio_ts": "...",
  "tts_last_audio_ts": "...",
  "client_first_audio_ts": "...",            // from client telemetry
  "barge_in": false,
  "tool_calls": [{"name":"...","latency_ms":...}],
  "user_text": "...",
  "assistant_text": "...",
  "anthropic_usage": { "input_tokens": ..., "cached_input_tokens": ..., "output_tokens": ... },
  "stt_provider": "deepgram",
  "tts_provider": "cartesia",
  "llm_model": "claude-sonnet-4-6"           // verify-at-runtime
}
```

From this you compute: **voice-to-voice latency** (`client_first_audio_ts − user_speech_end_ts`), **cache-hit ratio** (`cached_input_tokens / input_tokens`), **barge-in rate**, **tool-call tax**. These are the four numbers the relay operator steers by.

### 2.8 When to use an off-the-shelf relay vs build your own

| Scenario | Recommendation |
|---|---|
| Browser-first product, want to ship in a week | LiveKit Agents' `VoicePipelineAgent` with Anthropic LLM plugin + chosen STT/TTS plugins. The infra engineering is already written. |
| Telephony-first product with SIP ingress | LiveKit Agents + LiveKit SIP, or Pipecat with Twilio/Daily SIP bridges. |
| Heavy custom state machine (banking, clinical) with tight guardrails | Build on Pipecat or custom asyncio for finer control over the frame graph. |
| Just need to prove the Claude voice path works | Minimal asyncio scaffold like §2.4 is enough. |
| Self-hosted end-to-end (data residency) | LiveKit self-hosted or a custom relay; make sure STT/TTS vendors are also deployable in-region. |

All of these are relays by the definition in §2.1. "Build your own" means owning the barge-in, reconnect, backpressure, and prompt-caching code that LiveKit / Pipecat have already written.

## 3. Validation steps

A finished submission should pass each check.

1. **Architecture diagram present.** The three-stage pipeline with the STT/LLM/TTS boxes is drawn, media transport is labelled, and the point at which the API key for Anthropic terminates is marked. No arrow goes from the client directly to Anthropic.
2. **Streaming at every stage.** Submission explicitly states: STT streams partials + final, Claude is `stream=True`, TTS is streaming, playout starts on first speakable chunk. A submission that awaits the full LLM output before starting TTS fails this check.
3. **Prompt caching on.** The system prompt + tools block is wrapped in `cache_control: ephemeral` and the relay preserves the cached prefix across turns inside a session. Submission names expected cache-hit ratio.
4. **Barge-in triple-cancel.** On user-speech-start, the submission cancels the Claude stream, cancels the TTS stream, and flushes local playout — all three.
5. **Tool-call latency plan.** Submission names the filler-phrase technique or justifies why it is not needed for the chosen tool shape.
6. **Observability fields.** The per-turn log carries at least: user_speech_end_ts, stt_final_ts, llm_first_token_ts, tts_first_audio_ts, client_first_audio_ts, barge_in, tool_calls, anthropic_usage (with cached_input_tokens broken out), and provider + model identifiers.
7. **Latency budget quoted.** Submission names a voice-to-voice target (e.g., < 1.0 s for cached-prompt short answers) and the per-stage budget that adds up to it.
8. **Reproducer.** A minimal working scaffold exists (modified §2.4 is fine) that runs end-to-end with at least one chosen STT and TTS provider.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Architecture | 15 % | Three stages drawn, API key boundaries marked, media transport named | Both telephony and browser ingress drawn; failure-mode of each named |
| Streaming end-to-end | 15 % | STT partials + Claude streaming + TTS streaming + sentence-level pipeline-break | Clause-level pipeline-break for long answers; measured TTFB per stage |
| Prompt caching | 10 % | `cache_control: ephemeral` on system/tools block | Cache invalidation plan on system-prompt updates called out |
| Barge-in | 15 % | All three cancellation points fire; client playout flush is an explicit event | "Barge-in-to-silence" metric defined and measured |
| Tool calls | 10 % | Filler-phrase pattern named, or absence justified | Speculative read-side fetch pattern discussed |
| Observability | 15 % | Per-turn log with the fields in §2.7 | Dashboards for voice-to-voice p50/p95, cache-hit ratio, barge-in rate, tool-call tax |
| Latency budget | 10 % | Named target and per-stage budgets | Dependency on STT finalisation latency and TTS first-chunk latency explicitly charted |
| Reproducer | 10 % | Running scaffold with one STT + one TTS provider, Claude streaming | Reproducer supports LiveKit/Pipecat plugin surface; swap-in test for two TTS providers |

Scoring < 70 % is a fail-and-redo. The most common sub-70 failure: awaiting full Claude output before starting TTS, so the voice-to-voice latency is ~3× what it could be, and never measuring where the time actually went.

## 5. Common mistakes

- **"Streaming" that isn't streaming.** Using `client.messages.create(...)` (non-streaming), then feeding the full text to a streaming TTS. The streaming is only on one side — total latency is dominated by Claude's generation time.
- **Pipeline-break on message-end only.** Waiting for `message_stop` before starting TTS gives up most of the latency advantage. Break at sentence (or clause) boundaries inside `content_block_delta`.
- **Prompt cache not reused across turns.** Each turn constructs a new `system` block without `cache_control` or with different text; cache-hit ratio stays near zero and the voice-agent cost looks wildly higher than a speech-native endpoint.
- **Exposing the Anthropic key to the client.** The whole point of a relay is to keep that key server-side. Shipping it in the browser is a credential-leak class incident.
- **Barge-in that only stops local playout.** The user feels the barge-in because playout stops, but Claude keeps generating and TTS keeps synthesising. You pay for both.
- **Barge-in that only cancels Claude.** TTS continues to emit audio that the client discards; you pay for the synthesis anyway.
- **Blocking on tool calls without a filler.** A tool call that takes 800 ms turns a normal voice reply into an obvious pause. For latency-sensitive tools, prime a filler.
- **One streaming STT final per turn — no VAD on the relay side.** You rely on the STT's VAD entirely, so a provider's endpointing bug ends up as the agent's bug. Keep a backup VAD in the relay.
- **Resample-on-the-fly in the hot path.** Running a high-quality resampler per 20 ms frame can cost real latency on CPU. Negotiate rates at media-terminate time so the hot path is passthrough.
- **No reconnect for STT or TTS.** Both can drop mid-session. Without reconnect the voice loop freezes and the user hears dead air; the fix is per-turn re-open plus a short retry window.
- **One global `history` list growing unbounded.** Long sessions blow past Claude's context window and latency climbs. Trim or summarise once you cross a threshold.
- **Measuring "first-token latency" instead of "voice-to-voice."** The user does not hear tokens. Measure the time from `user_speech_end_ts` to `client_first_audio_ts`.

## 6. References

Primary — Anthropic and relay-framework documentation:

- Anthropic Messages API — `https://docs.anthropic.com/en/api/messages`
- Anthropic streaming messages — `https://docs.anthropic.com/en/api/messages-streaming`
- Anthropic prompt caching — `https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching`
- Anthropic tool use — `https://docs.anthropic.com/en/docs/agents-and-tools/tool-use`
- Anthropic Python SDK — `https://github.com/anthropics/anthropic-sdk-python`
- LiveKit Agents (VoicePipelineAgent) — `https://docs.livekit.io/agents/`
- LiveKit Anthropic plugin — `https://docs.livekit.io/agents/integrations/llm/anthropic/` <!-- verify-at-runtime: URL may have been restructured -->
- Pipecat framework — `https://docs.pipecat.ai/` and `https://github.com/pipecat-ai/pipecat`
- Vocode (open-source voice agent framework) — `https://github.com/vocodedev/vocode-core`
- OpenAI "realtime relay" reference sample — `https://github.com/openai/openai-realtime-agents` <!-- verify-at-runtime: repo name/URL -->

STT and TTS providers commonly used in Claude relays (vendor docs — current at pull time):

- Deepgram (STT + Aura TTS) — `https://developers.deepgram.com/docs`
- AssemblyAI (STT) — `https://www.assemblyai.com/docs`
- OpenAI `gpt-4o-transcribe` (STT) — `https://platform.openai.com/docs/guides/speech-to-text`
- Cartesia Sonic (TTS) — `https://docs.cartesia.ai/`
- ElevenLabs (TTS) — `https://elevenlabs.io/docs`
- OpenAI TTS — `https://platform.openai.com/docs/guides/text-to-speech`

Related exercises inside this module:

- exercise-01 (OpenAI Realtime) — the speech-native alternative the relay stands in for.
- exercise-02 (Gemini Live) — the Google speech-native alternative; the relay also stands in for products that need Claude specifically.
- exercise-04 (pipeline vs native) — this exercise supplies the "pipeline" half of that decision.
- exercise-05 (half vs full duplex) — the relay here is half-duplex by construction; discussing why is part of the decision.

Standards referenced indirectly:

- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — the "measure, govern, document" discipline behind the per-turn observability log.
- OWASP Machine Learning Security Top 10 — `https://owasp.org/www-project-machine-learning-security-top-10/` — relevant to prompt-injection-via-audio and tool-exfiltration risks in the relay.
- MITRE ATLAS — `https://atlas.mitre.org/` — threat-model reference for audio-adversarial scenarios against the pipeline.

Source policy note. Specific Claude model IDs, Anthropic SDK method shapes, and provider feature details are pinned to doc URLs; the relay architecture is the stable part, and the vendor-side details are the part that moves. Re-verify SDK call shapes and model IDs against the current `docs.anthropic.com` pages and provider docs before quoting externally.
