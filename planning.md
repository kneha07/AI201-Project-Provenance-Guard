# Provenance Guard — Planning Document

## Architecture

### Submission Flow

```
POST /submit
    │
    ▼
Validate request (text + creator_id required)
    │
    ▼
Signal 1: LLM via Groq (llama-3.3-70b-versatile)
    │         Returns ai_probability float [0.0, 1.0]
    │
    ▼
Signal 2: Stylometric Heuristics
    │         sentence length variance
    │         type-token ratio
    │         punctuation density
    │
    ▼
Combine signals (60% LLM, 40% stylometric)
    │
    ▼
Assign attribution label
    │   < 0.35  → likely_human
    │   0.35–0.65 → uncertain
    │   > 0.70  → likely_ai
    │
    ▼
Generate transparency label text
    │
    ▼
Log to SQLite (provenance.db)
    │
    ▼
Return JSON response to caller
```

### Appeal Flow

```
POST /appeal
    │
    ▼
Validate request (content_id + creator_reasoning required)
    │
    ▼
Look up content_id in database
    │
    ▼
Update status → "under_review"
Store creator_reasoning
    │
    ▼
Return confirmation to caller
    │
    ▼
(Human reviewer checks GET /log for under_review entries)
```

---

## Detection Signals

### Signal 1: LLM via Groq (weight: 60%)

We use the `llama-3.3-70b-versatile` model hosted on Groq to assess whether a piece of text reads as AI-generated. The model is prompted to act as an AI content detection expert and return a single JSON object with an `ai_probability` field. Temperature is set to 0.1 to get consistent, calibrated outputs. The 60% weight reflects that the LLM brings world-knowledge pattern recognition that stylometrics alone cannot replicate.

**Why chosen:** Large language models have seen both human and AI-generated text at scale. Their internal representations capture subtle distributional differences (hedging language, transition phrases, overly balanced sentence structure) that are hard to encode as explicit rules.

**Blind spots:** The LLM may be fooled by a creator who deliberately mimics AI prose or by AI-generated text that was lightly edited by a human. It is also subject to Groq API latency and rate limits.

### Signal 2: Stylometric Heuristics (weight: 40%)

Three sub-metrics are averaged to produce a stylometric score:

1. **Sentence length variance** — AI-generated text tends to produce uniformly structured sentences. Low variance across sentence lengths is interpreted as an AI signal.
2. **Type-token ratio (TTR)** — Vocabulary diversity. AI text often repeats the same word families or phrases; lower TTR indicates less diversity, raising the AI score.
3. **Punctuation density** — Human writers use more varied punctuation (em dashes, semicolons, parentheses, ellipses). A low density of these characters suggests AI prose.

**Why chosen:** Stylometrics are deterministic, fast, and do not require any external API call. They serve as a fallback if the LLM call fails and add a different axis of evidence that is harder to game simultaneously with the LLM signal.

**Blind spots:** Short texts (< 3 sentences) produce unreliable variance. Academic or technical human writing may score as AI-like due to formal prose conventions.

---

## Uncertainty Representation

The combined confidence score is a float in [0.0, 1.0] derived from the weighted average of both signals.

| Score range | Attribution label | Meaning |
|-------------|------------------|---------|
| < 0.35      | `likely_human`   | Strong evidence of human authorship |
| 0.35 – 0.65 | `uncertain`      | Neither signal is confident enough |
| > 0.70      | `likely_ai`      | Strong evidence of AI generation |

The "uncertain" band is intentionally wide (30 percentage points) because both signals have meaningful false positive/negative rates. Forcing a binary label on borderline content would be overconfident and misleading to creators. The uncertain label explicitly invites appeal rather than presenting a potentially wrong verdict as final.

---

## Transparency Label Design

All three label variants are presented verbatim to the end user in the API response under the `label` key.

**Variant 1 — likely_ai:**
```
Looks like AI-generated content ({pct}% confidence). Our system detected patterns
commonly found in AI writing. If you used an AI tool to help write this, consider
adding a note so readers know. Think this is wrong? You can file an appeal.
```

**Variant 2 — likely_human:**
```
Looks like human-written content ({pct}% confidence). Our system found strong signs
that a person wrote this — things like natural variation in sentence length and
personal punctuation habits. Great news for your attribution!
```

**Variant 3 — uncertain:**
```
We're not sure about this one ({pct}% confidence). The writing has some qualities
our system associates with AI and some it associates with humans. That can happen
with carefully edited work or content that crosses genres. If you wrote this
yourself, you can file an appeal and tell us why.
```

Design rationale: Each label names the verdict clearly in the first line so it is skimmable. The body sentence explains what the score means in plain language. The "uncertain" variant explicitly points users toward the appeals mechanism, reducing dead-ends.

---

## Appeals Workflow

1. Creator receives an attribution verdict they disagree with (typically `likely_ai` when the content is human-written, or `uncertain`).
2. Creator calls `POST /appeal` with their `content_id` and a free-text `creator_reasoning` field explaining why the verdict is wrong.
3. The system updates the submission record's `status` from `reviewed` to `under_review` and stores the reasoning text.
4. A human moderator periodically queries `GET /log` to inspect all `under_review` entries. They read the original attribution, both signal scores, and the creator's argument.
5. The moderator makes a final decision (accept or reject the appeal) and updates the record manually (or through a future `PATCH /appeal/:id/resolve` endpoint not yet implemented).
6. The creator is notified (out of scope for this prototype; would require email integration).

---

## Edge Cases

### Edge Case 1: Very short submissions (< 20 words)

A tweet-length text gives the stylometric signal almost no data to work with. Sentence variance is undefined or trivially small; TTR is artificially high because there are few repeated words. The system handles this by defaulting variance_score to 0.5 when fewer than 2 sentences are detected, effectively neutralizing that sub-metric and leaning more heavily on the LLM signal. Documentation should warn creators that short texts will frequently return `uncertain`.

### Edge Case 2: Groq API failure or timeout

If the Groq API is unreachable (network error, rate limit, malformed JSON response), the `llm_signal` function catches all exceptions and returns a neutral score of 0.5. This means the combined confidence score is anchored at 0.5 × 0.6 + stylometric × 0.4. The submission still completes and returns a result, but the label may be `uncertain` even for clearly AI-generated text. The API response does not currently expose whether the LLM call failed — a future improvement would be to add a `signals_used` field to the response JSON.

---

## AI Tool Plan

### M3 — Signal Design

Use Claude/ChatGPT to generate a diverse test corpus: 10 clearly AI-generated paragraphs (using GPT-4) and 10 clearly human-written paragraphs (forum posts, opinion pieces). Run both signals against this corpus to calibrate thresholds before finalizing the 0.35 / 0.65 cutoffs. Ask the AI assistant to suggest additional stylometric features beyond the three implemented (e.g., modal verb frequency, passive voice rate).

### M4 — Planning & Spec Review

Use an AI assistant to red-team the planning document — ask it to find edge cases and failure modes not covered above. Use it to draft the transparency label copy and iterate on wording until it is clear to a non-technical audience. Ask it to verify that the architecture diagram is internally consistent (e.g., confirm that the SQLite schema captures all fields referenced in the API response).

### M5 — Code Generation & Debugging

Use GitHub Copilot or Claude to scaffold boilerplate (SQLite helper functions, Flask route structure, rate limiter setup). Use AI assistance to write the regular expressions for stylometric parsing and to debug JSON parsing in `llm_signal` when the model returns extra prose around the JSON object. Ask AI for suggestions on how to improve test coverage for the stylometric functions.

---

## Stretch: Ensemble Detection (3-Signal)

### Signal 3: Modal Verb Density (weight: 20%)

AI-generated text systematically overuses hedging modal verbs — "would," "could," "may," "might," "should," "can," "shall," "must," "ought." This is a distinct stylistic tendency that arises because AI models hedge assertions to avoid being definitively wrong. Human writers use modals more sparingly and contextually.

**What it measures:** The count of modal verbs per 100 words of text.
**Output:** A float in [0.0, 1.0] where 1.0 means density ≥ 6 modals per 100 words (fully AI-like) and 0.0 means no modals detected.
**Blind spots:** Legal or academic human writing legitimately uses many modals. Technical specifications ("the system should…") will score as AI-like regardless of authorship.

### Updated Ensemble Weighting

| Signal | Weight | Rationale |
|--------|--------|-----------|
| LLM (Groq) | 50% | Broadest pattern recognition; most reliable overall |
| Stylometric | 30% | Structural/surface properties; deterministic fallback |
| Modal verb density | 20% | Captures a specific, narrow AI tell with low false-positive rate |

**Conflict resolution:** The weighted average implicitly handles conflicts — no single signal can override both others. A signal must shift the combined score by more than its own weight to flip the final attribution, which prevents outlier signals from dominating.

---

## Stretch: Analytics Dashboard

### GET /analytics

Returns three metrics in a single JSON response:

1. **Detection distribution** (`ai_ratio`, `human_ratio`, `uncertain_ratio`): The fraction of all submissions attributed to each category. Useful for monitoring whether the system is biased toward one label.
2. **Appeal rate** (`appeal_rate`): The fraction of submissions that have been appealed (status = `under_review`). A high appeal rate signals over-classification or miscalibration.
3. **Average confidence** (`average_confidence`): The mean combined confidence score across all submissions. Persistently high or low averages may indicate the scoring is poorly calibrated for the platform's typical content.
