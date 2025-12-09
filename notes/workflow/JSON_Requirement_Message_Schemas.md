# JSON Requirements & Complete Message Schemas

## Implementation Notes (Global)

- Use UTC ISO 8601 for wall-clock timestamps (e.g., `2025-12-09T18:21:34.123Z`) and an additional monotonic float seconds field (`mono_ts`) for strict ordering across streams.
- All IDs are strings and should be globally unique (`UUIDv4` or `prefix-<hex>`).
- Keep message sizes small: avoid embedding full image bytes in the JSON; use `ref` fields (filesystem path, shared memory key, IPC handle, or presigned URL).
- Include `schema_version` in every top-level message for forward/backward compatibility.
- Use required fields listed below—optional fields are allowed but listed as optional.

---

## 1. Session Metadata (session start / config) — Session

**Topic/queue:** `session.meta`

```json
{
  "schema_version": "1.0",
  "session_id": "sess-7a3f2e4b",
  "created_at": "2025-12-09T18:00:00.000Z",
  "mono_start": 1712700000.000,
  "candidate": {
    "candidate_id": "cand-001",
    "name": "Abrarali Sunasara",
    "consent": true
  },
  "interview": {
    "type": "technical",
    "panel": ["intv-1","intv-2"],
    "expected_questions": ["Q1: explain ..."]
  },
  "capture_config": {
    "video_fps": 30,
    "video_resolution": [1280,720],
    "audio_sample_rate": 48000,
    "vad_model": "webrtc-vad-3"
  },
  "storage_policy": {
    "retain_days": 90,
    "encrypt": true
  }
}
```

**Fields (required):** `schema_version`, `session_id`, `created_at`, `mono_start`, `candidate`, `interview`, `capture_config`.

**Validation notes:** `consent` must be explicit `true` for processing to proceed.

---

## 2. Frame Message (Capture → Detector) — Frame

**Topic/queue:** `capture.frames`

```json
{
  "schema_version": "1.0",
  "frame_id": "frame-000001",
  "session_id": "sess-7a3f2e4b",
  "mono_ts": 1712700000.033,
  "timestamp": "2025-12-09T18:00:00.033Z",
  "width": 1280,
  "height": 720,
  "format": "bgr",
  "image_ref": "shm://frame-000001", 
  "exif": {
    "camera": "Logitech C920",
    "exposure": "1/30"
  }
}
```

**Fields (required):** `schema_version`, `frame_id`, `session_id`, `mono_ts`, `timestamp`, `width`, `height`, `format`, `image_ref`.

**`image_ref` options:** path (`/tmp/frame.jpg`), shared memory key (`shm://...`), IPC handle, or video container offset.

---

## 3. Detection Message (Detector → Cropper / Tracker) — Detections

**Topic/queue:** `vision.detections`

```json
{
  "schema_version": "1.0",
  "frame_id": "frame-000001",
  "session_id": "sess-7a3f2e4b",
  "mono_ts": 1712700000.033,
  "detections": [
    {
      "det_id": "det-9b2",
      "box": [0.12,0.22,0.40,0.56],
      "score": 0.92,
      "landmarks": {
        "left_eye": [0.20,0.30],
        "right_eye": [0.28,0.30],
        "nose": [0.24,0.36],
        "mouth_left": [0.21,0.42],
        "mouth_right": [0.27,0.42]
      },
      "face_embedding_ref": null, 
      "face_id": null, 
      "occluded": false
    }
  ],
  "frame_ref": "shm://frame-000001"
}
```

**Notes:** `box` is normalized `[ymin,xmin,ymax,xmax]`. `face_id` is `null` until tracking assigns one. `face_embedding_ref` may contain a ref to a 128-d embedding blob.

---

## 4. Track Assignment (Tracker → downstream) — TrackAssignment

**Topic/queue:** `vision.tracks`

```json
{
  "schema_version": "1.0",
  "frame_id": "frame-000001",
  "mono_ts": 1712700000.033,
  "assignments": [
    {
      "det_id": "det-9b2",
      "face_id": "f-123",
      "track_score": 0.98,
      "last_seen_mono": 1712699998.500
    }
  ]
}
```

**Notes:** `face_id` should be persistent across session. Provide `last_seen_mono` for eviction logic.

---

## 5. Crop/Batch Message (Cropper → VisualModel) — CropBatch

**Topic/queue:** `vision.crop_batches`

```json
{
  "schema_version": "1.0",
  "batch_id": "batch-1",
  "session_id": "sess-7a3f2e4b",
  "created_at": "2025-12-09T18:00:00.200Z",
  "items": [
    {
      "face_id": "f-123",
      "frame_id": "frame-000001",
      "mono_ts": 1712700000.033,
      "crop_ref": "shm://crop-f-123-frame-000001",
      "crop_size": [224,224],
      "align_meta": {"rot_deg": 1.4}
    }
  ],
  "batch_meta": {"batch_size": 1}
}
```

**Batching guidance:** keep `crop_size` uniform across batch; include `align_meta` for reproduction.

---

## 6. Visual Inference Output (VisualModel → fusion/logger) — VisualInference

**Topic/queue:** `vision.inference`

```json
{
  "schema_version": "1.0",
  "batch_id": "batch-1",
  "session_id": "sess-7a3f2e4b",
  "predictions": [
    {
      "face_id": "f-123",
      "frame_id": "frame-000001",
      "mono_ts": 1712700000.033,
      "probs": {
        "neutral": 0.12,
        "happy": 0.67,
        "surprised": 0.15,
        "confused": 0.06
      },
      "top": ["happy", 0.67],
      "model_name": "mobilenetv2-rafdb-v1",
      "inference_latency_ms": 23
    }
  ]
}
```

**Notes:** `probs` must sum (approximately) to 1. Include `model_name` and `inference_latency_ms` for monitoring and reproducibility.

---

## 7. Audio Segment/ASR Request & Result — AudioSegment & ASRSegment

**Topics/queues:** `audio.segments` (request) → `asr.results` (response)

### AudioSegment (incoming to ASR)

```json
{
  "schema_version": "1.0",
  "segment_id": "seg-42",
  "session_id": "sess-7a3f2e4b",
  "mono_start": 1712700001.200,
  "mono_end": 1712700004.800,
  "timestamp_start": "2025-12-09T18:00:01.200Z",
  "timestamp_end": "2025-12-09T18:00:04.800Z",
  "audio_ref": "shm://seg-42.wav",
  "sample_rate": 48000,
  "channels": 1,
  "vad_conf": 0.98
}
```

### ASRSegment (ASR → downstream)

```json
{
  "schema_version": "1.0",
  "segment_id": "seg-42",
  "session_id": "sess-7a3f2e4b",
  "mono_start": 1712700001.200,
  "mono_end": 1712700004.800,
  "transcript": "I implemented a distributed queue using ZMQ with retries.",
  "asr_confidence": 0.92,
  "words": [
    {"word":"I","start":1.20,"end":1.24,"conf":0.98},
    {"word":"implemented","start":1.24,"end":1.50,"conf":0.87}
  ],
  "asr_model": "whisper-large-v2-offline",
  "latency_ms": 720
}
```

**Notes:** Word timestamps are relative to `mono_start` or use absolute `mono_ts` (choose one—document it). Recommended: use absolute `mono = mono_start + word.start`.

---

## 8. Prosody & Paralinguistic Output — Prosody

**Topic/queue:** `audio.prosody`

```json
{
  "schema_version": "1.0",
  "segment_id": "seg-42",
  "session_id": "sess-7a3f2e4b",
  "mono_start": 1712700001.200,
  "mono_end": 1712700004.800,
  "pitch_mean_hz": 165.2,
  "pitch_std_hz": 12.4,
  "rms_mean": 0.032,
  "jitter": 0.003,
  "shimmer": 0.0012,
  "speech_rate_wpm": 120,
  "filler_rate": 0.05,
  "pauses": [
    {"start": 1.50, "end": 1.62, "duration": 0.12}
  ],
  "vad_segments": [{"start":1.20,"end":1.80}],
  "prosody_model": "pyannote-prosody-v1"
}
```

**Normalization:** provide both absolute and normalized features where helpful. `speech_rate_wpm` is words-per-minute across the segment.

---

## 9. Diarization Output (optional) — Diarization

**Topic/queue:** `audio.diarization`

```json
{
  "schema_version": "1.0",
  "session_id": "sess-7a3f2e4b",
  "mono_start": 1712700000.000,
  "mono_end": 1712700300.000,
  "speakers": [
    {"speaker_id": "spk-1", "label": "candidate"},
    {"speaker_id": "spk-2", "label": "interviewer"}
  ],
  "segments": [
    {"speaker_id":"spk-2","start":0.5,"end":2.0,"mono_start":1712700000.5,"mono_end":1712700002.0},
    {"speaker_id":"spk-1","start":2.1,"end":6.0,"mono_start":1712700002.1,"mono_end":1712700006.0}
  ],
  "diarization_model": "pyannote-diarize-v2",
  "conf": 0.91
}
```

**Notes:** `label` is optional human-friendly mapping. `mono_start`/`mono_end` on segments recommended for direct alignment.

---

## 10. Fusion Window (Fusion → UI / Logger) — FusionWindow

**Topic/queue:** `fusion.windows`

```json
{
  "schema_version": "1.0",
  "window_id": "w-900",
  "session_id": "sess-7a3f2e4b",
  "start_mono": 1712700001.500,
  "end_mono": 1712700002.500,
  "face_id": "f-123",
  "visual": {"label":"happy","probs":{"happy":0.67,"neutral":0.12},"source":"mobilenetv2"},
  "audio": {"label":"positive","score":0.73,"pitch":150,"source":"prosody-v1"},
  "transcript_tokens": [
    {"word":"ZMQ","mono_start":1712700001.710, "mono_end":1712700001.750, "conf":0.85}
  ],
  "fused_emotion": {"label":"happily_surprised","score":0.78,"fusion_method":"weighted_v0.2"},
  "scores": {"communication":0.82,"confidence":0.71,"technical":0.63},
  "uncertainty": {"fused_entropy":0.55},
  "meta": {"fusion_latency_ms": 45}
}
```

**Notes:** `fusion_method` = e.g., `heuristic:0.6V-0.4A`, `learned:v1`. Include `uncertainty` to expose ambiguous windows.

---

## 11. Per-Question Summary (Q/A mapper → dashboard) — QuestionSummary

**Topic/queue:** `qa.summaries`

```json
{
  "schema_version": "1.0",
  "question_id": "q-01",
  "session_id": "sess-7a3f2e4b",
  "question_text": "Explain how you implemented the queue with retries.",
  "asker_id": "intv-1",
  "answer": {
    "start_mono": 1712700001.200,
    "end_mono": 1712700018.000,
    "transcript": "I implemented a distributed queue using ZMQ with retries and backoff...",
    "technical_score": 0.63,
    "communication_score": 0.78,
    "confidence_score": 0.71,
    "highlights": ["mentioned backoff", "covered retry logic"],
    "weaknesses": ["no LRU eviction mention"]
  }
}
```

**Notes:** This message is used to show per-question analytics and anchor timeline segments.

---

## 12. Session Final Report (Logger → storage / dashboard) — SessionReport

**Topic/queue:** `session.reports`

```json
{
  "schema_version": "1.0",
  "session_id": "sess-7a3f2e4b",
  "generated_at": "2025-12-09T18:45:00.000Z",
  "summary": "Candidate showed good communication and moderate technical depth.",
  "overall_scores": {"technical":0.68,"communication":0.81,"confidence":0.73,"composite":0.74},
  "per_question": [
    {"question_id":"q-01","technical":0.63,"communication":0.78,"confidence":0.71}
  ],
  "evidence_refs": [
    {"type":"audio","ref":"storage://sess-7a3f2e4b/seg-42.wav"},
    {"type":"frame","ref":"storage://sess-7a3f2e4b/frame-000120.jpg"}
  ],
  "artifacts": {
    "transcript_ref": "storage://sess-7a3f2e4b/transcript.json",
    "fused_json_ref": "storage://sess-7a3f2e4b/fused_results.jsonl"
  },
  "auditable": true,
  "notes": "Check ASR low confidence around 00:02:15; suggest human review."
}
```

**Notes:** `evidence_refs` must be encrypted/presigned per Session rules if stored.

---

## 13. Health & Monitoring Message — HealthMetrics

**Topic/queue:** `monitoring.metrics`

```json
{
  "schema_version": "1.0",
  "node_id": "node-gpu-01",
  "session_id": "sess-7a3f2e4b",
  "timestamp": "2025-12-09T18:10:00.000Z",
  "gpu": {"util_percent": 56, "temp_c": 69, "memory_used_mb": 6144},
  "cpu": {"util_percent": 32, "load_1m": 0.82},
  "queues": {"vision.crop_batches": 2, "asr.results": 0},
  "alerts": []
}
```

**Notes:** Use these metrics to trigger autoscaling or fallback behaviors.

---

## 14. Alert / Error Message — Alert

**Topic/queue:** `monitoring.alerts`

```json
{
  "schema_version": "1.0",
  "alert_id": "alert-001",
  "session_id": "sess-7a3f2e4b",
  "timestamp": "2025-12-09T18:12:00.000Z",
  "severity": "critical",
  "component": "vision.inference",
  "message": "GPU temperature exceeded threshold",
  "details": {"gpu_temp_c": 86},
  "actions": ["reduce_inference_batch", "notify_admin"]
}
```

**Severity options:** `info`, `warning`, `critical`. Provide `actions` for automated responses.

---

## 15. Storage Manifest / Archival Message — StorageManifest

**Topic/queue:** `storage.manifests`

```json
{
  "schema_version": "1.0",
  "session_id": "sess-7a3f2e4b",
  "manifest_generated_at": "2025-12-09T18:46:00.000Z",
  "items": [
    {"type":"audio","ref":"s3://interviews/sess-7a3f2e4b/seg-42.wav","size_bytes": 1234567,"encrypted":true},
    {"type":"transcript","ref":"s3://interviews/sess-7a3f2e4b/transcript.json","size_bytes": 2345,"encrypted":true}
  ],
  "retention_until": "2026-03-09T18:00:00.000Z"
}
```

**Notes:** Use for audit and GDPR/consent logging.

---

## Queue/Topic Mapping (Recommended)

| Topic/Queue | Purpose |
|------------|---------|
| `session.meta` | Session start/config |
| `capture.frames` | Raw frames |
| `vision.detections` | Detector outputs |
| `vision.tracks` | Tracking assignments |
| `vision.crop_batches` | Batched crops requests |
| `vision.inference` | Visual model outputs |
| `audio.segments` | VAD/audio chunks |
| `asr.results` | ASR outputs |
| `audio.prosody` | Prosody/paralinguistic features |
| `audio.diarization` | Diarization results (optional) |
| `fusion.windows` | Fused multimodal windows |
| `qa.summaries` | Per-question summaries |
| `session.reports` | Final report |
| `monitoring.metrics` | Node metrics |
| `monitoring.alerts` | Alerts/errors |
| `storage.manifests` | Archival manifest |

**Pattern:** Use `component.message_type` for clarity and to support ACLs.

---

## Validation Rules & Schema Versioning

1. **Schema Version:** Every message MUST include `schema_version`. When you change a schema, increment to `1.1`, `2.0`, etc. Modules should validate `schema_version` compatibility at startup.

2. **Timestamps:** Include both `timestamp` (ISO 8601) and `mono_ts` (float seconds). If only one is present, consumer must normalize. **Prefer both.**

3. **Size Limits:**
   - Frame JSON: < 1 KB if only refs used.
   - CropBatch: `batch_meta.batch_size` <= 32 (recommendation).
   - VisualInference: include `model_name` and `inference_latency_ms`.

4. **Security:** Confidential fields (names, storage refs) must be encrypted/signed; enforce ACLs at queue level.

5. **Low Confidence:** If `asr_confidence < 0.5`, tag message with `low_confidence: true` in `ASRSegment`.

6. **Compatibility:** Use `schema_version` compatibility maps in orchestration config.

---

## Example End-to-End Timeline (Short)

1. `session.meta` created.
2. `capture.frames` published at 30 FPS (refs only).
3. `vision.detections` produced per frame.
4. `vision.tracks` assigns `face_id`.
5. `vision.crop_batches` created every 200ms.
6. `vision.inference` returns emotion probs.
7. `audio.segments` created by VAD; `asr.results` and `audio.prosody` published.
8. `fusion.windows` created every 500ms and published to `fusion.windows`.
9. `qa.summaries` created per detected question.
10. `session.reports` generated at session end and `storage.manifests` saved.

---

## Next Steps

For machine-usable validation, consider generating:

- **Full JSON Schema files** (Draft 7 / Draft 2020-12) for all message types
- **Python validators** using Pydantic
- **TypeScript interfaces** with Zod validation
- **Kafka topic config** with retention/security settings

---

**Note:** This specification ensures consistency, auditability, and forward compatibility across all pipeline modules.
