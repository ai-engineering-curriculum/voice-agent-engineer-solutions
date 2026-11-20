# exercise-01 — Voice Agent Engineer Landscape Read and Map — Reference Solution

## 1. Solution overview

This exercise is a design / reading exercise. There is no single correct map — the correct answer is **a defensible map, drawn from current postings you actually read**, that uses the six-category frame from Chapter 1 as scaffolding. This solution shows:

- A worked example of what "defensible" looks like: eight placements, spread across four categories, each with a cited centre-of-gravity line from the posting and a declared stack.
- The decision rationale a reviewer should look for.
- The one-page synthesis format.
- The "this is us / this is adjacent / this is red flag" breakdown applied to one worked posting.

Because posting URLs decay fast, the worked example below uses **clearly-labeled illustrative composites** (`Posting A`…`Posting H`) that match the shape of real postings on the job-site pages of the companies named in Chapter 1 at the time the chapter was written. When you do the exercise, **replace these composites with real, archived postings** (Wayback Machine, `archive.today`, or a PDF print) so a reviewer can verify the centre-of-gravity line against the source.

The six categories (from Chapter 1) are:

1. Voice-AI pure-plays (ASR / TTS / audio-first labs)
2. Realtime-voice orchestration platforms
3. Telephony providers with voice-AI surfaces
4. Frontier labs with realtime voice stacks
5. Applied-voice product companies (CX / contact-centre / agent-ops)
6. Vertical-voice applications

## Implementation

The worked answer below is the implementation of this exercise: the artefacts a reviewer should see in a finished submission (map, ambiguity justifications, synthesis, and the this/adjacent/red-flag breakdown), followed by validation steps, rubric, and common mistakes. To execute the exercise yourself, follow the shape of §2 against your own eight archived postings rather than the illustrative composites.

## 2. Worked answer

### 2.1 The map (illustrative composite)

The table below is what a reviewer-ready map looks like when filled in. Every row cites the one line in the posting that locked its placement; a reviewer should be able to click back to the archived posting and find that line.

| # | Posting (archived) | Category | Centre-of-gravity (one line from the JD) | Stack mentioned |
|---|---|---|---|---|
| A | *Posting A — "Voice AI Engineer" at a pure-play ASR vendor* (illustrative) | 1. Pure-play | "Build reference voice-agent integrations against our streaming ASR and publish latency benchmarks." | Deepgram Nova / AssemblyAI Universal-Streaming-style ASR; ElevenLabs/Cartesia TTS; GPT-4o / Claude; WebSocket + WebRTC |
| B | *Posting B — "Developer Experience Engineer (Voice)" at a TTS lab* (illustrative) | 1. Pure-play | "Own the end-to-end voice-agent SDK examples; measure time-to-first-audio on each demo." | Cartesia Sonic / ElevenLabs TTS; LiveKit Agents or Pipecat; OpenAI Realtime |
| C | *Posting C — "Senior Voice Agent Engineer" at an orchestration framework vendor* (illustrative) | 2. Orchestration | "Design and extend pipeline primitives — VAD, turn-taking, interrupt handling — in `livekit-agents` / `pipecat-ai`." | LiveKit / Pipecat; Opus-over-WebRTC; Silero VAD; provider-plugin surface |
| D | *Posting D — "Conversational Voice AI Engineer" at a CPaaS vendor* (illustrative) | 3. Telephony | "Bridge Twilio Media Streams and ConversationRelay to realtime LLM providers; hit five-nines on carrier reliability." | Twilio Media Streams / ConversationRelay; µ-law / PCMU; WebSocket bridge; OpenAI Realtime or cascaded |
| E | *Posting E — "Realtime Voice Engineer" at a frontier lab* (illustrative) | 4. Frontier lab | "Ship first-party realtime voice demos on top of the Realtime API; profile and reduce TTFT at the provider boundary." | OpenAI Realtime API / Gemini Live; WebRTC; in-house S2S model |
| F | *Posting F — "Voice AI Engineer, Contact Center" at a CX agent company* (illustrative) | 5. Applied-voice | "Own barge-in, latency, and compliance for an inbound-triage voice agent deployed at Fortune-500 contact centres." | Deepgram or Google STT; ElevenLabs TTS; Anthropic / OpenAI LLM; Pipecat or LiveKit; Twilio PSTN |
| G | *Posting G — "AI Engineer — Ambient Scribe" at a clinical documentation company* (illustrative) | 6. Vertical | "Build the realtime transcription + structured-note pipeline that HIPAA-scoped data flows through; work with clinicians on terminology." | Clinical-tuned ASR (provider or self-hosted); Claude / GPT; EHR write-back; HIPAA-compliant infra |
| H | *Posting H — "Voice Engineer, Sales Conversations" at a sales-intelligence company* (illustrative) | 6. Vertical | "Ship live-coaching whispers on live sales calls; measure insertion latency and whisper visibility to the rep only." | Streaming ASR; low-latency TTS; LLM for coaching prompts; CTI integration |

Category coverage: 1 (A, B), 2 (C), 3 (D), 4 (E), 5 (F), 6 (G, H). Six of six covered — a passing map needs at least four of six.

### 2.2 Two ambiguous placements, justified

**Posting B — "Developer Experience Engineer (Voice)" at a TTS lab.** On title alone this looks like a DevRel / community role (category not in the frame). The reason it lands in category 1 (pure-play) is that the one-liner — "own the end-to-end voice-agent SDK examples; measure time-to-first-audio" — describes the voice-agent-engineer deliverable (reference pipeline on top of the house model). To move it out of category 1 it would have to be true that the SDK examples are owned by someone else and this role is only writing blog posts and conference talks about them; the JD as cited does not say that.

**Posting F — "Voice AI Engineer, Contact Center" at a CX agent company.** Could be read as category 3 (telephony) because the deployment target is a contact centre and Twilio is in the stack. The reason it lands in category 5 (applied-voice product) is that the product the company sells is the inbound agent, not the telephony connectivity — the Twilio dependency is a sub-component, not the centre of gravity. To move it to category 3 it would have to be true that the company's core product is the CPaaS layer itself, which it isn't.

### 2.3 One-page synthesis

**Which category generates the most volume in this sample?** In the illustrative composite above, categories 1 and 6 tie at two postings each. The real answer in your map is a number out of eight, defended from the actual postings you collected — not a brand feel. **A reviewer should check that the ratio is reported honestly, not rounded to flatter a hypothesis.**

**Which category most emphasises latency, which most emphasises compliance, and does that match Chapter 1's archetype questions?** In this worked example:

- *Latency-heavy categories* — 1 (pure-plays) and 4 (frontier labs). Their one-liners mention "time-to-first-audio", "TTFT", "latency benchmarks". That matches Chapter 1's archetype interview question for those categories ("Where do you spend your latency budget?").
- *Compliance-heavy categories* — 5 (applied-voice contact-centre) and 6 (vertical, especially clinical). Their one-liners mention HIPAA, barge-in, recording. That matches Chapter 1's archetype for applied / vertical ("A customer complains the bot interrupts them, misses medical terminology, and ‘sounds robotic'").

When the mapping *doesn't* match the archetype, that's the interesting part of the synthesis — write a sentence on why (e.g., "the vertical posting I expected to be compliance-heavy was actually pure latency because the customer already does the compliance review themselves").

**Which category would I preferentially target, and why?** This answer is personal — a reviewer is looking for a reason that references *your own* background, not a generic statement like "applied-voice is growing." A defensible shape is: "Target category 5 (applied-voice) because (a) the loop can be deeply L35 end-to-end — pipeline design, barge-in, compliance, eval — rather than being pulled toward either (b) L30 text-LLM work at a non-audio company, or (c) L40 model-research work at a frontier lab. The latency-and-compliance archetype (Chapter 1) matches the modules in this track most directly."

### 2.4 One posting broken into "this / adjacent / red flag"

Worked on **Posting G** (ambient scribe, illustrative).

- *This is us (L35 voice-agent engineer).* "Design and ship the realtime streaming transcription + structured-note pipeline, including endpointing and medical-terminology handling." Centre of gravity is the pipeline — architecture brief, latency budget, pipeline design, telephony integration doc, eval bundle. Direct match for Chapters 2 and 5.
- *This is adjacent (not us).* "Train custom ASR models for paediatric cardiology accents." That is `training-pipeline-engineer` and `ASR research scientist` work (Chapters 3 and 4). A voice-agent engineer *informs* the data needs and *integrates* the resulting model; they do not own the training objective or the dataset curation. If the posting makes this the primary deliverable, the role is mis-labelled.
- *This is a red flag.* "Also responsible for clinical validation of generated notes against physician review." That's a clinical-informatics role with a safety-evidence programme behind it. A voice-agent engineer *implements* the review flow (routing, logging, dissent capture) but does not *own* the clinical validation evidence. Listing both the pipeline *and* the clinical-evidence programme as single-engineer ownership is a sign the hiring team hasn't decided whether they're hiring one person or three.

## 3. Validation steps

A finished map should pass all of the following checks before you submit it.

1. **Archive check.** Every posting has an archive URL (Wayback, archive.today, PDF) that still resolves. The map's centre-of-gravity quote matches a visible sentence in the archived posting.
2. **Spread check.** At least four of the six categories (Chapter 1) are represented. If five or six are represented, note the edge cases explicitly — multi-category placements erode the frame.
3. **Centre-of-gravity test.** For each row, the one-liner is a *quote or paraphrase from the JD*, not a category description. "Latency-focused pipeline work" is a category statement; "P95 end-to-end response under 1.5 s on real calls" is a centre-of-gravity line.
4. **Ambiguity declaration.** Two postings are called out as hard to categorise, with the alternative category and the fact that would flip the decision named explicitly.
5. **Synthesis grounding.** The volume / latency / compliance answers cite actual rows, not category stereotypes.
6. **This / adjacent / red-flag breakdown.** One posting is dissected; the "this is adjacent" item points to a specific adjacent role from Chapter 3 or 4; the red flag is justified by scope (one person doing the job of several), not by snark.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Sample size & archiving | 20 % | 8+ real postings; each has an archive link | 12+ postings, archived at the time of collection, timestamps visible |
| Category spread | 15 % | 4+ of 6 categories represented | All 6 represented, with ties broken honestly |
| Centre-of-gravity precision | 20 % | Each row cites a JD sentence, not a category stereotype | One-liners ≤ 15 words and interchangeable across reviewers |
| Ambiguity handling | 15 % | 2 ambiguous postings called out with the flip-fact | Alternatives come with "what would have to be true" sentences grounded in the JD text |
| Synthesis quality | 15 % | All three synthesis questions answered with numbers / examples, not brand feel | Mismatches with the Chapter 1 archetype questions are called out and explained |
| This / adjacent / red-flag breakdown | 15 % | One posting broken out; adjacent item names a specific adjacent role (Chapters 3–4); red flag is scope-based | Red flag is actionable: "I would ask at phone-screen whether X is actually one role or three" |

A submission scoring < 70 % in aggregate is a *fail-and-redo*; it indicates the learner has substituted brand recognition for posting-text reading.

## 5. Common mistakes

- **Mapping by brand, not by posting.** "Deepgram = category 1" is a shortcut that fails as soon as Deepgram opens a telephony-partnerships engineer role. Read the posting.
- **Over-counting pure-plays.** Because the pure-play category is the most *visible* one in the curriculum, learners tend to collect too many of them and under-sample categories 3 and 6. The acceptance criterion is spread across four categories specifically to catch this.
- **Confusing "Conversational AI Engineer" roles.** At a contact-centre vendor this is almost always category 5 (applied-voice), not a conversation-designer role — Chapter 1 flags this explicitly.
- **Confusing "AI Engineer — Voice" at a non-audio company.** Often an applied-AI role (L30) that happens to call an API. Chapter 1 flags this as a trap.
- **Picking a soft red flag.** "Requires React" on a backend voice role is a stack mismatch, not a red flag about role design. The red flag the exercise cares about is scope confusion — one role trying to span L30, L35, L40, and a non-engineering discipline.
- **Writing the synthesis as marketing copy.** "Voice is the hottest category" is not a synthesis. "Four of eight postings in my sample concentrate on category 5, which matches the Chapter-1 archetype; the two postings with the shortest latency budgets are both in category 1" is a synthesis.

## 6. References

Primary — the chapters this exercise unpacks:

- `lessons/mod-101-voice-agent-position/01-the-voice-agent-engineer-landscape.md` — the six categories and the archetype interview questions.
- `lessons/mod-101-voice-agent-position/02-position-on-the-ladder.md` — why "L35 voice-agent engineer" is the right level and what distinguishes it from L30 roles (whose postings are the main source of false positives).

Vendor pages and open-source projects named in Chapter 1 (use when locating postings; archive at the time of collection):

- Deepgram — `https://deepgram.com/`
- AssemblyAI — `https://www.assemblyai.com/`
- ElevenLabs — `https://elevenlabs.io/`
- Cartesia — `https://cartesia.ai/`
- LMNT — `https://lmnt.com/`
- Hume — `https://hume.ai/`
- LiveKit (`livekit-agents`) — `https://livekit.io/`
- Pipecat (`pipecat-ai`) — `https://github.com/pipecat-ai/pipecat`
- Vocode — `https://vocode.dev/`
- Retell AI — `https://www.retellai.com/`, Vapi — `https://vapi.ai/`, Bland AI — `https://www.bland.ai/`
- Twilio Programmable Voice / Media Streams / ConversationRelay — `https://www.twilio.com/docs/voice`
- Vonage Voice API — `https://developer.vonage.com/en/voice/overview`
- Zoom Phone / Contact Center — `https://developers.zoom.us/`
- OpenAI Realtime API — `https://platform.openai.com/docs/guides/realtime`
- Google Gemini Live API — `https://ai.google.dev/gemini-api/docs/live`
- Kyutai (Moshi, Mimi) — `https://kyutai.org/`

Note on vendor career-page URLs. Vendor careers URLs are moved by SEO teams frequently; confirm the current archive-friendly URL of each vendor's careers page at exercise-run time rather than copying forward from this reference.

Source policy note. The six-category frame itself is a local-to-this-curriculum construct documented in Chapter 1; it is not claimed to be industry-standard. The adjacent-role boundary (ASR research vs. voice-agent engineer, IVR vs. voice-agent engineer, etc.) is documented in Chapter 3 and referenced in the red-flag breakdown. External standards (OWASP ML Top 10, MITRE ATLAS, NIST AI RMF) are not load-bearing for this exercise and are not invoked here.
