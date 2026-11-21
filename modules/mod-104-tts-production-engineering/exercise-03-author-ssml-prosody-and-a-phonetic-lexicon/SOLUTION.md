# exercise-03 — Author SSML Prosody and a Phonetic Lexicon — Reference Solution

## 1. Solution overview

This is an authoring exercise. The learning target is **writing SSML and a phonetic lexicon well enough that a reviewer can tell you understand *why* each tag and lexeme is there**, not merely that you can produce output that renders. The deliverable has three parts:

1. A worked SSML snippet exercising the tags a production voice agent actually uses day-to-day: `<speak>`, `<break>`, `<prosody>`, `<say-as>`, `<sub>`, `<phoneme>`, `<lang>`, `<mark>`, and, where supported, `<emphasis>` / voice-provider style tags.
2. A PLS (Pronunciation Lexicon Specification) lexicon covering the five entity classes most likely to break on your corpus: brand/product names, people names, place names, abbreviations/acronyms, and numbers-that-aren't-numbers (SKUs, order IDs, model codes).
3. Rationale, validation, rubric, and common-mistakes list.

SSML is a W3C standard (SSML 1.1); PLS is a W3C standard. Provider conformance is partial — every provider documents their own subset. The authored artefacts below are conformant to the W3C core; a short cross-walk table in §2.6 shows which provider-specific extensions are safe where.

## 2. Implementation

### 2.1 Target properties

A reviewer should see the authored SSML prove each of these:

1. **Tags are used for a reason.** Every `<break>`, `<prosody>`, `<say-as>`, `<sub>`, `<phoneme>` has a one-line rationale — what would sound wrong without it.
2. **Prosody is restrained.** Pitch, rate, volume deltas are small (±10 %, ±2 st, ±2 dB) and only where the neutral delivery is actually wrong. Over-tuned prosody sounds worse than neutral.
3. **Phonemes use a stated alphabet.** `ipa` or `x-sampa` is set explicitly on `<phoneme alphabet="ipa">`. A snippet using phonemes without naming the alphabet is a bug.
4. **Lexicon is loaded, not inlined.** `<lexicon uri="..."/>` or the provider's equivalent — not 300 `<phoneme>` tags inline. The lexicon is a reusable, versioned artefact.
5. **Numbers that look like numbers aren't always.** `"ORD-1024"` must be spelled; `"$49.99"` must be rendered as currency; `"2026-11-21"` must be rendered as a date. One wrong `<say-as>` is one of the loudest failures a listener catches.
6. **Language switches are declared.** Mid-sentence foreign terms wrapped in `<lang xml:lang="...">` prevent the English voice from mangling them.
7. **Marks allow synchronisation.** `<mark name="..."/>` is included where the caller wants to time an on-screen highlight or a transport cue. (See exercise-02 for cancellation; `<mark>` complements that.)

### 2.2 Worked SSML — the "order status" reply

Scenario: an outbound reminder/service voice agent reads a customer's order status, with a phonetically tricky brand name ("Weizenmann"), a Spanish product line ("Pan Dulce Clásico"), an order ID ("ORD-10428-B"), a price ("$149.99"), a tracking URL to spell letter-by-letter ("wzm.co/t/XB7"), and a short branded pause after the sign-off.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<speak version="1.1"
       xmlns="http://www.w3.org/2001/10/synthesis"
       xml:lang="en-US">
  <lexicon uri="https://assets.example.com/tts/lex/brand-en-us.v3.pls"
           type="application/pls+xml"/>

  <!-- 1. Greeting; moderate warmth without overdoing it. -->
  <prosody rate="95%" pitch="+1st">
    Hi, this is <sub alias="Weitzen-mon">Weizenmann</sub> Supply.
  </prosody>
  <break time="250ms"/>

  <!-- 2. The core fact.  Slow slightly; this is the part the caller will repeat back. -->
  <prosody rate="92%">
    Your order
    <mark name="order_id_begin"/>
    <say-as interpret-as="characters">ORD</say-as>-<say-as interpret-as="digits">10428</say-as>-<say-as interpret-as="characters">B</say-as>
    <mark name="order_id_end"/>
    for
    <lang xml:lang="es-MX">Pan Dulce Clásico</lang>
    has shipped.
  </prosody>
  <break time="300ms"/>

  <!-- 3. Price; money, not a float. -->
  Total charged:
  <say-as interpret-as="currency" language="en-US">$149.99</say-as>.
  <break time="200ms"/>

  <!-- 4. Tracking URL; letter-by-letter, lower-case explicit. -->
  You can track at
  <say-as interpret-as="characters">wzm</say-as>
  dot co, slash, t, slash,
  <say-as interpret-as="characters">XB7</say-as>.
  <break time="250ms"/>

  <!-- 5. Branded sign-off; slight pitch lift, then a mandatory pause before hangup. -->
  <prosody pitch="+1st">Thanks for choosing Weizenmann.</prosody>
  <break time="400ms"/>
</speak>
```

### 2.3 Why each tag is there

| Line | Tag | What breaks without it |
|---|---|---|
| Greeting prosody | `<prosody rate="95%" pitch="+1st">` | The default voice reads "Hi, this is …" too fast and too flat for a customer-service opening. Deltas are small (±1st, ±5 %) — any larger and it sounds performative. |
| Brand name | `<sub alias="Weitzen-mon">Weizenmann</sub>` | Neutral English voice pronounces "Weizenmann" as "wee-zen-man" which is jarring. `<sub>` swaps *what the voice says* without changing on-screen text. Alternative: a `<phoneme>` tag — see §2.5 for why this reference uses the lexicon file instead. |
| Order ID | `<say-as interpret-as="characters">ORD</say-as>-<say-as interpret-as="digits">10428</say-as>-<say-as interpret-as="characters">B</say-as>` | Default reading is "ord ten-thousand four hundred twenty eight dash b" — unusable for a customer trying to write it down. Splitting by `<say-as>` is deliberate: letters as letters, digits as digits, trailing letter as a letter. |
| Mid-sentence Spanish | `<lang xml:lang="es-MX">Pan Dulce Clásico</lang>` | The English voice would read "Clásico" as /ˈklasɪkoʊ/ with a weird diphthong. `<lang>` either switches to a Spanish voice (provider-dependent) or applies Spanish pronunciation rules. Also the honest move when branding is bilingual. |
| Price | `<say-as interpret-as="currency" language="en-US">$149.99</say-as>` | Without this the voice reads "dollar one four nine dot nine nine" or "one hundred forty nine point nine nine dollars" depending on the model. `interpret-as="currency"` with `language` gives "one hundred forty-nine dollars and ninety-nine cents." |
| Marks | `<mark name="order_id_begin"/>` / `_end` | The caller can subscribe to mark events to highlight the order ID on screen, or to pre-load a "would you like me to repeat that?" fallback branch. Marks have zero audio cost. |
| Breaks | `<break time="250ms"/>`, `300ms`, `400ms` | Natural pause length — short enough to not feel robotic, long enough for the listener to catch up. The 400 ms before hangup is a brand convention from the exercise prompt; the others are standard. |
| Tracking URL | letter-by-letter `<say-as interpret-as="characters">` | Users will transcribe this; readable-aloud format matters more than natural flow. The `dot co, slash` is spelled in-text deliberately so a human proof-reader can audit it. |

### 2.4 Worked PLS lexicon — `brand-en-us.v3.pls`

This file lives in versioned storage and is loaded via `<lexicon uri="..."/>` above. The alphabet is IPA (W3C PLS §4.2); `x-sampa` is an acceptable alternative where a provider cannot consume IPA.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<lexicon version="1.0"
         xmlns="http://www.w3.org/2005/01/pronunciation-lexicon"
         alphabet="ipa"
         xml:lang="en-US">

  <!-- Brand names ================================================ -->
  <lexeme>
    <grapheme>Weizenmann</grapheme>
    <phoneme>ˈwaɪt.sən.mɑn</phoneme>
  </lexeme>
  <lexeme>
    <grapheme>Weizenmann's</grapheme>
    <phoneme>ˈwaɪt.sən.mɑnz</phoneme>
  </lexeme>

  <!-- Product lines (code-switched) =============================== -->
  <lexeme xml:lang="es-MX">
    <grapheme>Pan Dulce Clásico</grapheme>
    <phoneme>pan ˈdul.se ˈkla.si.ko</phoneme>
  </lexeme>

  <!-- People names ================================================ -->
  <lexeme>
    <grapheme>Siobhan</grapheme>
    <phoneme>ʃɪˈvɔːn</phoneme>
  </lexeme>
  <lexeme>
    <grapheme>Dvořák</grapheme>
    <phoneme>ˈdvɔr.ʒɑːk</phoneme>
  </lexeme>

  <!-- Places ====================================================== -->
  <lexeme>
    <grapheme>Worcester</grapheme>
    <phoneme>ˈwʊs.tər</phoneme>
  </lexeme>
  <lexeme>
    <grapheme>Boise</grapheme>
    <phoneme>ˈbɔɪ.si</phoneme>
  </lexeme>

  <!-- Abbreviations / acronyms ==================================== -->
  <!-- SKU: spelled; alias form so no phoneme list required. -->
  <lexeme>
    <grapheme>SKU</grapheme>
    <alias>ess kay you</alias>
  </lexeme>
  <lexeme>
    <grapheme>HVAC</grapheme>
    <alias>H V A C</alias>
  </lexeme>
  <lexeme>
    <grapheme>HIPAA</grapheme>
    <phoneme>ˈhɪp.ə</phoneme>  <!-- said as a word, not spelled -->
  </lexeme>

  <!-- Numbers-that-aren't-numbers ================================= -->
  <!-- "ORD-xxxxx-B" pattern handled at the SSML layer with say-as.
       The lexicon only pins what the letters say in isolation. -->
  <lexeme>
    <grapheme>ORD</grapheme>
    <alias>oh are dee</alias>
  </lexeme>

  <!-- Known misreads (defensive lexemes) =========================== -->
  <lexeme>
    <grapheme>reboot</grapheme>
    <phoneme>riːˈbuːt</phoneme>  <!-- some voices misstress as 'REE-boot' -->
  </lexeme>

</lexicon>
```

### 2.5 Lexicon design decisions and their rationale

| Decision | Choice | Why |
|---|---|---|
| File format | W3C PLS | Standard, versionable, provider-portable where supported. A plain JSON map is tempting but non-standard, and most providers accept PLS directly. |
| Alphabet | IPA | More widely consumed than x-SAMPA; one alphabet keeps the file greppable. Providers that only accept provider-specific markup (e.g. Azure phoneset) need a separate rendering. |
| Versioning | `.v3.pls` in the URI | Lexicon changes change *how the bot sounds*. Immutable-URL + version-bump lets you A/B two lexicons without a deploy. |
| Scope | One PLS per `(language, product-surface)` | Mixing all languages into one file turns every entry into a conflict-risk. One per locale keeps diffs auditable. |
| Entity classes covered | Brands, people, places, abbreviations, numbers-that-aren't | These are the five classes that cause 80 %+ of reported TTS misreads in a typical product corpus. |
| Alias vs phoneme | Alias for "say it as these letters"; phoneme only for things aliases can't express (stress, vowel colour) | Phonemes are harder to review in PR; aliases are proof-readable. Use aliases unless the stress/vowel/syllabification is actually what's wrong. |
| Defensive lexemes | Include known-misread common words | Some voices get common words wrong (`route`, `reboot`, `ate`). A one-line lexeme is cheaper than a lifetime of QA tickets. |

### 2.6 Provider conformance cross-walk

SSML and PLS are W3C standards; every provider documents their own subset. The matrix below is deliberately coarse — confirm against your provider's current docs at exercise time.

| Feature | Google Cloud TTS | Azure AI Speech | Amazon Polly | OpenAI | ElevenLabs | Deepgram Aura |
|---|---|---|---|---|---|---|
| `<prosody>` rate/pitch/volume | ✓ | ✓ | ✓ | ~ (limited) | ~ (style tags rather than SSML) | ~ |
| `<break time>` | ✓ | ✓ | ✓ | ~ | ✓ | ✓ |
| `<say-as>` | ✓ | ✓ | ✓ | partial | ~ | ~ |
| `<phoneme alphabet="ipa">` | ✓ | ✓ (ipa + x-sampa + microsoft-phoneset) | ✓ (ipa + x-sampa) | — | ~ | — |
| `<sub alias>` | ✓ | ✓ | ✓ | — | ✓ | ✓ |
| `<lang xml:lang>` | ✓ | ✓ | ✓ | — | — | — |
| `<mark>` | ✓ | ✓ (word/viseme events) | ✓ (speech marks) | — | ~ | ~ |
| `<lexicon uri>` | ✓ | ✓ | ✓ | — | — | — |
| Provider-specific style tags | voice-model-specific | `<mstts:express-as style="…">` | `<amazon:emotion name="…">` | Instruction-prompt style | Voice-settings controllability | Style via API arg |

Cells marked `~` or `—` do not mean the provider can't produce a good result — they mean the SSML path isn't the control surface, so the authored snippet must have a provider-specific fallback. The worked SSML above degrades cleanly: on a provider without `<lang>`, the Spanish fragment reads with an English accent; on a provider without `<lexicon uri>`, you inline a small number of `<phoneme>` / `<sub>` tags for the top misreads.

## 3. Validation steps

1. **XML is well-formed.** `xmllint --noout reply.ssml` and `xmllint --noout brand-en-us.v3.pls` return clean. (Use `pip install lxml; python -c "from lxml import etree; etree.parse('reply.ssml')"` if `xmllint` isn't installed.)
2. **Schema check.** Validate against the SSML 1.1 and PLS XSDs where available; if the provider also ships a schema (Azure does), validate against that too. `lxml`-based validate is acceptable.
3. **Provider dry-run.** Submit both documents to the chosen provider's synthesise endpoint; a 200 means the provider accepted every tag — not that it did what you meant.
4. **Listen-to-both check.** A/B the audio against the same text with an empty `<speak>` wrapper (no prosody, no breaks, no lexicon). Each tag should produce an *audible, correct* difference. A tag that produces no audible difference is a tag you don't need.
5. **Lexicon-swap test.** Change the lexicon URL to `brand-en-us.v2.pls` (older version). The output must change in the ways §2.4 predicts, and only in those ways.
6. **Provider-fallback test.** Rebuild the snippet for a provider from §2.6's `—` columns; the rebuild should still read correctly, falling back to the provider's control surface (style tags, instruction prompt, voice settings).
7. **Review pass.** A non-author reviewer reads §2.3 and can map each tag in the SSML to a one-sentence rationale.

## 4. Rubric / review checklist

| Dimension | Weight | Passing bar | Stretch |
|---|---|---|---|
| SSML well-formedness | 10 % | Validates against SSML 1.1 (XML + schema) | Also validates against provider-specific schema |
| Tag justification | 20 % | Every `<prosody>`, `<break>`, `<say-as>`, `<phoneme>`, `<sub>`, `<lang>`, `<mark>` has a one-line rationale | Rationale cites what breaks without the tag, with an audible example |
| Prosody restraint | 10 % | No pitch delta > ±3 st; no rate delta > ±20 %; deltas only where neutral is wrong | A/B against neutral shows each delta improves the delivery |
| Phoneme discipline | 10 % | `<phoneme alphabet="ipa">` is used only where alias/`<sub>` can't do the job; alphabet declared | IPA is proof-read by a speaker of the entity's source language |
| Lexicon coverage | 15 % | ≥ 5 entity classes covered; brands, people, places, abbreviations, numbers-that-aren't present | Defensive lexemes for known-misread common words |
| Lexicon versioning | 5 % | Immutable, versioned URL for the lexicon | Changelog records why each lexeme was added |
| Provider cross-walk | 10 % | A fallback for the chosen provider is authored if the provider is in §2.6's `—` columns | Two-provider portability demonstrated |
| Review audibility | 10 % | Listen-to-both test passes for every tag in the snippet | Mark-event subscription demonstrated end-to-end |
| Language declaration | 10 % | Every foreign-language fragment is wrapped in `<lang>` with xml:lang | Code-switched tokens that the lexicon covers also carry xml:lang on the lexeme |

Scoring < 70 % is a fail-and-redo. The most common sub-70 failure is over-tuned prosody — large `rate` and `pitch` deltas that make the voice sound performative rather than natural.

## 5. Common mistakes

- **Prosody tuning as decoration.** `<prosody rate="70%" pitch="-5st">` sounds broken, not emphatic. Keep deltas small; use them only where neutral is actually wrong.
- **Phonemes without an alphabet.** `<phoneme ph="wIt-sən-mAn">Weizenmann</phoneme>` without `alphabet="..."` is non-conformant; providers silently pick a default and your pronunciation drifts.
- **Alias masquerading as phoneme.** `<phoneme ph="ess kay you">SKU</phoneme>` is wrong; that's an alias, not phonemes. Use `<sub alias="ess kay you">`.
- **Inlining the lexicon.** 300 inline `<phoneme>` tags make every reply unreadable; the lexicon is a reusable, versioned artefact for a reason.
- **Reading numbers as numbers.** `ORD-10428-B` without `<say-as>` is unreadable aloud. So is `$149.99` without `interpret-as="currency"`.
- **Forgetting to language-switch.** English voices mangle "Clásico"; wrap in `<lang>`.
- **Breaks everywhere.** Pauses should be ~200–400 ms where the sentence structure demands one. A `<break time="1s"/>` between every clause feels patronising.
- **Marks for nothing.** `<mark>` tags are free in audio but noisy in event streams. Only mark what a subscriber will *consume*.
- **Treating provider style tags as portable.** `<mstts:express-as>` is Azure; `<amazon:emotion>` is Polly. Portable SSML keeps them behind a provider adapter.
- **Not listening to the output.** The final validation is always "listen to it." Readers don't hear SSML; they hear the result.

## 6. References

Standards:

- W3C Speech Synthesis Markup Language (SSML) Version 1.1 — `https://www.w3.org/TR/speech-synthesis11/`
- W3C Pronunciation Lexicon Specification (PLS) Version 1.0 — `https://www.w3.org/TR/pronunciation-lexicon/`
- Extensible SSML Version 1.0 Note — `https://www.w3.org/TR/ssml-xtended/`

Provider SSML/PLS documentation (verify URLs at exercise time; vendor doc trees are frequently reorganised):

- Google Cloud Text-to-Speech SSML reference — `https://cloud.google.com/text-to-speech/docs/ssml`
- Google Cloud Text-to-Speech custom pronunciations — `https://cloud.google.com/text-to-speech/docs/custom-pronunciations`
- Microsoft Azure AI Speech SSML reference — `https://learn.microsoft.com/azure/ai-services/speech-service/speech-synthesis-markup`
- Microsoft Azure AI Speech `<mstts:express-as>` — `https://learn.microsoft.com/azure/ai-services/speech-service/speech-synthesis-markup-voice`
- Microsoft Azure AI Speech custom lexicon — `https://learn.microsoft.com/azure/ai-services/speech-service/speech-synthesis-markup-pronunciation`
- Amazon Polly SSML tags — `https://docs.aws.amazon.com/polly/latest/dg/ssml.html`
- Amazon Polly lexicons — `https://docs.aws.amazon.com/polly/latest/dg/managing-lexicons.html`
- Amazon Polly speech marks — `https://docs.aws.amazon.com/polly/latest/dg/speechmarks.html`
- ElevenLabs text-to-speech markup — `https://elevenlabs.io/docs/speech-synthesis/prompting`
- Deepgram Aura SSML — `https://developers.deepgram.com/docs/tts-ssml`

Standards referenced indirectly:

- NIST AI Risk Management Framework — `https://www.nist.gov/itl/ai-risk-management-framework` — frames the "versioned, auditable" lexicon posture.

Source policy note. SSML 1.1 and PLS 1.0 are W3C standards and are the authoritative references for every tag used above. Provider cross-walk cells in §2.6 are coarse snapshots — confirm against each provider's current docs at exercise time; vendor subsets drift.
