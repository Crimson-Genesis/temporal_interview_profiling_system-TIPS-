# High-level pipeline (one line)

Record audio → Voice activity detection → (optional) Speaker diarization → ASR (speech → text) → Punctuation & capitalization → Linguistic / semantic analysis → Paralinguistic / prosodic analysis → Aggregate signals → Score / dashboard.

---

# Step-by-step with tools and what each step produces

## 1) Capture / recording

* Use the same webcam stream or a separate mic; sample rate **16 kHz or 48 kHz**, 16-bit PCM WAV preferred.
* Save per-interview file(s): `interview_<id>_mic.wav`.
* Tools: `arecord`/`ffmpeg` on Linux; in Python use `sounddevice` or `pyaudio`.

**Why**: high-quality single-channel audio reduces ASR errors and makes prosody extraction reliable.

---

## 2) Voice Activity Detection (VAD)

* Purpose: split audio into speech vs silence — reduces ASR cost and gives pause metrics.
* Tools:

  * WebRTC VAD (lightweight, realtime) — `webrtcvad` Python package.
  * `pyannote.audio` VAD (higher quality).
* Output: segments `[start, end]` of speech. Use to compute:

  * total speech time, total silence time, longest pause, mean pause duration.
* Why it matters: long/awkward pauses, frequent short pauses indicate hesitation or careful thinking.

---

## 3) (Optional but recommended) Speaker Diarization

* Purpose: in multi-person interviews (panel + candidate), split speech by speaker.
* Tools:

  * `pyannote.audio` (state-of-the-art diarization)
  * `webrtcvad` + clustering pipelines for simple cases
* Output: who spoke when (speaker labels + timestamps).
* Why: separate candidate vs interviewer; compute candidate speaking ratio, interruptions, turn-taking behavior.

---

## 4) Automatic Speech Recognition (ASR) — convert audio → text

* Options (local / open-source):

  * **OpenAI Whisper** (weights for local usage) — very robust to accents/noise.
  * **WhisperX** — Whisper + forced-alignment for word-level timestamps.
  * **VOSK** — lightweight, offline.
  * **Coqui STT / DeepSpeech** — older options.
* Cloud alternatives (higher accuracy & punctuation built-in): Google Speech-to-Text, AWS Transcribe, Azure Speech.
* Output:

  * transcript text (words)
  * timestamps for words/segments (if you use alignment/WhisperX)
  * ASR confidence scores
* Why: the transcript is the basis for semantic analysis, keyword spotting, and answer scoring.

---

## 5) Punctuation & capitalization & sentence segmentation

* Many ASR engines do not output punctuation. Punctuated text is essential for NLP.
* Tools:

  * Use models for punctuation restoration (e.g., transformers-based `ktrain` or HuggingFace seq2seq punctuation models).
  * Whisper sometimes returns punctuation; check model.
* Output: readable sentences — needed for NER, dependency parsing, and sentiment.

---

## 6) Forced alignment & word-level timestamps (important)

* Tools: **WhisperX**, `montreal-forced-aligner`, or `aeneas`.
* Output: exact word start/end times — needed to align prosody features with words (e.g., pitch during “confidence” words), and for highlighting hesitations.

---

## 7) Linguistic & semantic feature extraction (content analysis)

These are derived from the transcript.

**Features to extract**

* **Keywords / technical terms**: TF-IDF, domain-specific keyword lists, or simple substring checks.
* **Answer relevance**: semantic similarity between candidate’s answer and a “model answer” (use sentence embeddings like SBERT / `sentence-transformers`). Produce similarity score (0–1).
* **Readability & complexity**: Flesch–Kincaid, average word length, lexical diversity (type-token ratio).
* **Named Entities / factual mentions**: NER (spaCy) to find companies, tools, people, dates.
* **Claims & facts**: detect statements like “I built X”, “I led Y” — can be used to flag for followup.
* **Filler words rate**: count of “uh”, “um”, “like”, “you know” per minute.
* **Response length & verbosity**: words per answer, speaking rate (words/min).
* **Answer structure**: whether the candidate uses STAR format (Situation, Task, Action, Result) — you can check for phrases like “I did”, “We did”, “The result was…”.
* **Sentiment / emotion in content**: sentiment analysis (transformer classifier) to detect positive/negative language.

**Tools**: spaCy, NLTK, HuggingFace `transformers`, `sentence-transformers`, scikit-learn.

---

## 8) Paralinguistic / prosodic feature extraction (how they say it)

These come from raw audio + alignment.

**Features to extract**

* **Pitch (F0)**: mean, median, min, max, variance — via `librosa` (pyin) or `parselmouth` (Praat wrapper).
* **Intensity / loudness**: RMS energy per window.
* **Speech rate**: syllables per second (approx via words/time or vowel nuclei counting).
* **Pauses & hesitation**: number/duration of pauses, filled pauses.
* **Pitch contour dynamics**: rising/falling at sentence end (signals question vs statement/confidence).
* **Jitter & shimmer**: micro-variations related to voice steadiness (Parselmouth/Praat).
* **Confidence proxy**: high energy + steady pitch + faster speech often correlate with confidence (but not always).
* **Emotion / tone**: run an emotion classifier on audio features (openSMILE + pretrained models) to supplement visual emotion.
* **Voice quality**: breathiness, creak — sometimes relevant for stress.

**Tools**:

* `librosa` for STFT, RMS, zero-crossing, tempo
* `parselmouth` (Praat) for robust pitch/jitter/shimmer
* `opensmile` (via `python-opensmile`) — extracts standard ComParE/eGeMAPS feature sets (used in paralinguistics)
* `pyAudioAnalysis` for basic audio features

---

## 9) Dialog-level metrics (interaction quality)

* **Turn-taking**: average candidate/interviewer turn length.
* **Interruptions**: counts where the interviewer interrupts or candidate cuts off.
* **Responsiveness**: time between question end and candidate start (reaction latency).
* **Interruption ratio**: candidate vs interviewer.

---

## 10) Combine signals into evaluation metrics

Design interpretable composite scores:

* **Technical competency score**:

  * presence of technical keywords (weighted)
  * semantic similarity to expected answer
  * factual claims recognized
* **Communication clarity**:

  * speech rate in an ideal band (e.g., 120–160 wpm)
  * low filler-word rate
  * high lexical diversity & readability
* **Confidence / composure**:

  * low jitter/shimmer
  * steady pitch
  * short reaction latency
  * positive sentiment
* **Engagement & enthusiasm**:

  * energy (RMS), prosodic variation (not monotone), positive audio emotion
* **Cultural fit / soft skills**:

  * use of collaborative language: “we”, “I led”, “I helped”
  * mention of mentoring, teamwork, deliverables
* **Overall reliability**:

  * ASR confidence
  * transcript completeness (low filler interruptions)

Each composite score is a weighted sum of underlying features; make weights interpretable and tweakable.

---

## 11) Practical architecture (real-time vs offline)

* **Real-time streaming**:

  * VAD → small-chunk ASR (Whisper or streaming ASR), realtime punctuation model, live prosody extractor (frame-wise pitch + RMS), rolling aggregation for metrics.
  * Use websockets to send interim transcripts & metrics to UI.
* **Offline post-hoc**:

  * Process entire audio file with high-quality ASR + forced alignment + opensmile feature extraction + full NLP pipeline. Better accuracy, slower.

For interviews, real-time feedback can be helpful (e.g., interviewer dashboard), but save final scoring from offline pass.

---

## 12) Example outputs you should show in the interview dashboard

* Transcript (timestamped) with highlighted keywords and flagged claims.
* Timeline graph: emotions (audio+visual), speaking activity, filler words markers.
* Candidate summary cards:

  * Technical score, Communication score, Confidence score (0–100)
  * Top 5 keywords mentioned
  * Notable phrases & follow-up questions recommended (e.g., “Asked about scaling, candidate mentioned X; verify Y”)
* Raw audio snippets linked to timestamps for manual review (for any flagged concerns).

---

## 13) Privacy & ethics

* Get consent for recording.
* Keep data encrypted & access-controlled.
* Avoid overinterpreting emotion or confidence — present as supportive signals, not ground truth.
* Consider bias: some accents, neurodiversity, or speech differences can be unfairly penalized by acoustic measures.

---

## 14) Toolstack summary (quick reference)

* **Recording**: ffmpeg / sounddevice
* **VAD**: webrtcvad / pyannote
* **Diarization**: pyannote.audio
* **ASR**: Whisper / WhisperX (local) or Google/AWS (cloud)
* **Punctuation**: seq2seq punctuation model (HuggingFace)
* **Alignment**: WhisperX / Montreal Forced Aligner
* **Audio features**: openSMILE (eGeMAPS), parselmouth (Praat), librosa
* **NLP**: spaCy, sentence-transformers (SBERT), HuggingFace Transformers
* **Dashboard**: Streamlit / Flask + D3 / Grafana for telemetry
* **Storage**: S3 or local files (audio + transcript + metadata JSON)

---
