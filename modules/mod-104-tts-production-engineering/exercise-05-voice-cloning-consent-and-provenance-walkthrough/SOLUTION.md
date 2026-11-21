# exercise-05 — Voice-Cloning Consent and Provenance Walkthrough — Reference Solution

## 1. Solution overview

This is a governance walkthrough exercise. Voice cloning is the one TTS capability where "ship first, document later" is not an option: consent-of-record, provenance, and revocation are legal, ethical, and operational requirements before a cloned voice goes anywhere near production audio. The learning target is **a walkthrough that a voice-agent engineer can hand to their legal + security + product partners and get sign-off on** — not a "trust us" one-pager.

This reference ships, end-to-end, the four artefacts a defensible voice-cloning programme owns:

1. A **consent agreement template** that captures what the consenting speaker is actually consenting to, in plain-language terms.
2. A **provenance record schema** (C2PA-aligned) that attaches, to every clone and every generated utterance, a cryptographically auditable trail back to that consent.
3. A **threat-model walkthrough** mapped to OWASP ML Top 10 and MITRE ATLAS, with the control for each threat called out explicitly.
4. A **revocation + incident runbook** — what happens the day the speaker withdraws consent, or the day a cloned voice is reported mis-used.

Nothing below is legal advice. Everything below is the engineering substrate that a legal team can build a compliant programme on top of; the authoritative references are in §6.

## 2. Implementation

### 2.1 Target properties

A reviewer should see every one of these:

1. **Specific, not aspirational, consent.** Consent names the permitted uses, the duration, the compensation (if any), and the revocation path. Vague "model training" wording fails.
2. **Provenance is cryptographic, not a text log.** Every generated clip carries a signed manifest tying it to a specific consented voice ID and a specific call/session.
3. **Threat model is enumerated.** Impersonation, prompt injection (voice), supply-chain, and re-identification are each treated as line items with a named control.
4. **Revocation is tested.** There is a drill that proves revoking a voice ID actually prevents further synthesis within a stated SLA.
5. **The surfaces that touch regulated data are listed.** Who stores the consent form, who stores the voice recording, who holds the cloned-model artefact, who holds the private signing key for provenance.
6. **The policy stack is cited.** NIST AI RMF, OWASP ML Top 10, MITRE ATLAS, C2PA, and the jurisdiction-specific baseline (BIPA, CCPA/CPRA, EU AI Act) are each referenced by their official project page where applicable.

### 2.2 Consent agreement template (plain-language skeleton)

Author with your legal team; this is the engineering-facing skeleton, not final contract wording. `<<ANGLE-BRACKETED>>` fields are per-engagement.

```
Voice Cloning Consent Agreement v<<version>>

1. Who you are.
   Full legal name: <<name>>.
   Email / phone of record: <<contact>>.
   Jurisdiction of residence: <<jurisdiction>>.
   Evidence of identity verified by: <<method — government ID, employer attestation, etc.>>.

2. What this agreement covers.
   You are granting <<company>> a licence to use a recording of your voice to
   train, host, and generate speech from a model (the "Cloned Voice") for the
   purposes listed in Section 3.  No other use is authorised by this document.

3. Permitted uses (enumerate; everything not listed is NOT permitted).
   [ ] Internal demos and employee-facing testing.
   [ ] A specific named product: <<product + route>>.
   [ ] Specific customer-facing scripts in categories: <<e.g. onboarding, reminders>>.
   [ ] Multilingual output in: <<languages>>.
   NOT permitted unless separately agreed in writing:
   - Impersonation of any other real person.
   - Political, religious, or sexual content.
   - Training a general-purpose voice model other than the Cloned Voice itself.
   - Sale or sublicensing of the Cloned Voice to any third party.

4. Duration and termination.
   This licence starts <<start>> and ends <<end — fixed term, not perpetual>>.
   You may revoke this licence at any time by emailing <<revocation address>>.
   <<company>> will cease synthesis within <<revocation SLA, e.g. 48 hours>>
   of receipt and will delete the Cloned Voice model within <<deletion SLA,
   e.g. 30 days>>, confirming by signed notice.

5. Compensation.
   <<flat fee / royalty / employment-scope statement>>.

6. Provenance and watermarking.
   Every clip generated from the Cloned Voice will carry a signed C2PA
   manifest (see §2.4) attesting its synthetic origin.
   We will apply an inaudible audio watermark where technically feasible.

7. Your rights.
   You may request a list of all generated clips tied to your voice at any
   time.  You may request deletion of any identifying recording used to
   train the model.  You may raise an incident at <<incident address>>.

8. Governing law and liability.
   <<Legal to draft.>>

Signed: _______________________________   Date: _________
Witness / notary (where required): __________________________
```

### 2.3 Provenance record schema (C2PA-aligned)

Every cloned voice gets a **VoiceLicense** record; every generated clip gets a **GenerationClaim** record. Both are signed with the production signing key; both are retrievable by ID through a provenance-service API. Field names map to the C2PA assertion vocabulary where possible (see §6 for the C2PA spec).

```json
{
  "VoiceLicense": {
    "voice_id": "vc-2026-11-21-eb0c",
    "consent_doc_hash": "sha256:7d9e...",
    "consent_doc_version": "v3",
    "consent_doc_signed_at": "2026-11-21T14:02:00Z",
    "speaker_identity_claim": {
      "verifier": "<<kyc-provider>>",
      "verified_at": "2026-11-20T11:00:00Z"
    },
    "permitted_uses": [
      "product:order-reminders",
      "product:onboarding",
      "lang:en-US",
      "lang:es-MX"
    ],
    "forbidden_uses": [
      "political",
      "impersonation-of-third-party",
      "sublicense"
    ],
    "license_start": "2026-11-21",
    "license_end": "2027-11-21",
    "revocation_contact": "voice-ops@example.com",
    "revocation_sla_hours": 48,
    "deletion_sla_days": 30,
    "signing_cert_fingerprint": "sha256:ea1a..."
  }
}
```

```json
{
  "GenerationClaim": {
    "clip_id": "clip-2026-11-21T14:22:07Z-...",
    "voice_id": "vc-2026-11-21-eb0c",
    "voice_license_hash": "sha256:7d9e...",
    "generated_at": "2026-11-21T14:22:07.142Z",
    "model": "xtts-v2-2026-10",
    "model_hash": "sha256:bb5e...",
    "text_hash": "sha256:1fbe...",
    "watermark": {"scheme": "<<watermark-scheme>>", "present": true},
    "session": {
      "product": "order-reminders",
      "route": "outbound-en-us",
      "call_id": "call-...",
      "turn_id": "turn-17"
    },
    "ssml_hash": "sha256:acc3...",
    "c2pa_manifest_url": "https://provenance.example.com/clip-.../manifest.c2pa",
    "signing_cert_fingerprint": "sha256:ea1a..."
  }
}
```

**Design decisions and rationale.**

| Field / choice | Why |
|---|---|
| `consent_doc_hash` on the License | A text contract can be altered; the hash is the anchor of what was actually agreed. |
| `voice_license_hash` on every Claim | Each clip can be audited back to a specific consent version without round-tripping to a DB. |
| `permitted_uses` / `forbidden_uses` as a list | Pre-generation policy check is a list membership. "Trust the operator to read the contract" is not a control. |
| Separate `license_end` + `deletion_sla_days` | Licence end is the business terminus; deletion is the operational terminus. They're not the same date. |
| `watermark.present` as boolean, scheme named | Some scenarios can't watermark (telephony 8 kHz µ-law may degrade the carrier); the record tells you *that* — honest failure mode. |
| `text_hash` + `ssml_hash` | Allows after-the-fact audit of exactly what was spoken, without storing PII plaintext. |
| `model_hash` | Ties the clip to the model binary; if a specific model version is later revoked, you can find every clip generated from it. |
| C2PA manifest URL | The provenance system is the authority; the JSON here is a convenient summary. |

### 2.4 Threat model (OWASP ML Top 10 + MITRE ATLAS mapping)

| # | Threat | Control | Policy anchor |
|---|---|---|---|
| T1 | **Voice impersonation of a real person** — adversary provides a target's voice to the cloner | Identity verification of the consenting speaker before clone creation; policy-level ban on third-party-identity clones; consent agreement §3 forbids it | NIST AI RMF Govern/Map; MITRE ATLAS — Reconnaissance → Abuse ML to generate deceptive content |
| T2 | **Prompt / content injection into the voice** — an upstream LLM is manipulated into emitting phishing lines in a trusted voice | Provenance manifest requires `text_hash`; red-team the LLM prompt surface; policy enforcement on generated content before synthesis | OWASP ML Top 10 — ML01 Input Manipulation; MITRE ATLAS — ML Attack Staging |
| T3 | **Model-weights exfiltration** — the cloned model is copied and used without the consent chain | Model artefact stored encrypted-at-rest; access-logged; signing cert on every Claim; rotation plan for compromised signing keys | OWASP ML Top 10 — ML05 Model Theft; NIST AI RMF Manage 4 |
| T4 | **Supply-chain compromise of the TTS pipeline** — an SBOM-in-the-chain delivers a compromised model or runtime | Pinned model hashes in the GenerationClaim; SBOM for the TTS runtime; reproducible-build target where feasible | OWASP ML Top 10 — ML06 AI Supply Chain; NIST AI RMF Manage 1 |
| T5 | **Re-identification of training recording** — the raw voice sample is linked to the speaker's identity in a breach | Encrypted-at-rest; retention set to the shortest period that supports retraining; segregated storage from general product data | OWASP ML Top 10 — ML08 Model Skewing / data-exposure adjacent; NIST AI RMF Map 2 |
| T6 | **Voice cloning for fraud (deepfake of the brand itself)** — adversary clones *your* voice product and uses it against your customers | Watermark where feasible; publish the clips-from-us provenance endpoint for customers to verify; monitoring on social media / abuse channels | MITRE ATLAS — Credential Access via deepfake voice; NIST AI RMF Govern 5 |
| T7 | **Consent-withdrawal not honoured** — speaker revokes, synthesis continues | Revocation drill (§2.5); synthesis-gate reads live license store on every call | NIST AI RMF Govern 1.4; EU AI Act Article 50 transparency obligations (applicable to deepfake-labelled content) |
| T8 | **Audio-adversarial transcript** — carefully-crafted input makes the voice say something unintended (prosody-smuggling) | Content policy pre-SSML; SSML lexicon pinned (exercise-03); per-utterance human-reviewable `text_hash` | MITRE ATLAS — ML Attack Staging; OWASP ML Top 10 — ML01 |
| T9 | **PII leakage through synthesis** — the voice reads out confidential data | DLP on the input text; redaction of PII before TTS; log minimisation (store `text_hash`, not plaintext) | NIST AI RMF Map 5 (privacy) |
| T10 | **Model update changes the voice without consent** — retraining drifts the voice away from what the speaker consented to | License covers only approved models (`model_hash` list); speaker is notified of material model updates | NIST AI RMF Govern 1.2 |

### 2.5 Revocation and incident runbook

**Standard revocation (speaker initiates).**

1. Revocation request arrives at `voice-ops@example.com` (or product-specific channel).
2. Request verified against the consent doc's `revocation_contact` line (same person; verified via a second channel).
3. The voice ID is flagged `revoked=true` in the license store. *From this moment, the synthesis gate refuses any GenerationClaim against the voice ID.* Target: within 1 hour of verified request.
4. Any in-flight calls referencing the voice complete their current turn, then fail over to a fallback voice. Target: within the stated `revocation_sla_hours` (e.g. 48 h) all live traffic has cut over.
5. Within `deletion_sla_days` (e.g. 30 days), model artefact and training recordings are deleted from primary and backup storage. A signed deletion notice goes to the speaker.
6. All GenerationClaims against the voice ID are retained for audit — this is *not* deleted, because a future dispute needs the record. The license's `permitted_uses` prevents new generation; the Claims are evidence of what happened before.

**Incident — reported misuse of a cloned voice.**

1. Pause the voice ID (`paused=true`) immediately; synthesis stops within minutes.
2. Open incident channel; preserve logs, Claims, and manifest signing-key access records.
3. Verify whether clips were generated inside our stack (provenance manifest resolves) or outside (adversary cloned our voice independently — different response).
4. If inside: notify the speaker; notify affected customers; notify regulator where required; begin forensics on the generation path (prompt, tool-call, operator).
5. If outside: publish the clips-from-us provenance endpoint; coordinate with platform(s) hosting the content; file abuse reports; consider legal action.
6. Post-incident: update threat model (§2.4); if a control failed, update the control, not just the specific log.

**Drill (quarterly minimum).**

Run through standard revocation end-to-end on a non-production voice ID. Measure:

- Time from request receipt to `revoked=true` in license store.
- Time to live-traffic failover.
- Time to model deletion confirmation.

Publish the drill result internally; a drill with no written result is a drill that didn't happen.

### 2.6 Surfaces and ownership

| Surface | Stored where | Who owns | Minimum controls |
|---|---|---|---|
| Consent agreement (signed PDF/HTML) | Legal document store | Legal | Immutable version; cryptographic hash in `VoiceLicense.consent_doc_hash` |
| Training recording | Encrypted blob store | Voice-ops team | Retention = shortest period needed for retraining; separate access control from product data |
| Cloned model artefact | Encrypted model store | ML-platform team | Pinned hash; access-logged; deletion on revocation |
| License store (`VoiceLicense` records) | Transactional DB | Voice-ops team | Backed up; audit log for every write; read-replicated to the synthesis gate |
| GenerationClaim log | Append-only log | Voice-ops team | Retained for stated audit period; hash-chained if the compliance tier requires |
| Provenance signing key | HSM / KMS | Security team | Access gated by break-glass; rotation plan; alarm on signing-rate spike |
| Public provenance verification endpoint | Public web | Product team | Allows customers / regulators to verify "did this clip come from you?" by manifest URL |
| Watermark detection tool | Internal + partner-shared | Security team | Documented FAR / FRR on the carriers you actually use |

## 3. Validation steps

1. **Consent coverage.** For every production voice ID, there is a signed, versioned consent doc; `consent_doc_hash` on the License matches the stored doc; `permitted_uses` and `forbidden_uses` are both populated; `license_end`, `revocation_sla_hours`, and `deletion_sla_days` are explicit numbers.
2. **Pre-generation gate.** The synthesis service refuses a GenerationClaim if (a) the voice ID is revoked, (b) the requested use is not in `permitted_uses`, or (c) the SSML / text hash is missing. Negative test cases exist for each.
3. **Provenance round-trip.** Pick a random recent clip; resolve its manifest URL; verify the signature; trace it back to a VoiceLicense; trace that to the stored consent doc hash.
4. **Threat-model coverage.** Every row in §2.4 has a named owning team and a named control in the codebase or runbook; none are left as "TODO" or "tracked in Jira."
5. **Revocation drill.** Within the last 90 days, a drill has been run; written artefact includes the three measured timings and a comparison to the stated SLAs.
6. **Deletion drill.** Within the last 180 days, a model-deletion drill has been run end-to-end on a non-production voice ID; confirmation signed off by the voice-ops lead.
7. **Policy mapping.** The threat-model table cites OWASP ML Top 10, MITRE ATLAS, and NIST AI RMF entries by their official URLs (not paraphrases).
8. **No PII plaintext in logs.** Spot-check a random day of GenerationClaim log entries; `text_hash` and `ssml_hash` are present, plaintext is not.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| Consent specificity | 15 % | Permitted uses enumerated; forbidden uses enumerated; duration and revocation SLA are explicit numbers | Jurisdiction-specific baselines (BIPA / CCPA / EU AI Act) cited in the agreement where applicable |
| Provenance schema | 15 % | VoiceLicense + GenerationClaim fields populated; hashes link License → Consent and Claim → License; C2PA-alignment stated | Clip-level manifest URL resolves publicly and signature verifies on a vanilla C2PA toolchain |
| Threat model | 15 % | ≥ 8 threats enumerated; each with a named control and a policy-anchor citation | Each threat has a named *owning team* and a listed test case |
| Revocation runbook | 10 % | Steps, named SLAs, and the drill cadence are stated | Last drill result is in the repo with measured timings |
| Deletion / retention | 10 % | Deletion SLA stated; retention period stated and justified | Deletion confirmation sent to speaker per the signed template |
| Surface / ownership matrix | 10 % | Each surface has a named owning team and a minimum control | Signing-key rotation plan documented |
| Policy citation discipline | 10 % | OWASP ML Top 10, MITRE ATLAS, NIST AI RMF, C2PA cited by official URLs | Jurisdiction-specific citations also by official URL |
| Pre-generation gate | 10 % | Synthesis service refuses on revoked / out-of-scope / missing-hash; negative tests exist | Audit log every refusal with reason code |
| Audit traceability | 5 % | Random clip → manifest → License → consent hash round-trip works | Round-trip is a scripted test the grader can run |

Scoring < 70 % is a fail-and-redo. The most common sub-70 failures are: (a) a one-paragraph "we will seek consent" statement with no permitted-use enumeration, (b) a provenance log that's just a text file (unsigned), (c) a revocation plan that's never been drilled.

## 5. Common mistakes

- **Vague consent.** "You consent to use of your voice for business purposes" fails on first review. Enumerate uses; enumerate forbidden uses.
- **Perpetual licence.** A perpetual, non-revocable voice licence is probably unenforceable in most jurisdictions and is a bad relationship. Set an end date.
- **No revocation SLA.** "We will stop within a reasonable time" is not an SLA. Pick a number; drill against it.
- **Confusing the consent doc with the License record.** The doc is the agreement; the License record is the operational lookup. The hash binds them; they're not the same thing.
- **Provenance in a text log.** `audit.log` is not provenance. Sign the records; use C2PA where you can.
- **Watermark claimed but not tested.** A watermark with no measured FAR/FRR on your actual carriers (telephony 8 kHz µ-law especially) is a marketing claim. Measure it or don't claim it.
- **Threat model without owners.** A table with 10 threats and no owning teams is a document, not a programme.
- **Policy citations paraphrased.** Reviewers (legal, security, regulators) want the official URL, not your summary.
- **One signing key.** A single signing key with no rotation is an incident waiting to happen. Rotate on a schedule and on break-glass.
- **Drill avoidance.** The most informative part of this programme is the drill. Skipping drills to avoid finding problems does not avoid the problems.
- **Treating the pre-generation gate as advisory.** If `permitted_uses` can be bypassed by setting a flag, you don't have a gate; you have a comment. Enforce in code, log every refusal.
- **Deletion without confirmation.** Deleting the model without sending the speaker a signed deletion notice creates a dispute that could have been closed.

## 6. References

Standards and official policy projects (these are the authoritative anchors cited by the threat-model table):

- **NIST AI Risk Management Framework (AI RMF 1.0)** — `https://www.nist.gov/itl/ai-risk-management-framework` — the "Govern / Map / Measure / Manage" frame used throughout §2.4 and §2.6. *Official NIST standard.*
- **OWASP Machine Learning Security Top 10** — `https://owasp.org/www-project-machine-learning-security-top-10/` — model-level threat vocabulary for ML01 (Input Manipulation), ML05 (Model Theft), ML06 (AI Supply Chain), ML08 (Model Skewing / data-exposure adjacent). *Official OWASP project.*
- **MITRE ATLAS** — `https://atlas.mitre.org/` — adversarial-ML tactics/techniques matrix; the "Reconnaissance", "ML Attack Staging", and "Credential Access" anchors cited in §2.4 come from ATLAS. *Official MITRE project.*
- **C2PA (Coalition for Content Provenance and Authenticity) Technical Specification** — `https://c2pa.org/specifications/specifications/2.1/specs/C2PA_Specification.html` (stable landing at `https://c2pa.org/specifications/`) — the manifest / assertion format the provenance schema in §2.3 aligns to.

Jurisdiction-specific references the consent template should be tuned against (consult counsel):

- EU AI Act (Regulation (EU) 2024/1689) — overview at `https://artificialintelligenceact.eu/` and the official text at `https://eur-lex.europa.eu/eli/reg/2024/1689/oj`. Article 50 is the deepfake-transparency anchor cited in §2.4 T7. <!-- verify-at-runtime: confirm Article 50 is the final numbering in the adopted text at review time -->
- Illinois Biometric Information Privacy Act (BIPA) — official text via `https://www.ilga.gov/legislation/ilcs/ilcs3.asp?ActID=3004` — informs consent form fields for Illinois-resident speakers.
- California Consumer Privacy Act (CCPA) / CPRA — official regulations at `https://cppa.ca.gov/regulations/` — informs deletion / data-subject-rights fields.

Standards / technical references used in the provenance design:

- RFC 8785 JSON Canonicalization Scheme — `https://www.rfc-editor.org/rfc/rfc8785` — for stable hashing of JSON records before signing.
- NIST SP 800-57 Key Management — `https://csrc.nist.gov/publications/detail/sp/800-57-part-1/rev-5/final` — informs the signing-key rotation policy.

Related curriculum anchors:

- Safety / compliance layer that owns the policy stack — `mod-108-safety-compliance-voice`
- Eval / observability layer that runs the drills — `mod-112-voice-multimodal-eval-and-observability`

Source policy note. OWASP ML Top 10, MITRE ATLAS, NIST AI RMF, and C2PA are each cited by their official project / standard URL above. Jurisdiction-specific statutes are cited to their official government source. Nothing in this reference is legal advice; the engineering substrate is designed to make compliant deployment achievable with appropriate legal review. Where a factual claim about the EU AI Act numbering could not be confirmed at write time, a `<!-- verify-at-runtime: ... -->` marker is present rather than an invented citation.
