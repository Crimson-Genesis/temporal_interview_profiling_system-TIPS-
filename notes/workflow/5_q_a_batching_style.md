Perfect plan — you’ve got the right high-level approach. I’ll condense that into a tight, actionable blueprint you can follow **one step at a time**, plus concrete recommendations and the *very next* minimal action to start the visual part.

---

## Short summary of your workflow (confirmed)

1. **Visual emotion detection** (face → crop → emotion) — run locally.
2. **Audio → ASR + diarization** to separate interviewer vs candidate.
3. **Build Q↔A map** by aligning interviewer turns (questions) with candidate turns (answers).
4. **Per-QA batching**: after each Q→A transaction, create a compact evidence bundle (visual events, audio prosody, transcript, semantic similarity) and send that to the LLM to get a human-like QA summary.
5. **Realtime & final**: realtime per-QA short summaries during the interview; final full-report at the end (merge per-QA summaries + global signals).
6. **Dashboard**: timeline, per-QA cards, evidence links (audio/video snippets), final report export.

Yes — that flow is right. Batching per Q→A transaction is *the best* practical choice: it’s natural, interpretable for humans, keeps LLM inputs small, and mirrors how interviewers think (question-by-question).

---

## Why per-QA batching is a good design (quick reasons)

* **Natural units**: interviewer asks, candidate answers → meaningful chunk of content.
* **Bounded context**: keeps prompts small so LLMs are cheaper/faster and less likely to hallucinate.
* **Good alignment**: you can attach evidence (timestamps, cropped frames, audio snippets) to a specific question.
* **Easier auditing**: manager can jump to Q3’s evidence to verify any claim.
* **Realtime-friendly**: produce short per-QA summaries immediately once the candidate finishes speaking.

---

## How to detect a “QA transaction” automatically (robust recipe)

* **Diarize** (or at least detect speaker roles). Label segments `INTERVIEWER` and `CANDIDATE`.
* Mark a **question start** at the beginning of an interviewer speech segment and **question end** when interviewer goes silent (or when candidate starts speaking).
* Mark candidate **answer start** at candidate segment start, **answer end** at candidate segment end (or with a timeout if silence > X seconds).
* Use heuristics to handle multi-turn follow-ups:

  * If interviewer interjects within the candidate answer, treat as continuation of Q (same QA) if interjection < 2s and contains a short prompt like “uh-huh” or “and?”.
  * If interviewer asks a follow-up new question (longer, with wh-words), start a new QA.

**Timeouts**:

* If no speech for `> 6s` after candidate starts, mark the answer ended (and push the QA batch).
* If candidate speaks for `> 3 min` (edge case), auto-close the answer and push partial batch.

---

## QA-batch schema (compact JSON to feed the LLM per QA)

Use this exact compact shape for each QA batch — small & grounded:

```json
{
  "qa_id": "q-7",
  "question": {
    "text": "What architecture did you use for the queue?",
    "start": 1700000100.12, "end": 1700000106.20, "asr_conf": 0.96
  },
  "answer": {
    "text": "I used ZeroMQ with retries, backoff, and idempotent workers...",
    "start": 1700000107.00, "end": 1700000115.10, "asr_conf": 0.92
  },
  "visual_events": [
    {"time":1700000108.5, "label":"happy", "score":0.72, "frame":"frame_1085.jpg"},
    {"time":1700000111.7, "label":"thinking", "score":0.55, "frame":"frame_1117.jpg"}
  ],
  "prosody": {"pitch_mean":160,"pitch_std":9,"filler_rate":0.03,"speaking_rate_wpm":120},
  "semantic": {"similarity_to_model_answer":0.78},
  "meta": {"candidate_id":"C001","interviewer_id":"I01","duration":8.1}
}
```

* Keep `question` + `answer` text short (trim long transcripts to top-K sentences or use summary-in-chunking).
* Include direct evidence references (short transcript snippets, frame filenames, audio offsets) so the LLM must cite them.

---

## Prompt templates (drop-in, safe & evidence-anchored)

### Realtime per-QA prompt (short)

```
You are a concise interview assistant. Using the JSON below, produce:
1) One-sentence summary of the candidate's answer (neutral).
2) One short technical accuracy rating (0-10) with one-line evidence.
3) Two suggested follow-ups (one technical, one behavioral).
4) One short evidence list (timestamp or quote).

JSON:
<<QA_JSON>>

Output format:
SUMMARY: <sentence>
TECH_SCORE: <0-10> — EVIDENCE: <short>
FOLLOWUPS: - <tech question> - <behavioral question>
EVIDENCE: - <item> - <item>
```

### Final-report prompt (merge per-QA summaries)

When interview ends, pass all QA batches (or chunk them) and ask for:

* TL;DR, strengths, weaknesses, scorecard (Technical / Communication / Confidence), top 3 evidence snippets, 3 follow-ups.

Always require `EVIDENCE` lines for each main claim.

---

## Fusion + weighting rules for per-QA scoring

Simple deterministic rule works well initially:

```
tech_score = round(10 * (0.7 * semantic_similarity + 0.3 * keyword_coverage))
communication_score = round(10 * (0.6*(1 - filler_rate_norm) + 0.4*speaking_rate_score))
confidence = round(10 * (0.5*visual_confidence + 0.5*(1 - prosody_jitter_norm)))
```

Tune weights after you collect a few interviews. Keep formulas in config for transparency.

---

## Realtime cadence & UX

* Trigger per-QA LLM summary **immediately when the candidate's answer ends** (or after a short debounce, e.g., 0.6s of silence), so interviewer sees a short card before moving on.
* Limit frequency: only one output per QA; if candidate speaks longer and interviewer interjects, update QA instead of creating new ones if within the same topic.
* Show "evidence" inline: small transcript snippet + 2 thumbnails + audio link — interviewer can click to hear the exact moment.

---

## Fail-safe & hallucination controls

* **Always** attach the `EVIDENCE` array returned in the prompt — LLM must cite timestamps or text extracts.
* Post-LLM **verifier**: check each claim maps to a real evidence entry in the QA JSON (simple exact match or fuzzy check). If verifier fails, mark claim as “unsupported” and remove it.
* Limit LLM context size: send only that QA JSON (plus maybe last 2 QA summaries for context) — never the whole interview in one prompt unless chunking.

---

## How to build final report from per-QA outputs

* Collect all per-QA summaries (short), technical scores, and follow-ups.
* Merge: compute averages for scorecard; list consistent strengths/weaknesses appearing in ≥2 QAs as top items.
* Pass combined compact summary + top QA transcripts to the final-report LLM prompt (with the evidence requirement).
* Produce final PDF/HTML with clickable evidence.

---

## Practical considerations for latency & cost

* Use a small local LLM for realtime (gpt4all / llama.cpp quantized) to keep latency < 2–6s per QA. If you need higher-quality narrative, do an offline pass with a stronger LLM (or cloud) after interview.
* Keep per-QA prompts short (< 500 tokens) to reduce compute cost and speed up responses.

---
