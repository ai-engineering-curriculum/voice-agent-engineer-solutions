# exercise-04 — When to Stay Generalist, When to Escalate — Reference Solution

## 1. Solution overview

This exercise is a timed decision drill. Each of six vignettes gets a four-field decision log (Call / Why / First action / Signal to re-decide) under 10 minutes of pressure, then one 300-word reflection after a revision pass.

The reference solution below gives:

- One defensible decision log per vignette. For each one it names which of the five escalation scenarios from Chapter 4 (if any) is being invoked.
- A worked 300-word reflection that revises at least one call.
- The rubric a reviewer should hold learners to — including the structural check that *not all six* calls end up either "own" or "escalate" (Chapter 4 explicitly flags this as mis-calling).

Target balance: 2 "own", 3 "escalate", 1 mixed (own-now, escalate-if-condition). The exercise's acceptance criteria require "at least two entries labelled own and at least two labelled escalate."

## Implementation

The worked answer below is the implementation of this exercise: the six decision-log entries (four fields each), a target-balance summary across own/escalate/mixed, and a revision-pass reflection, followed by validation steps, rubric, and common mistakes. To execute the exercise yourself, run the six vignettes under 10-minute pressure first, then produce the reflection after a revision pass rather than editing the first-pass entries in place.

## 2. Worked answer — six decision-log entries

### Vignette 1 — Tail-latency mystery

- **Call.** Escalate to `ai-infra-inference-engineer`. *This is Chapter 4 Scenario 1 almost verbatim.*
- **Why.** P95 is bimodal and traces are clean on your side; the slow tail lives in LLM-provider TTFT. Fixing it requires levers you don't own — serving tier, batching behaviour, provider negotiation. If the fix turns out to be on your side after all (e.g., you were sending oversized context on 5 % of turns), the inference engineer's first look will catch it faster than another week of you tracing by yourself.
- **First action (next hour).** Package a reproducer: 200-trace bundle from the last 24 h with hop-by-hop timestamps, request IDs, provider request IDs where returned, and a short note stating the measured P95 vs. budget. File the deferral contract (Chapter 4 §deferral) in a ticket; cc your engineering manager.
- **Signal to re-decide.** If the inference engineer comes back with "the TTFT is fine on the provider side, your traces must be mis-measured," your call flips back to "own" and you re-audit your tracing before anything else.

### Vignette 2 — Weird eval results

- **Call.** Escalate to `ai-eval-engineer`. *Chapter 4 Scenario 2.* (Peer is on vacation — this means a delay in the handoff, not a change in who owns the problem.)
- **Why.** You have a judge-methodology concern (length bias in LLM-as-judge is a known failure mode). Even if your hypothesis is right, the fix is calibration + methodology (paired comparisons, controlling for length, inter-rater agreement) — a discipline the eval engineer owns.
- **First action (next hour).** Document the suspected length bias with specific examples: pairs where the judge preferred the longer response despite worse content. Pin the eval-engineer's return date. In the interim, stop merging on the current judge's output as a launch gate and switch to pre-calibration manual review on contested configs.
- **Signal to re-decide.** If the eval engineer returns and says "this looks correct on my first read — the long response actually is better, the voice difference is confounding your intuition," the call stays escalated but widens: you may need a multimodal judge (Chapter 2 §vs. ai-eval-engineer). You do not revert to the old call.

### Vignette 3 — Medical-term accuracy

- **Call.** Mixed — **own the first mitigation tier now, escalate the training work if the first tier is insufficient.** *This is the explicit layering Chapter 4 Scenario 3 describes.*
- **Why.** The provider's custom-vocabulary / word-boost is the first-tier mitigation and is your job. You have tried it; it reduced error but not enough. The next tier (provider-side domain fine-tuning, which you've never done) is `training-pipeline-engineer` work. Before escalating, exhaust the levers that are still yours: in-prompt LM biasing where the provider supports it, post-hoc LLM correction, swapping to a provider whose off-the-shelf model is better on your corpus. If none of those close the gap, escalate.
- **First action (next hour).** Build the error histogram on your in-domain gold set: which terms fail, at what rate, under what audio conditions. Try at least one additional lever today (post-hoc correction pass is cheapest). File a Chapter-4-style deferral contract draft for the escalation case so it's ready if needed.
- **Signal to re-decide.** When the additional levers close less than half the remaining gap to the clinical acceptance threshold, convert "mixed" into full escalate — hand the error histogram, the labelled corpus, and the threshold to the training-pipeline engineer.

### Vignette 4 — Deepfake complaint

- **Call.** Escalate to `security-learning` track / `agentic-safety-engineer` jointly; keep legal / FDE on the thread. *This straddles Chapter 4 Scenarios 4 and 5.*
- **Why.** You cannot adjudicate the external-cloning claim from your logs alone, and even if you could, the response is a security / safety / legal programme question, not a pipeline question. Your scope is to (a) confirm whether your product *could* have been the source (log audit, authentication-event audit, voice-model sample comparison), (b) ship any product-side mitigations that reduce future risk, and (c) write the artefacts the specialists need.
- **First action (next hour).** Open a Sev-2 incident. Scope the audit — which logs could show the call (if the cloning claim is traceable), which authentication events in that customer's account around the alleged incident time. Write the one-page incident packet (claim, evidence available, evidence requested, product-side mitigations currently in place, 48-hour deadline) and hand it to the security specialist. Do *not* individually promise the customer an answer in 48 h — that's an FDE / legal / exec conversation.
- **Signal to re-decide.** If the audit concludes the voice was not synthesised by your product and no customer account was compromised, you stay escalated on the response but downgrade the product-side urgency. If the audit is *inconclusive* (which is likely), you stay escalated and start the broader security review of voice-authentication flows.

### Vignette 5 — Barge-in regression

- **Call.** Own. *This is squarely in the "80 % of voice work you own end-to-end" zone from Chapter 4.*
- **Why.** VAD behaviour, silence thresholds, and barge-in handling are the voice-agent engineer's core craft. You shipped the VAD change; your constants were tuned for the old VAD; nothing here requires methodology, training, or security specialists. Escalating this would be exactly the over-escalation failure mode Chapter 4 warns about — asking a specialist to make a pipeline decision that is yours.
- **First action (next hour).** Reproduce on your eval harness by sweeping silence-threshold and min-utterance-duration parameters against the pre-regression numbers. Shortlist 2–3 candidate settings that recover the metric without silence-dragging at the other end. Expand the barge-in eval to include noisy-condition traces so the revalidation covers more than your clean set.
- **Signal to re-decide.** If the sweep does not recover the metric at any threshold, or reveals that the new VAD's false-positive characteristic is qualitatively different from the old one on specific audio conditions (child voices, bilingual interjections, overlapping speech), escalate to `agentic-ai-engineer` or ASR-side peers depending on the signature. Changing your call because threshold-tuning didn't close the gap is not failure — it's the system working.

### Vignette 6 — Capability creep

- **Call.** Escalate to `agentic-safety-engineer`; keep legal in the loop; push back on Product via your engineering manager. *Chapter 4 Scenario 5.*
- **Why.** "Explain medication interactions pulled from the patient's EHR" is a capability-tier decision. Even if you *could* implement it tomorrow, the question of whether it should be shipped belongs to the safety programme (dangerous-capability evaluation, refusal policy, liability review). Your guardrail refuses it today; changing the guardrail without a policy change is the exact mis-call the chapter names.
- **First action (next hour).** Write the capability-expansion artefact: a one-page scope definition (what exactly is being asked for, with representative transcripts — common case and worst case), the current guardrail behaviour, the specific business pressure (competitor claim), and the open question (what new policy would permit this, if any). Hand to `agentic-safety-engineer` for the risk tier; cc your engineering manager so Product gets the "no" from a level that can hold it.
- **Signal to re-decide.** If the safety engineer returns a policy — e.g., "allowed with a licensed-pharmacist review loop and explicit disclaimers, outside of urgent-care scenarios, and with human-in-the-loop on doses" — your call flips to "own the implementation." If the policy is "no, not in the current risk tier," the call flips to "own the respectful decline back to Product with alternatives."

### Decision-log balance

- "Own" calls: Vignette 5 (and the first-tier portion of Vignette 3).
- "Escalate" calls: Vignettes 1, 2, 4, 6 (and the second-tier portion of Vignette 3).
- Mixed: Vignette 3.

Balance is 2 full-own, 4 full-escalate, 1 mixed. Satisfies the "at least two / at least two" acceptance rule from the exercise.

## 3. Worked 300-word reflection

> A second-pass read of these six decisions reveals two calls I'd now soften.
>
> First, Vignette 2 (eval judge bias). My first-pass escalate was correct in direction but too fast. The ten-minute pressure made me reach for the specialist because "LLM-as-judge length bias" is a famous category of problem. On revision, I think the right sequencing is: spend an hour designing a controlled test — same content, two voices with different pacing, length-normalised scoring — before handing it off. The specialist will do a better job with a pre-made controlled example than with "I think length is confounding." The call stays "escalate" but the first-action gets a small controlled experiment in front of the handoff.
>
> Second, Vignette 4 (deepfake complaint). My first-pass instinct was to make this a security escalation, and I stand by that. On revision, I noticed I did not initially name "do not promise the customer a 48-hour answer yourself" — I wrote that only when thinking through first-actions. In a weekly review that's exactly the point I would be most useful: naming the customer-facing promise as an FDE / legal / exec decision, not an engineer's call. The call is unchanged; the phrasing of *what stays yours* is sharper.
>
> The one call I most want to resist softening is Vignette 5. The barge-in regression is the kind of problem that feels tempting to escalate because "a specialist could maybe tune VAD better." That is exactly the over-escalation failure mode Chapter 4 warns about. The honest answer is: this is the voice-agent engineer's core craft; own it.
>
> Taken together, the reflection mostly tightens phrasing rather than flipping calls — which I interpret as the first-pass instincts being closer to correct than my initial self-doubt suggested.

(298 words; within the 300-word target.)

## 4. Validation steps

A finished submission should pass each of the following checks.

1. **Six entries, each with four fields.** Call, Why, First action, Signal to re-decide. No field left blank. "It depends" is never the whole content of Call — Chapter 4 explicitly rejects this.
2. **Balance.** At least two Owns and at least two Escalates. All-own or all-escalate is a mis-call, as the exercise acceptance criteria state.
3. **Each escalation names a target role.** Not "escalate" as a verb. "Escalate to `ai-infra-inference-engineer`" or similar — the specific peer.
4. **First-action is concrete and time-boxed to the next hour.** Not "think about it" — a named artefact to produce or a named test to run.
5. **Signal-to-re-decide is a measurement or event, not a feeling.** "P95 doesn't drop below 1.5 s after the provider swap" is a signal; "if it's still slow" is not.
6. **Reflection is 300 words (±10 %), revises at least one call.** The reflection's job is self-critique; a reflection that reads as validation isn't doing the work.
7. **No vignette left as pure "it depends."** If the answer genuinely depends on a fact you don't have, state the fact and say which way each value would push the call.

## 5. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Field completeness | 15 % | All 4 fields populated on all 6 vignettes | First-action names an artefact at a path |
| Balance check | 10 % | ≥ 2 own and ≥ 2 escalate; no all-own or all-escalate | Mixed / conditional call used at least once with the condition named |
| Escalation targets | 15 % | Each escalation names the specific target role | Target matches the Chapter-4 scenario explicitly |
| First-action quality | 20 % | Concrete, time-boxed, produces an artefact | Includes the reproducer / deferral-contract filing step |
| Signal-to-re-decide quality | 20 % | A measurement or an event; not "if it feels wrong" | Signal is numeric or logged (threshold-crossing, event-counted) |
| Reflection | 20 % | 300 words, revises at least one call with a stated reason | Names the chapter concept that caused the revision (e.g., over-escalation failure mode) |

A submission scoring < 70 % is a fail-and-redo. The most common sub-70 failure: four blank or near-blank "Signal to re-decide" fields.

## 6. Common mistakes

- **All six escalations.** Signals that the learner is treating the voice-agent engineer as a triage desk rather than an L35 generalist. Chapter 4 is explicit: 80 % is yours.
- **All six owns.** The reverse failure mode — the learner is defaulting to individual heroism. The point of the drill is that some problems are genuinely not yours.
- **"Signal to re-decide: when I feel it's not working."** This field is the most important one in the exercise, and it is a measurement, not a vibe. If you can't name a threshold, you haven't thought about how you'd know.
- **Treating the eval-engineer-is-on-vacation signal as a call-changer.** It is not — vacation changes timing, not ownership. Chapter 4's boundaries are durable; your calendar is not.
- **Escalating the barge-in regression.** The most common error on Vignette 5. The silence-threshold tune-up is exactly the work this role exists to do.
- **Owning the deepfake complaint.** The reverse error on Vignette 4. You are not qualified to adjudicate a biometric-spoofing claim; that's a security / safety / legal programme.
- **Writing the reflection as validation.** "On reflection my first instincts were correct" is almost never the whole story. The reflection must revise at least one call, per the exercise's acceptance criteria.
- **Padding to the word limit.** 300 words is the target, not the floor. 280–310 is fine. 450 is not.

## 7. References

Primary — chapters this exercise unpacks:

- `lessons/mod-101-voice-agent-position/04-generalist-stance-and-escalation.md` — the default-you-own stance, the five escalation scenarios, the deferral contract shape, the over/under-escalation warning signs.
- `lessons/mod-101-voice-agent-position/02-position-on-the-ladder.md` — where each escalation target role sits and what they own.
- `lessons/mod-101-voice-agent-position/03-adjacent-voice-ecosystem-roles.md` — relevant to Vignette 3 (ASR research boundary) and Vignette 4 (indirect, biometric-spoofing boundaries sit outside the voice-ecosystem but the handoff shape is the same).

Standards and specialist domains referenced in the vignettes:

- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — relevant to Vignette 6 (dangerous-capability framing).
- OWASP Machine Learning Security Top 10 — `https://owasp.org/www-project-machine-learning-security-top-10/` — relevant to Vignette 4 (audio-adversarial / biometric-spoofing threat modelling).
- MITRE ATLAS — `https://atlas.mitre.org/` — threat-model reference for Vignette 4; the TTP catalogue is useful context for the security-specialist handoff, though not invoked directly in the voice-agent engineer's first-hour artefacts.
- ISO/IEC 30107 (Presentation Attack Detection) — named in Chapter 4 Scenario 4 as the biometric-spoofing standard family. External reference: `https://www.iso.org/standard/79520.html` (overview page).

Note on ISO/IEC 30107 sub-standards. The ISO/IEC 30107 family has multiple parts (30107-1 framework, 30107-2 data formats, 30107-3 testing / reporting). Sub-standard selection depends on whether the deployment is data-format, test-methodology, or evaluation-focused; confirm the applicable part against the current deployment context before citing a specific part number in the Vignette-4 handoff artefact.

Source policy note. Numbers inside the vignettes (780 ms P50, 2.4 s P95, 9.7 % error rate, etc.) come directly from the Chapter-4 scenario prompts (illustrative there, illustrative here). No real customer, incident, or metric is attributed to any real company. The five escalation scenarios and the four-field decision-log format are documented in Chapter 4 of the paired learning repo.
