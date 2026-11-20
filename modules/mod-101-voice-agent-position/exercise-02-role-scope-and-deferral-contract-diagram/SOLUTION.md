# exercise-02 — Role Scope and Deferral Contract Diagram — Reference Solution

## 1. Solution overview

This exercise produces two artefacts:

- **Part A — a scope diagram** (RACI-style matrix) across the eight voice-feature-lifecycle deliverables from Chapter 5 and the roles from Chapter 2 (and, where needed, Chapter 3).
- **Part B — a filled-out deferral contract** for one of the five escalation scenarios in Chapter 4.

Both are *drawing lines in ink* — the learner should resist "we'll figure it out" phrasings in favour of explicit ownership letters and specific, measurable fields.

The worked example below uses:

- RACI letters: `O` owns, `C` contributes, `R` reviews, `I` informed, blank = not involved.
- A deferral contract for **tail-latency escalation to `ai-infra-inference-engineer`** (Chapter 4 Scenario 1), grounded in an illustrative but concrete scenario clearly labelled as such.

## Implementation

The worked answer below is the implementation of this exercise: the two artefacts a reviewer should see (scope diagram with justifications and at-risk cells; filled-out deferral contract with a simulated peer-review pass), followed by validation steps, rubric, and common mistakes. To execute the exercise yourself, follow the shape of §2 against your own chosen escalation scenario rather than the illustrative tail-latency case.

## 2. Worked answer

### 2.1 Part A — scope diagram

Rows are the eight deliverables from Chapter 5. Columns are the seven L30/L35/L40 roles explicitly called out in Chapter 2, plus `agentic-safety-engineer` for the safety / compliance row (Chapter 4 explicitly routes dangerous-capability concerns there).

| Deliverable | voice-agent-engineer (you) | applied-ai-engineer | agentic-ai-engineer | ai-eval-engineer | forward-deployed-engineer | ai-infra-inference-engineer | agentic-safety-engineer | senior-agentic-ai-engineer | ai-engineering-manager |
|---|---|---|---|---|---|---|---|---|---|
| 1. Architecture brief | **O** | C | C | C | C | C | I | R | R |
| 2. Latency budget | **O** | I | I | C | C | C | | R | I |
| 3. Pipeline design | **O** | C | C | | I | C | | R | I |
| 4. Telephony integration doc | **O** | | | | C | I | | I | I |
| 5. Safety / compliance review | C | | | | C | | **O** | R | R |
| 6. Eval bundle | **O** on voice-specific dims | C | C | C on methodology | C | | R | R | I |
| 7. Launch-gate decision | C | | | C | C | | R | R | **O** |
| 8. Operations runbook | **O** | | | | C | C | I | I | R |

Justifications, cell by cell (voice-agent-engineer column only — the exercise explicitly requires *a reason that is not just "because it's on the role charter"*):

- *Architecture brief — O.* You are the only reader of all the components simultaneously; a specialist in any single component writes a narrower doc. Chapter 5 §1 names the voice-agent engineer as the author of record.
- *Latency budget — O.* Chapter 2 §vs. ai-infra-inference-engineer: you own the full-voice-loop budget; the inference engineer owns inside-the-model-server. The budget document must therefore live with the pipeline owner.
- *Pipeline design — O.* Chapter 5 §3 — the author is the only engineer accountable for backpressure and barge-in semantics across ASR, VAD, LLM, TTS, orchestration. These semantics do not have an owner elsewhere.
- *Telephony integration doc — O.* Chapter 2 and Chapter 3 both put the bridge contract on the voice-agent engineer: the telephony specialist owns the carrier side, you own the audio stream once it hits your pipeline. The *doc* is the hinge between them, written by you.
- *Safety / compliance review — C.* Chapter 4 Scenario 5 explicitly escalates dangerous-capability framing to `agentic-safety-engineer`; you *contribute* the per-feature guardrail, refusal logging, and red-team artefacts (Chapter 5 §5) but you don't own the organisational policy.
- *Eval bundle — O on voice-specific dimensions.* Chapter 2 §vs. ai-eval-engineer: they own methodology, you own voice-specific metrics and audio-grounded test sets. The *bundle* is yours; *judge design* inside it is theirs (contribution).
- *Launch-gate decision — C.* Chapter 5 §7: the launch-gate owner is the engineering manager; you contribute the scorecard against pre-committed gates. You do not unilaterally decide to ship.
- *Operations runbook — O.* Chapter 5 §8: the reader is oncall; you are the only engineer who sees all component failure modes. SRE *contributes* (you borrow their dashboarding patterns) but the failure-mode catalogue is yours.

### 2.2 Three annotated at-risk cells (where the boundary is most likely to be mis-drawn)

| Row × column | Common failure mode (one line) |
|---|---|
| Eval bundle × ai-eval-engineer | They review methodology but get drawn into writing voice-specific metrics because you haven't named the dimensions; eval slips while the two of you negotiate scope. |
| Telephony integration doc × forward-deployed-engineer | On a customer with a bespoke telephony stack, the FDE starts writing the bridge doc because they are onsite; the pipeline side drifts out of sync; next customer's handoff doesn't work. |
| Safety / compliance review × voice-agent-engineer | You default to "I'll own compliance, legal is slow" and end up making risk-tier calls that belong to `agentic-safety-engineer` — exactly the failure mode Chapter 4 Scenario 5 is designed to prevent. |

### 2.3 Part B — deferral contract

Scenario chosen: **Tail-latency mystery** (Chapter 4 Scenario 1), handed to `ai-infra-inference-engineer`.

Context (illustrative, concrete, clearly labelled). *Product:* an outbound appointment-reminder voice agent for a (fictional) multi-site medical-group customer. *Measurement window:* 7 days, ~18,000 inbound-answered calls. *Observation:* P50 end-to-end response time (user utterance end → agent first-audible-syllable) is 780 ms, well under the 1.1 s budget; P95 is 2.4 s; P99 is 4.1 s. Per-component traces attribute >85 % of tail overage to LLM TTFT from the current hosted provider on a specific regional endpoint. Numbers are illustrative but shaped to a plausible scenario.

| Field | Content |
|---|---|
| **Problem statement** | On route `outbound-reminder-v4`, LLM-TTFT P95 is 1.9 s (budget 400 ms), making end-to-end voice-loop P95 miss the 1.5 s conversational-feel target. Observed 2025-11-13 to 2025-11-19, hosted provider region `us-east-1`. |
| **What I've tried** | (1) Confirmed via distributed tracing that STT finalisation, VAD endpointing, TTS TTFB, and transport jitter are each inside their individual budgets at P95. (2) Verified the issue is request-side, not prompt-side: the same prompt at 02:00 local shows P95 TTFT of 480 ms; the issue clusters at 09:00–11:00 local. (3) Ruled out token-count variance: TTFT is pre-first-token, so prompt-completion length is irrelevant. (4) Attempted prefix-cache–friendly prompt rewrite; no change at P95. |
| **Minimal reproducer** | A 200-call recorded trace bundle at `s3://voice-evals/trace-bundles/outbound-reminder-v4-tail/`, with request IDs, provider request IDs where returned, hop-by-hop timestamps in a shared clock, and the `curl`-reproducible form of the exact request being sent. The 20 slowest calls include provider request IDs the vendor can use for server-side correlation. |
| **Budget / threshold** | "Fixed" = LLM-TTFT P95 ≤ 400 ms and P99 ≤ 800 ms during 09:00–11:00 local on the same workload. If the fix is a provider-tier upgrade, cost-per-call delta must be < $0.01 for this product to stay economic. |
| **What stays mine** | (a) Re-running the end-to-end voice-loop eval bundle after any serving-side change. (b) Re-validating barge-in semantics if the provider is swapped. (c) Owning the user-facing rollback switch and the per-call retry policy. (d) Writing the postmortem once resolved. |
| **What's theirs** | (a) Root-cause the tail: is it noisy-neighbour batching, autoscale lag, routing to a cold region, or provider-side degradation? (b) Choose the lever: tier change, provider negotiation, self-hosted fallback, prefix-cache topology change, speculative decoding. (c) Own the vendor-escalation conversation and the SLA re-negotiation if that's the lever. |

### 2.4 Part C — peer-review loop, simulated

What a competent peer-reviewer would push back on if handed this submission:

- *Cell to move.* The peer might argue **Telephony integration doc × forward-deployed-engineer** should be `C` not `I` on the general case, because in most CPaaS integrations the FDE knows the customer's PBX better than you do. Accept the move for scenarios where the customer has a bespoke telephony stack; keep as `I` for commodity CPaaS integrations where the only FDE input is "the customer uses Twilio."
- *Field to tighten.* The peer might flag the **"Budget / threshold"** line "cost-per-call delta must be < $0.01" as vague — delta against *what baseline*, over *what measurement window*, including or excluding the eval traffic? Tighten to: "cost-per-call delta ≤ $0.01 on steady-state production traffic, measured on the first full billing week after the change, excluding eval and canary traffic."

## 3. Validation steps

A finished submission should pass each of the following checks.

1. **All eight rows present.** Architecture brief, latency budget, pipeline design, telephony integration doc, safety / compliance review, eval bundle, launch-gate decision, operations runbook. If any row is missing, the deliverable isn't done.
2. **At least six columns.** Chapter 2 lists seven roles; six is the acceptance-criterion floor.
3. **Every voice-agent-engineer cell has a reason.** Not "role charter." A reason from Chapter 2 §vs. a specific peer, or from Chapter 5's reader list.
4. **Three at-risk cells annotated.** Each with a one-line failure mode; failure modes name the artefact that drifts, not a personality trait.
5. **Deferral contract has all six fields populated.** Every field passes the "could a peer start work from this today?" test. "What I've tried" lists concrete attempts, not "I thought about it."
6. **Numbers are labelled.** Any metric that is illustrative must say so. No numbers presented as real measurements unless they are.
7. **Peer-review note exists.** One cell to move with the rationale; one field to tighten with the tightened phrasing.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Diagram completeness | 15 % | 8 rows × ≥6 columns filled | All eight columns from Chapter 2 plus `agentic-safety-engineer` |
| Ownership precision | 20 % | Every voice-agent cell has a reason rooted in Chapter 2 or 5 | Reasons cite the specific chapter paragraph |
| At-risk cells | 15 % | 3 cells flagged with named failure modes | Failure modes reference a Chapter-3 or Chapter-4 handoff artefact |
| Deferral-contract completeness | 20 % | All six fields non-empty, each actionable | Reproducer is a real pointer (S3 path, repo link), not a description of one |
| Scenario specificity | 15 % | Product, metric, numbers (even if illustrative) called out | Numbers are labelled as illustrative; still internally consistent across the contract |
| Peer-review loop | 15 % | One cell moved + one field tightened | Both come with the rewritten version, not just the critique |

A submission scoring < 70 % is a fail-and-redo. The most common sub-70 failure mode: a diagram with letters but no reasons.

## 5. Common mistakes

- **Over-owning safety / compliance.** The default voice-agent-engineer instinct is to write "O" on the safety / compliance review row. Chapter 4 explicitly routes dangerous-capability framing to `agentic-safety-engineer`; the voice engineer contributes, does not own.
- **Writing "all cells = contributor."** A RACI with no clear owners is a RACI that will fail at deadline. Force at least one `O` per row.
- **Writing the deferral contract in prose.** The whole point of Chapter 4's table is the structure. Six fields, each with a measurable answer.
- **"What's mine / what's theirs" that is actually a wish-list.** "Theirs: fix the latency." No. "Theirs: identify the root cause of LLM-TTFT P95 degradation at 09:00–11:00 local, choose the serving-side lever, negotiate the provider-tier change if that's the lever." Specific levers, not aspirations.
- **No reproducer.** "Here are our logs, feel free to grep" is not a reproducer. A reproducer is a named file, an S3 path, a trace bundle, a `curl` command, a jupyter notebook — something that produces the same measurement on another engineer's laptop.
- **Treating "informed" and "not involved" as the same thing.** They are not. If the inference-engineer should see the architecture brief, mark `I`, do not leave blank. The RACI's job is to make the distribution list implicit rather than implicit.
- **Inventing peer-review feedback.** If you cannot find a peer, simulate it adversarially: name the two most specific objections someone who hates your diagram would raise. Be your own reviewer; don't write "Reviewer said: looks good."

## 6. References

Primary — chapters this exercise unpacks:

- `lessons/mod-101-voice-agent-position/02-position-on-the-ladder.md` — role column list and per-boundary rules.
- `lessons/mod-101-voice-agent-position/03-adjacent-voice-ecosystem-roles.md` — IVR / contact-centre-ops boundary used for the telephony-doc row.
- `lessons/mod-101-voice-agent-position/04-generalist-stance-and-escalation.md` — deferral-contract structure (the six-field table).
- `lessons/mod-101-voice-agent-position/05-deliverables-across-the-voice-feature-life-cycle.md` — the eight-deliverable row set.

Standards referenced indirectly (for the compliance / dangerous-capability rows; included for completeness, not invoked in this specific contract):

- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — the broader frame under which dangerous-capability concerns are escalated.
- OWASP Machine Learning Security Top 10 — `https://owasp.org/www-project-machine-learning-security-top-10/` — relevant when the safety-review row intersects model-security concerns.
- MITRE ATLAS — `https://atlas.mitre.org/` — threat-model reference for audio-adversarial and prompt-injection cases (Chapter 4 Scenario 4).

Note on the `agentic-safety-engineer` column. This solution adds `agentic-safety-engineer` to the Chapter-2 column set because Chapter 4 names it as the escalation target for Scenario 5; the curriculum does not strictly require that column in the diagram, but including it makes the safety / compliance row's ownership legible rather than implicit.

Source policy note. The RACI letters and six-field deferral contract are generic operational conventions, not external standards. The specific numbers in Part B are labelled as illustrative; no claim is made that they come from a real incident.
