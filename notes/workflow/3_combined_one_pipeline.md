Sweet — here’s the **final, unified pipeline** that combines **visual emotion detection** (face → emotion) and **audio analysis** (ASR + prosody + semantics) into one coherent interview-evaluation system. I’ll give you a clear data flow, component responsibilities, input/output contracts, performance targets, example data schemas, deployment options, failure modes, and a small checklist of “first actions” you can run. No code dump — just a precise, usable blueprint you can implement step-by-step.

I’ll present it as: **overview → stages → interfaces & formats → timing/latency targets → metrics & dashboard → deployment & scaling → testing → ethics & privacy → first actions**. Scroll to the part you want and say **“next”** to implement that chunk.

---

# ONE-LINE OVERVIEW

Live interview → Capture (video + audio) → VAD & face detection → crop/alignment → visual emotion classifier (batched) + audio ASR & prosody extraction (word-level aligned) → multimodal fusion & smoothing → scoring + logging → real-time dashboard + post-interview report.

---

# 1) High-level components (what each piece does)

1. **Capture** — reads camera frames & microphone audio (timestamps, frame IDs). Produces `Frame` and `AudioChunk`.
2. **VAD & Deskew** — voice-activity detection, chunking audio, silence detection.
3. **Face Detector** — per-frame detection (boxes + optional landmarks) → face crops + face IDs (track across frames).
4. **Crop & Align** — crop to faces, optionally rotate to align eyes horizontally.
5. **Visual Emotion Inference** — MobileNetV2 (RAF-DB compound) runs on batched face crops → per-face probabilities + timestamps.
6. **ASR + Alignment** — use Whisper/WhisperX (or streaming ASR) → transcript with word timestamps & confidence.
7. **Prosody & Paralinguistic Extractor** — from raw audio + alignment: pitch, energy, jitter/shimmer, pauses, speaking rate, filler words.
8. **Diarization (optional)** — separate interviewer(s) vs candidate voice; needed if panel interview.
9. **Multimodal Fusion** — align visual emotion probabilities with audio emotions/prosody + transcript semantics → create fused per-segment emotion & composite metrics.
10. **Temporal Smoothing & Aggregation** — moving averages, EMA, hysteresis to stabilize labels.
11. **Scoring Engine** — compute interpretable scores (tech, clarity, confidence, engagement) from features.
12. **Logger + Storage** — store raw audio, annotated video frames or bounding boxes, transcripts, feature JSONs.
13. **Dashboard / UI** — real-time overlays (bounding boxes + emotion labels), timeline graphs, per-question summaries, downloadable report.
14. **Supervisor & Health** — GPU monitor, process restarts, FPS monitoring, safe shutdown triggers on overheating.

---

# 2) Data flow & timing (frame-by-frame timeline)

* Capture produces:

  * Video frames at `F_fps` (e.g., 15–30 fps) with monotonic timestamps.
  * Audio stream at `Fs` (16k or 48k) with same clock.
* For every camera frame `t`:

  * Face Detector → boxes `B_i(t)` + landmarks `L_i(t)`
  * Tracking associates `face_id` across frames (IOU + re-id features)
  * If `face_id` is candidate: crop → queue for batching
* Audio operates continuously:

  * VAD identifies speech segments `S_j = [start, end]`
  * ASR processes segments into transcripts with word timestamps `W_k(start_k, end_k)`
  * Prosody extractor computes frame-level pitch/energy aligned to `W_k`
* Fusion:

  * For each time window (e.g., 500ms to 2s), gather:

    * visual emotion vector `v(t_window)` aggregated for candidate face(s)
    * audio emotion vector `a(t_window)` from prosody/emotion classifier
    * transcript tokens `w_k` in the window + semantic similarity to expected answers
  * Fuse into `fused_emotion = fuse(v, a)` (e.g., weighted sum, learned fusion network)
* Output per window:

  * `label`, `confidence`, `source_breakdown` (what visual/audio said), `scores` (tech/clarity/confidence)

---

# 3) Interfaces, formats & example JSON schema

All modules communicate via small JSON messages over Redis/ZeroMQ or in-memory queues (multiprocessing). Example schemas:

## Frame message (from Capture → Detector)

```json
{
  "frame_id": 12345,
  "timestamp": 1700000000.123,   // monotonic seconds
  "width": 1280,
  "height": 720,
  "image_format": "bgr",         // or "rgb"
  "image_ref": "/tmp/frame-12345.jpg" // or raw bytes in IPC
}
```

## Detection message (Detector → Cropper)

```json
{
  "frame_id": 12345,
  "timestamp": 1700000000.123,
  "detections": [
    {
      "face_id": null,
      "box": [0.12, 0.22, 0.40, 0.56], // [ymin,xmin,ymax,xmax] normalized
      "score": 0.92,
      "landmarks": {"left_eye":[x,y], "right_eye":[x,y]},
      "frame_ref": "/tmp/frame-12345.jpg"
    }
  ]
}
```

## Crop/Batch message (Cropper → VisualModel)

```json
{
  "batch_id": "batch-1",
  "items": [
     {"face_id": "f-123", "frame_id": 12345, "crop_ref": "/tmp/crop-1.jpg", "timestamp": ...}
  ]
}
```

## Visual inference output

```json
{
  "batch_id": "batch-1",
  "predictions": [
    {"face_id":"f-123","frame_id":12345,"timestamp":...,
     "probs": {"happy":0.67,"sad":0.02,"happily_surprised":0.15,...},
     "top": ["happy",0.67]}
  ]
}
```

## ASR & alignment (ASR → downstream)

```json
{
 "segment_id": "seg-42",
 "start": 1700000001.200,
 "end": 1700000004.800,
 "transcript": "I implemented a distributed queue using ZMQ...",
 "words": [
   {"word":"I","start":1.20,"end":1.24,"conf":0.98},
   ...
 ],
 "asr_confidence": 0.92
}
```

## Prosody output

```json
{
  "segment_id": "seg-42",
  "pitch_mean": 165.2,
  "pitch_std": 12.4,
  "rms_mean": 0.03,
  "jitter": 0.003,
  "filler_rate": 0.05
}
```

## Fused window output (to UI + Logger)

```json
{
  "window_id": "w-900",
  "start": 1700000001.5,
  "end": 1700000002.5,
  "face_id": "f-123",
  "visual": {"label":"happy","score":0.67},
  "audio": {"label":"positive","score":0.73,"pitch":150},
  "fused_emotion": {"label":"happily_surprised","score":0.78},
  "scores": {"communication": 0.82,"confidence":0.71,"technical":0.63},
  "transcript_summary": "Candidate described building a queue with ZMQ and retries"
}
```

---

# 4) Fusion strategies (simple → advanced)

* **Heuristic fusion**: weighted average of visual & audio softmax vectors (e.g., 0.6 visual, 0.4 audio), then argmax.
* **Rule-based**: prefer visual when audio SNR is low; prefer audio when visual crop size < threshold.
* **Learned fusion**: small neural net that takes `[v_probs, a_probs, prosody_feats, transcript_embeddings]` and outputs fused probs. Train on a small labeled multimodal emotion dataset (if possible).
* **Temporal fusion**: apply EMA or a sliding-window majority vote across last `T` windows.

---

# 5) Temporal smoothing & stability

* **Per-face buffer**: keep last `N` fused_probs. Compute EMA (α=0.6) or majority vote.
* **Commit rule**: only switch the displayed label if new label holds for `M` consecutive windows or confidence improvement > δ.
* **Uncertain handling**: if `max_prob < 0.45` → mark `uncertain` and show top-2.

---

# 6) Scoring engine (interpretable composites)

Design scores from normalized features (0..1):

* **Technical competency**:

  * `0.6 * semantic_similarity(model_answer, candidate_text)`

  - `0.4 * keyword_coverage_score`
* **Communication clarity**:

  * `0.5 * (1 - filler_rate_norm)`

  - `0.5 * lexical_diversity_norm`
* **Confidence**:

  * `0.5 * audio_confidence_metric (pitch stability & RMS)`

  - `0.5 * visual_confidence_metric (visual softmax entropy low)`
* **Engagement**:

  * `0.5 * speaking_ratio` + `0.5 * prosodic_variation`
* **Final composite**: weighted sum with tunable weights for your interview needs.

Store both raw features and aggregated scores for auditability.

---

# 7) Real-time constraints & targets

* **Frame capture**: 15–30 FPS
* **Face detection latency**: ≤ 50 ms per frame (SSD/BlazeFace on GPU)
* **Batching window**: 200–500 ms
* **Emotion inference latency**: ≤ 30–80 ms per batch (B=8)
* **ASR streaming latency**: < 1s (chunked WhisperX or streaming ASR)
* **End-to-end update interval**: UI updates every 300–500 ms (real-time feel)
* **Target total CPU+GPU utilization**: < 75% to keep headroom
* **Fallback**: if GPU temp > 85°C, reduce batch size / detector frequency

---

# 8) Storage, logging & replay

* Save:

  * raw audio file (WAV)
  * video metadata (frame refs or video file)
  * detection logs (jsonl)
  * ASR transcripts (with timestamps)
  * fused results & scores
* Provide replay UI: timeline slider, click timestamp → view frame & audio snippet & transcript.

---

# 9) Deployment options (dev → prod)

* **Single machine GPU demo**: All processes on laptop/desktop (best for interview demos).
* **Server-based**: capture client (web) streams to server(s) for heavy compute; use GPU-backed containers.
* **Hybrid**: do lightweight capture & VAD client-side; send chunks to server for ASR & fusion.
* **Containerization**: use Docker + NVIDIA runtime; orchestrate with docker-compose or Kubernetes for scale.
* **Edge/TFLite**: export emotion model to TFLite for CPU-only machines; do heavy ASR on server.

---

# 10) Monitoring & health

* **Metric endpoints**: FPS, detector latency, classifier latency, ASR latency, GPU temp/pstate, memory usage.
* **Alerts**: GPU overheat, process crash, ASR backlog.
* **Auto-scaling**: if server-side, add workers for emotion inference based on queue length.

---

# 11) Failure modes & mitigations

* **Low-light / occluded faces**: fallback to audio-only scoring for those windows.
* **Noisy audio**: run noise suppression (RNNoise) and fall back to visual cues.
* **ASR failure**: mark low ASR confidence and rely more on prosody/visual signals.
* **Network latency (server)**: degrade gracefully (lower sampling, smaller batches), show “delayed results”.
* **Privacy breach**: encrypt all stored audio & video, access controls.

---

# 12) Ethics & bias mitigation

* Get candidate consent; explain what is computed and how results are used.
* Use scores as **assistive** not deterministic — show raw evidence snippets for each score.
* Audit model performance across accents, genders, ages; adjust weights or retrain to reduce bias.
* Allow human override and exportable audit logs for fairness checks.

---

# 13) Example minimal runtime architecture (processes)

* `capture.py` → frames + audio → `frame_queue`, `audio_queue`
* `detector_worker.py` (TF on GPU) → `detections_queue`
* `crop_batcher.py` → `visual_batch_queue`
* `vision_infer.py` (TF on GPU) → `vision_results_queue`
* `asr_stream.py` (WhisperX) → `transcripts_queue`
* `prosody_worker.py` → `prosody_queue`
* `fusion_aggregator.py` (CPU/GPU) → `fused_results_queue`
* `renderer_ui.py` subscribes to fused_results for drawing & dashboard
* `logger` stores everything to disk/db

Use `multiprocessing` or Dockerized microservices with Redis/ZeroMQ.

---
