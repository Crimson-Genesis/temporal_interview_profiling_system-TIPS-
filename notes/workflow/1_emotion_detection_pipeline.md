# Pipeline (one line)

Webcam → Face detector → Crop & align → Batch & preprocess → MobileNetV2 (fine-tuned on RAF-DB compound labels) → Postprocess (softmax → label) → UI overlay + logging → (optional) TFLite export.

---

# 1. Components & responsibilities

* **Capture (producer)**: read frames from webcam (OpenCV), push into an input queue.
* **Face detector (worker)**: fast TF model (BlazeFace or TF-Hub SSD MobileNet V2) — returns normalized boxes & scores.
* **Crop & align (CPU thread)**: convert boxes → pixel coords, crop, optional face alignment (eyes horizontal) for better emotion accuracy.
* **Batcher (worker)**: accumulate crops into batches (size 8–16) for emotion model efficiency.
* **Emotion classifier (GPU worker)**: Keras/TensorFlow MobileNetV2 fine-tuned on RAF-DB compound labels; infer on batched crops.
* **Postprocessor & UI (consumer)**: take returned probs, map to labels, render bounding box + label + confidence on the original frames; compute FPS and log results.
* **Supervisor**: health checks, GPU temperature/pstate monitor, restart model workers if they crash.

---

# 2. Face detection stage (details)

* **Model choices**: BlazeFace (very fast), TF-Hub SSD MobileNet V2 (easy TF), or converted YOLOv8 face if you want extreme speed.
* **Input type**: full frame as `uint8` RGB (or BGR then convert).
* **Preferred runtime**: TensorFlow on GPU for this stage if using TF-Hub; if you pick BlazeFace TFLite, could run on CPU for even lower overhead.
* **Output**:

  * `detection_boxes` — shape `[N, 4]` in normalized `[ymin, xmin, ymax, xmax]`
  * `detection_scores` — shape `[N]`
  * `detection_classes` — (not needed, we only care faces)
* **Thresholds**: accept `score >= 0.4` by default for live; raise to `0.6` if many false positives.
* **Run frequency**: detect every frame if fast detector; or every *k* frames (k=1..3) if you want lower CPU/GPU load and you track faces between detections.

---

# 3. Crop & alignment (why and how)

* **Crop**: convert normalized coords → pixel coords:

  ```
  x1 = int(xmin * W), y1 = int(ymin * H), x2 = int(xmax * W), y2 = int(ymax * H)
  face = frame[y1:y2, x1:x2]
  ```
* **Margin**: add a small margin (e.g., 10–20% of box size) to include context (chin, hair).
* **Alignment (optional but recommended)**:

  * Use facial landmarks (if detector provides them) or a small landmark model (eyes/nose).
  * Rotate the crop to make eyes horizontal. This reduces intra-class variance for emotion classifier.
* **Minimum size**: skip faces smaller than e.g., 40×40 px — MobileNetV2 needs decent resolution.

---

# 4. Preprocessing for emotion model

* **Color**: RGB (consistent with how model was trained).
* **Resize**: `224 × 224` (MobileNetV2 default). If you trained on a different size (e.g., 112 or 128) use that.
* **Normalization**: float32, scale pixels to `[0, 1]` (divide by 255). If your training used mean/std normalization, apply same transform.
* **Batching**: collect crops into a batch tensor shape `[B, 224, 224, 3]`. Typical batch sizes: `B = 8 or 16` for best GPU throughput on RTX3050.
* **Dtype**: `tf.float32`. Use mixed precision (float16) in training & inference for speed if you enable TF mixed precision on GPU.

---

# 5. Emotion classifier (model & training notes)

* **Backbone**: MobileNetV2 (pretrained on ImageNet), head replaced with:

  * GlobalAveragePooling
  * Dense(512) optional + BatchNorm + ReLU (optional)
  * Dense(num_classes) → softmax
* **Num classes**: RAF-DB compound set. Typical compound set is 11–12 classes (list below).
* **Loss**: categorical crossentropy (or focal loss if class imbalance is heavy).
* **Optimizer**: Adam or SGD with cosine/step decay LR. Start low LR for fine-tuning (e.g., `1e-4` or `1e-5` for backbone).
* **Augmentation (important)**:

  * horizontal flip (if dataset supports)
  * random brightness & contrast
  * gaussian blur, small rotations (±15°)
  * color jitter, cutout
  * random zoom/crop
  * MixUp or CutMix can help generalization
* **Class imbalance handling**:

  * weighted loss (class weights) or oversampling minority classes.
* **Evaluation metrics**: Accuracy, per-class recall, F1, and confusion matrix. In interviews you can show top-1 accuracy + confusion for candidate predictions.

---

# 6. RAF-DB compound labels (example set)

Compound labeling variants differ across repos; a common 11/12-class compound set:

1. Neutral
2. Happy
3. Sad
4. Angry
5. Surprise
6. Fear
7. Disgust
8. Contempt
9. Happily surprised (compound)
10. Sadly angry (compound)
11. Fearfully surprised (compound)
12. ... (other compound like disgust+anger or mixed)

> When you fine-tune, pick the exact label mapping used by the checkpoint/data you use; keep a `label_map.json` and load it in inference.

---

# 7. Inference outputs & decision rules

* **Raw output**: softmax probabilities `p[class]`.
* **Label**: `argmax(p)` → predicted class.
* **Confidence**: `max(p)`.
* **Secondary/polytomous reporting**: if `max(p) < 0.45`, label as `uncertain` or `neutral`. Or show top-2 labels if you want nuance.
* **Smoothing over time**: for live webcam, apply temporal smoothing to reduce flicker:

  * Simple moving average of softmax probabilities over last `T` frames (T=5–10).
  * Or an exponential moving average (EMA) to weigh recent frames more.

---

# 8. Real-time & multiprocessing architecture (practical)

Use a producer-consumer pattern with bounded queues:

* **Process A (Capture)**: reads frames, pushes to `queue_frames`.
* **Process B (Detector)**: pops from `queue_frames`, runs detection, pushes `(frame, boxes)` to `queue_detected`.
* **Process C (Crop & Batcher)**: pops from `queue_detected`, crops faces, batches crops into `[B,224,224,3]`, pushes batches to `queue_batches`.
* **GPU Process D (Emotion infer)**: pops batches, performs inference, pushes results to `queue_results`.
* **Process E (Renderer/UI)**: reads `queue_results` and original frame, draws boxes and labels, shows frame.

Implementation notes:

* Use `multiprocessing.Process` with `multiprocessing.Queue` for cross-process isolation and to avoid Python GIL.
* Pin GPU work to one process to avoid context thrash — keep the TF model in a single process for inference (multi-threaded inside).
* Use `torch.multiprocessing` / TF `tf.data` if integrating with those ecosystems — but single TF process for model inference is simplest.

---

# 9. Performance tuning & safety

* **Mixed precision**: enable `tf.keras.mixed_precision.set_global_policy('mixed_float16')` during inference to reduce latency and memory.
* **Batching**: inference B=8–16 to maximize throughput; but keep latency acceptable (larger B increases latency).
* **Face detection frequency**: if detector is heavy, run every 2–3 frames and track faces between detections (simple IoU-based tracker).
* **GPU memory**: warm up model once (do a dummy inference) so TF allocates GPU memory and avoids spikes later.
* **Thermal checks**: monitor `nvidia-smi` pstate and temp; if temp gets >85°C, throttle inference (reduce input size or batch size).
* **CPU-GPU copies**: minimize host-device copies; prepare batches on CPU pinned memory and transfer once.

---

# 10. Logging, metrics & UX

* **Log**: timestamp, frame index, face bbox, predicted label, confidence, inference time (ms) for both detector and classifier.
* **Metrics dashboard**: FPS (camera), detector latency (ms), classifier latency (ms), total pipeline latency, GPU util/temp.
* **Visual UX**:

  * draw bounding box + label (text with confidence)
  * color-code emotions (e.g., green=happy, red=angry)
  * optional small timeline showing emotions over the last 30 seconds for the candidate
* **Privacy**: you’re doing interviews — log cautiously, anonymize or encrypt saved videos/results if required.

---

# 11. Training / fine-tuning plan (if you want custom accuracy)

* **Data**: RAF-DB compound training set; augment heavily.
* **Split**: train/val/test (e.g., 80/10/10).
* **Training recipe**:

  * Freeze backbone for first `N` epochs (e.g., 3–5) with a higher LR on head.
  * Unfreeze and fine-tune whole network with lower LR.
  * Use LR scheduler (cosine or step decay).
  * Early stopping based on val loss / val F1.
* **Checkpointing**: save best-model by val F1.
* **Quantization**: after training, test post-training quantization and/or convert to TFLite for smaller model on CPU.

---

# 12. Export & deployment options

* **SavedModel** (TensorFlow) — easiest for serving.
* **TFLite** with float16 or int8 quantization — ideal for CPU-only demo machines or embedded devices.
* **ONNX** conversion (optional) — for cross-framework compatibility.
* **Docker**: package the whole stack with GPU support (nvidia-container-toolkit) for easy reproducible demos.

---

# 13. Failure modes and mitigations

* **Small faces or occlusions** → lower accuracy. Mitigate with higher-res capture / better detector / add more small-face images to training.
* **Lighting changes** → augment with brightness/contrast during training.
* **Model drift (candidate population differs)** → collect some in-the-wild interview images and fine-tune.
* **Flicker/jitter** → apply temporal smoothing and require N consecutive frames before committing to a label.
* **Ethical pitfalls** → emotions are noisy proxies — always present results as suggestions, not absolute truths.

---

# 14. Practical hyperparameters (defaults you can use now)

* face detector score threshold = `0.5`
* min face pixel size = `40` px
* crop margin = `0.15` (15% of box size)
* emotion model input size = `224 × 224`
* batch size for inference = `8`
* softmax confidence threshold for acceptance = `0.45` (below → show `uncertain`)
* smoothing window = `ema_alpha = 0.6` or simple moving average over last `T=7` frames
* target runtime latency ≤ `80 ms` per frame (for acceptable real-time feel) — tune batch size and detector frequency to achieve this

---


