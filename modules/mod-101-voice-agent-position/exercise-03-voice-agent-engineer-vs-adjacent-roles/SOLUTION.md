# exercise-03 — Voice Agent Engineer vs. Adjacent Roles — Reference Solution

## 1. Solution overview

This exercise produces two artefacts anchored to one specific voice feature:

- **A six-row comparison table** across the adjacent roles from Chapter 3 (ASR research scientist, TTS research scientist, conversation designer, voice-UX designer, IVR / CTI developer, contact-centre operations engineer) — each row naming (a) the question that role is answering, (b) the question *you* are answering, and (c) the handoff artefact between you.
- **A boundary-breakage postmortem (300–500 words)** describing a realistic — clearly-labelled fictional — incident where the boundary with one of those roles was broken and the feature suffered. Action items are process / artefact fixes, not individual blame.

The worked example anchors on **an ambient clinical-documentation feature** (the same shape as products in Chapter 1 category 6): an in-exam-room scribe that ingests a clinician–patient conversation, produces a structured note, and writes back into an EHR. The most uncomfortable requirement is **HIPAA-grade PHI handling across paediatric accents and heavy clinical terminology, under a two-second end-of-utterance budget before the clinician perceives lag.**

## Implementation

The worked answer below is the implementation of this exercise: the two artefacts a reviewer should see (six-row comparison table anchored on one named feature; a 300–500-word boundary-breakage postmortem with structural action items), followed by validation steps, rubric, and common mistakes. To execute the exercise yourself, follow the shape of §2 against your own chosen voice feature rather than the illustrative ambient-scribe case.

## 2. Worked answer

### 2.1 Feature pick and the "uncomfortable" requirement

**Feature.** An ambient clinical-scribe voice agent, deployed in paediatric cardiology exam rooms, producing SOAP-note–structured output written back to the EHR.

**One most uncomfortable requirement.** Accurately transcribing paediatric-cardiology drug names (e.g., β-blockers, inotropes, antiarrhythmics with multi-syllable brand and generic names), spoken by physicians with varied accents in the presence of child-voice and medical-equipment noise, under a 2-second end-of-utterance lag before the scribe feels disruptive, and with HIPAA-compliant redaction and retention. This requirement touches almost every adjacent role at once — which is exactly why it anchors the exercise.

### 2.2 Six-row comparison table

| Adjacent role | Their centre of gravity on this feature | Your centre of gravity on this feature | Handoff artefact |
|---|---|---|---|
| **ASR research scientist** | Which pre-training data distribution, decoder topology, and biasing mechanism best covers paediatric-accented English and cardiology drug terminology without regressing general WER? Would a domain fine-tune or an LM-biasing module pay off more? | Which provider / model to integrate today; what the product-level WER on *this product's* corpus must be; whether the current error rate is launch-blocking. | A **model-selection memo** — the current product-level WER breakdown on an in-domain gold set, the error-category histogram (drug names, paediatric prosody, cross-talk), and the acceptance threshold. ASR research reads this to decide if a training programme is worth starting. |
| **TTS research scientist** | Which acoustic model and vocoder produce natural clinician-register speech at low latency, and which prosody controls expose the right knobs for the product to use? | Which voice to ship, when the scribe should speak back vs. stay silent, SSML / markup conventions, and streaming chunk boundaries that don't clip at phoneme boundaries. For an ambient-only scribe TTS is minimal; for a "read back the note" variant it is central. | A **voice selection and markup spec** — one page naming voices, markup tags, latency budget per sentence, and clipped-boundary examples for research to debug. |
| **Conversation designer** | The clinician-facing dialogue: how the scribe introduces itself, how it discloses recording, how it recovers from mishearings, how it asks for confirmation of a dose. Specific wording approved by clinical leadership and legal. | How those dialogue states are rendered into pipeline behaviour: barge-in windows while the clinician is speaking, when to re-prompt, how to hold a confirmation turn open while the clinician looks at a monitor. | A **dialogue-state contract** — a tagged script with turn-level annotations (what can be interrupted, what must be confirmed) that the engineer maps to orchestration code. |
| **Voice-UX designer** | The multimodal UX: when does the scribe stay silent and reflect on a paired screen vs. speak; how to signal recording vs. not-recording; how to let a clinician pause / resume with a glance or a single word. | Making those decisions implementable: timers on the orchestration layer, screen-side status events, the single-word "pause" wake-phrase, fallback to touch. | A **UX-to-engineering spec** — a state diagram of visible / audible states and the pipeline events that drive them. |
| **IVR / CTI developer** | For an ambient scribe this role is usually absent, because there is no PSTN leg. If the product is extended to include tele-health or call-based follow-ups, this role owns the SIP trunking, codec negotiation, DTMF handling, and STIR/SHAKEN attestation. | The voice agent that sits on the far side of the carrier hop — once audio is in your pipeline, it's yours. For an in-room scribe, your job is to ensure the microphone edge doesn't suddenly become a telephony edge without an integration doc. | A **telephony integration doc** (Chapter 5 §4) — only required if / when the product adds a call-based surface. For the pure in-room scribe, a one-line "no telephony surface" statement in the architecture brief suffices. |
| **Contact-centre operations engineer** | Running the live operations: workforce scheduling for clinicians-as-users (training, feedback), QA review of produced notes, coaching when the scribe underperforms, compliance recording policy, PHI retention and deletion. | The product itself — the pipeline behaviour, the eval bundle that catches regressions before they hit a live exam room, the operations runbook the operations engineer reads when something is wrong. | An **operations runbook** (Chapter 5 §8) — signatures for common failures, dashboards for latency and error rate, escalation phone tree. |

Each cell is kept to ≤ 2 sentences as the exercise requires. The handoff-artefact column names a concrete document, not "collaboration."

### 2.3 Boundary-breakage postmortem (illustrative, fictional)

> *This postmortem is fictional. No specific customer, vendor, or product is referenced. The scenario and numbers are pedagogical.*

# Incident

On 2026-03-12, the ambient clinical-scribe feature began producing structured notes with a measurable regression in paediatric-cardiology drug-name accuracy at a pilot site. Error rate on a curated 500-utterance gold set rose from 3.1 % to 9.7 % over a two-day window, triggering the regression gate in the eval bundle and pulling the launch on 2026-03-14.

# What happened

In the sprint leading up to the regression, the voice-agent engineer identified that the hosted ASR's off-the-shelf model mis-recognised several high-frequency paediatric cardiology drug names. Rather than filing a model-selection memo with the ASR research scientist (per the handoff contract), the engineer directly uploaded a word-boost / custom-vocabulary list to the provider's API, including phonetic hints. This reduced errors on the specific drug names the engineer had audited and passed the engineer's spot-check eval.

Simultaneously, the ASR research scientist — unaware of the custom-vocabulary change — shipped a domain-adapted model variant for paediatric cardiology into the provider's endpoint that the integration pointed at. The two biasing systems interacted. On a subset of utterances (particularly those where a drug name was adjacent to a dosage number), the compound biasing produced worse results than either alone: the dosage was being over-biased toward phonetically-similar drug names, and vice versa.

The voice-agent engineer's acceptance tests used a gold set that was built from the drug names the engineer had audited, which overlapped the custom-vocabulary list. The regression was invisible on that set. It appeared immediately on the broader, periodically-refreshed in-domain gold set that the eval bundle ran during CI.

# Root cause

Two uncoordinated changes to the ASR biasing path — one at the provider's model layer by ASR research, one at the provider's custom-vocabulary layer by the voice-agent engineer — were deployed without either engineer knowing about the other's change. The root cause is *not* a bad decision by either engineer in isolation; it is the **missing handoff artefact**. The Chapter-3 boundary (research owns methodology, engineer owns integration, both share a model-selection memo) was collapsed when the engineer substituted an ad-hoc fix for the memo.

# What should have happened

On identifying the drug-name error pattern, the voice-agent engineer should have produced the model-selection memo — current product-level WER on the in-domain gold set, error-category histogram, acceptance threshold — and filed it with ASR research. ASR research would have disclosed their in-flight domain-adaptation work. The two of you would have decided *jointly* between (a) the engineer-side custom-vocabulary fix, (b) the research-side domain-adapted model, or (c) both, with an explicit coordination point. Either way the gold set used for validation would have been the broader in-domain set, not the drug-name subset the engineer had personally audited.

# Action items

1. Add the **model-selection memo** as a required artefact in the voice-agent engineer's definition of done for any ASR tuning change beyond default provider settings. Owner: voice-agent-engineer lead. Due: next sprint.
2. The ASR research scientist posts any domain-adaptation model shipment to a shared integration-changelog channel, with the affected provider endpoint named explicitly. Owner: ASR research lead. Due: next sprint.
3. CI runs the eval bundle against the **in-domain gold set only** for launch-gate decisions; the engineer's spot-check set is disallowed as evidence. Owner: voice-agent engineer. Due: within one week.
4. Add a quarterly "biasing-surface review" to the operations runbook (Chapter 5 §8): inventory every active biasing mechanism (custom vocab, LM biasing, prompt hints, domain fine-tune) and name the owner of each. Owner: engineering manager. Due: before next launch attempt.

(Word count: within the 300–500 target.)

Action items are process / artefact fixes (a required memo, a changelog channel, a gold-set rule, a periodic review). No individual is named; both engineers acted reasonably given what they could see.

## 3. Validation steps

A finished submission should pass each of the following checks.

1. **Named feature with one explicit uncomfortable requirement.** Not "voice agent" generically; a specific product with a named constraint.
2. **Six rows, each covering an adjacent role from Chapter 3.** Not five, not seven. If the IVR / CTI row is marked N/A, that is a valid cell content — but the row must still exist.
3. **"Their question" is a *question*, not a job-description sentence.** "Trains the model" is a job-description sentence; "Which pre-training data distribution best covers paediatric-accented English?" is a question.
4. **Handoff artefact is a named document / interface, not a verb.** "Collaboration" fails; "model-selection memo" passes; "a tagged dialogue script at `prompts/scribe-v3/*.yaml`" is even better.
5. **Postmortem has the five required headings.** `# Incident` / `# What happened` / `# Root cause` / `# What should have happened` / `# Action items`.
6. **Word count in 300–500.** The compression constraint is pedagogical; respect it.
7. **Root cause is structural, not individual.** No sentence of the form "engineer X did Y badly." If a reader can swap in the next engineer and have the same incident, the root cause is structural.
8. **Action items are artefact / process fixes.** Each action item is a required document, a changelog, a rule, a review, or a role assignment.
9. **Scenario is clearly labelled as real or fictional.** If fictional, say so at the top; no implicit claim that an invented incident actually happened.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Feature specificity | 10 % | Product and uncomfortable requirement named concretely | Multi-clause constraint that crosses ≥3 adjacent roles |
| Table completeness | 20 % | All 6 rows present, each cell ≤ 2 sentences | Handoff-artefact column cites a path or template, not a name |
| Question framing | 15 % | Each "their" cell is phrased as the question that role is answering | The question is concrete enough that another engineer could try to answer it |
| Handoff artefacts | 15 % | Each handoff names a document / interface, not "collaboration" | Matches a Chapter-5 deliverable where applicable |
| Postmortem structure | 15 % | All five headings; 300–500 words | Scenario is realistic and declaratively labelled real/fictional |
| Root cause quality | 15 % | Root cause is structural — missing artefact, missing changelog, misaligned definition of done | Names the specific Chapter-3 boundary that collapsed |
| Action items | 10 % | Process / artefact fixes, not individual blame; each has an owner and a timeframe | Fixes are durable (apply beyond this one incident) |

A submission scoring < 70 % is a fail-and-redo. The most common sub-70 failure: a postmortem whose root cause reads as "engineer made a mistake."

## 5. Common mistakes

- **Writing the feature as a slogan.** "An AI voice agent for healthcare" is not a feature. Name the deployment surface, the input modality, the user, and the compliance regime.
- **Using "research" and "engineering" as vibes.** The exercise is specifically about the *question* each role is answering. If "their question" and "your question" could be swapped without losing meaning, the row is not doing work.
- **"Handoff: alignment meeting."** Meetings are not artefacts. Artefacts persist after the meeting ends.
- **Postmortems that blame an engineer.** This is explicitly called out in the exercise text and in Chapter 4's ethos. The postmortem is a boundary-breakage artefact; the fix is a boundary, not a performance-management action.
- **Postmortems with no "what should have happened".** This is the diagnostic section. Without it, there is nothing to turn into an action item.
- **Action items that are "be more careful next time."** That is not an action item. Action items are new documents, new reviews, new gates, new owners.
- **Padding to the word limit.** 300–500 is a range, not a quota. If 320 words covers it cleanly, don't stretch.

## 6. References

Primary — chapters this exercise unpacks:

- `lessons/mod-101-voice-agent-position/03-adjacent-voice-ecosystem-roles.md` — the six adjacent roles and the sharp-boundary definitions.
- `lessons/mod-101-voice-agent-position/04-generalist-stance-and-escalation.md` — the deferral-contract pattern and why structural fixes beat individual fixes.
- `lessons/mod-101-voice-agent-position/05-deliverables-across-the-voice-feature-life-cycle.md` — the model-selection memo, telephony integration doc, and operations runbook named as handoff artefacts.

Domain-specific anchors for the worked example (ambient clinical scribe):

- HIPAA (US clinical privacy). Primary source: U.S. Department of Health & Human Services — `https://www.hhs.gov/hipaa/index.html`.
- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — used indirectly when the postmortem's action items name organisational artefacts (changelogs, periodic reviews, launch gates), which is the risk-management-function family in the AI RMF.

Note on forward cross-links. The ambient-scribe reference stack used in this worked example may be revisited in a later safety / compliance module; where that is the case, the two solutions should be cross-linked at that time. For this exercise, the Chapter-3 boundary definitions are sufficient grounding.

Source policy note. The postmortem is explicitly fictional and labelled as such; no real customer, provider, or incident is named. The feature description draws on companies listed in Chapter 1 (Suki, DAX, Abridge, Ambience) only to the extent that they are mentioned as practitioners in the space — no specific product behaviour is attributed to any of them. OWASP ML Top 10 and MITRE ATLAS are not load-bearing for this exercise and are not invoked.
