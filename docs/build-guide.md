# Build guide

A step-by-step path to building Neuraa from scratch, written for an engineer learning computer vision and ML as they go.

## How to use this guide

- **Each phase ships something that runs.** Don't move on until the "Done when" check passes.
- **Learn just-in-time.** The "Learn" list in each phase is what you need for *that* phase. Study it when you hit it, not before. Skip anything you already know.
- **Benchmark and document as you go.** Every experiment produces a number that goes in `docs/research-log.md`, and every notebook gets a status flip in `docs/roadmap.md`. This habit is half the value of the project.
- **Expect to get stuck.** Detection bugs are almost always coordinate conventions. Training bugs are almost always the data. Budget time for debugging — it's where the learning happens.

## Prerequisites

Assumed: comfortable with Python, basic NumPy, and the command line. If you've never trained or run a neural network, that's fine — you'll pick it up in Phase 1.

Worth brushing up first: what a tensor is, what a pretrained model is, and the difference between CPU and GPU inference. A free GPU (Google Colab or Kaggle) is enough for the whole research layer.

---

## Phase 0 — Foundations and setup

**Goal.** Get the repo, environment, and docs scaffold in place so every experiment has a home.

**Learn.** Git and GitHub basics (branch, commit, push, PR), Python virtual environments, Jupyter/Colab, and how this repo is laid out.

**Steps.**
1. Create the GitHub repo and clone it. Add the scaffold files (README, docs, CONTRIBUTING, mkdocs.yml, LICENSE, .gitignore).
2. Set up a virtual environment and install Jupyter.
3. Run `mkdocs serve` and confirm the docs site loads locally.
4. Make your first commit using the Conventional Commits style, and push.

**Done when.** Your repo is on GitHub, the docs site serves locally, and you can open and run an empty notebook.

---

## Phase 1 — Notebook 01: single-image detection

**Goal.** Load a pretrained open-vocabulary detector, run it on one image, draw the boxes, and measure how long it took.

**Learn.** What object detection is; bounding-box formats (xyxy vs xywh vs center-xywh, and normalized vs absolute pixels — this distinction causes most detection bugs); confidence scores; what "zero-shot / open-vocabulary" means; loading a model from Hugging Face; non-maximum suppression (NMS) and IoU at a conceptual level.

**Steps.**
1. Load and display a single RGB image.
2. Load Grounding DINO (via Hugging Face `transformers` or the official repo).
3. Run inference with a text prompt like `person . bicycle . dog .`. Note the two thresholds Grounding DINO uses: a box threshold and a text threshold.
4. Parse the raw output into a clean list of `(class, confidence, box)`.
5. Draw the boxes and labels on the image (the `supervision` library makes this easy, or use PIL/matplotlib).
6. Measure inference latency correctly: do a warmup run first, and call `torch.cuda.synchronize()` before/after timing on GPU.
7. Save the visualization.

**Pitfall.** If your boxes land in the wrong place, you almost certainly mixed up a coordinate convention (normalized vs pixel, or xywh vs xyxy). Print the raw box values and check against the image dimensions.

**Done when.** One image + one prompt produces a saved, correctly-labeled visualization and a printed latency.

---

## Phase 2 — Notebook 02: segmentation from boxes

**Goal.** Feed the boxes from Phase 1 into SAM2 and get segmentation masks.

**Learn.** Instance vs semantic segmentation; masks as boolean/float arrays; promptable segmentation (SAM takes box or point prompts); why SAM2 is *class-agnostic* (it segments whatever you box, regardless of class — this is the insight that makes novel classes tractable); overlaying a mask on an image.

**Steps.**
1. Load SAM2.
2. Pass each Phase-1 box in as a prompt and get a mask back.
3. Overlay the masks on the image with transparency.
4. Note the extra latency segmentation adds.

**Done when.** Boxes from Phase 1 become masks overlaid cleanly on the image.

---

## Phase 3 — Notebook 03 + package: the unified annotation schema

**Goal.** Define the one data structure that holds detections and masks, and refactor your Phase 1–2 code to produce it. This is the architectural keystone — everything downstream depends on it.

**Learn.** Python dataclasses or Pydantic models; schema design; JSON serialization; the adapter/interface pattern (a common contract that different models sit behind).

**Steps.**
1. In `neuraa/schema/`, define classes for `Image`, `Detection` (class, confidence, box), `Mask`, and an `Annotation` that ties them together.
2. Add methods to serialize to JSON and load back.
3. Refactor Phase 1 into a detector adapter with the signature `detect(image, prompt) -> List[Detection]`, and Phase 2 into `segment(image, boxes) -> List[Mask]`.
4. Document the schema in `docs/annotation-schema.md`.

**Done when.** Running detection + segmentation produces a schema object you can save to JSON, reload, and re-visualize identically.

---

## Phase 4 — The eval harness and first benchmark

**Goal.** Build the ability to *measure*, then benchmark two detectors fairly. This is the "research" muscle that sets the project apart.

**Learn.** Why benchmarking needs ground truth (you cannot measure accuracy on unlabeled images); IoU-based matching; mean average precision (mAP) and precision/recall; how the COCO evaluation works; measuring latency and GPU memory properly.

**Steps.**
1. Get a labeled eval set — start with a slice of COCO val2017, which comes with ground-truth annotations.
2. Write a harness in `benchmarks/` that runs a detector adapter over the eval set and computes mAP, precision, recall, latency, and peak GPU memory.
3. Add a second detector adapter (YOLO-World) behind the same interface and run both.
4. Log the comparison table to `docs/research-log.md`.

**Pitfall.** Zero-shot models that top public leaderboards often drop sharply on domain-specific imagery. Note this when you later test on your own novel classes.

**Done when.** A single script reproduces a table of model-vs-metrics, and the numbers are in the research log.

---

## Phase 5 — Notebook 05: the export engine

**Goal.** Convert the unified annotation object into the standard training formats. This is deterministic engineering — no model involved.

**Learn.** The actual file layouts and their coordinate conventions: YOLO (one `.txt` per image, `class cx cy w h` normalized 0–1), COCO (a single JSON with `images`, `categories`, `annotations`, where `bbox` is `[x, y, w, h]` in absolute pixels), and Pascal VOC (one XML per image, `xmin ymin xmax ymax` absolute).

**Steps.**
1. In `neuraa/export/`, write one writer per format that consumes the schema.
2. Write a round-trip test: export, then re-import, and assert the boxes match within a small tolerance.

**Pitfall.** Every format uses a different box convention. The round-trip test is what catches the off-by-a-convention bugs before they poison a whole dataset.

**Done when.** The schema exports to YOLO, COCO, and Pascal VOC, and each round-trips without drift.

---

## Phase 6 — Notebook 04: the auto-labeling loop (the heart)

**Goal.** Make novel classes work. This is the hardest and most valuable phase, so it breaks into sub-steps. Use a small novel-class dataset here — something Grounding DINO struggles with zero-shot.

**Learn.** Transfer learning and fine-tuning; train/val splits and overfitting on small data; confidence calibration; ensembling and test-time augmentation; active learning (uncertainty and diversity sampling); the human-in-the-loop pattern.

**Steps.**
1. **Cold start.** Confirm zero-shot fails on your novel class (this is your router logic in miniature). Then hand-label a small seed set — 2–5% of the images.
2. **Fine-tune a specialist.** Train a small detector (Ultralytics YOLO is the easy on-ramp) on the seed. Use augmentation and a held-out validation split to fight overfitting.
3. **Auto-label the rest.** Run the specialist over all remaining images, keeping the confidence of each prediction.
4. **Build the confidence gate.** Combine three signals: the specialist's calibrated confidence, agreement with the open-vocab detector (box IoU), and stability under a flip/rescale. Route predictions to auto-accept, review, or flag.
5. **Close the loop.** Send the uncertain (mid-confidence) predictions to a human, add the corrections to the training set, and retrain. Prioritize the most informative images (uncertainty sampling).
6. **Measure convergence.** Hold out a small labeled validation slice and plot auto-accept precision against the number of human labels and the number of loop rounds.

**Done when.** You can produce a curve showing that a small number of human labels yields high-precision auto-accepted labels across the dataset — proof the loop works.

---

## Phase 7 — Notebook 07: the folder pipeline

**Goal.** Scale the single-image pipeline to a whole folder with a dataset-level export.

**Learn.** Batching and dataloaders; memory management for large image sets; progress reporting (`tqdm`); optionally multiprocessing for the I/O-bound parts.

**Steps.**
1. Wrap the detect → segment → gate → export flow in a function that walks a folder.
2. Batch inference for throughput rather than one image at a time.
3. Report progress and write a single dataset-level export at the end.

**Done when.** You point Neuraa at a folder and get back an exported, gated dataset.

---

## Phase 8 — Layer 2 (stretch): the product

**Goal.** Turn the pipeline into a portfolio-grade application. Optional, but this is what makes it a *product* rather than a research repo.

**Learn.** API design (FastAPI); a review UI (Streamlit is the fast path; React if you want to show frontend range); a database for datasets and annotations; authentication and dataset management.

**Steps (in rough order).**
1. Wrap the pipeline in a FastAPI backend.
2. Build a review UI where a human accepts/corrects the gated annotations.
3. Add a database and dataset management.
4. Deploy it somewhere public and link it from the README.

**Done when.** Someone who isn't you can upload a folder, review the auto-labels in a browser, and download a dataset.

---

## The mindset to carry through

Three habits make this project resume-worthy rather than just finished: measure everything (a number beats an impression), document as you build (the docs are half the deliverable), and keep the layers separate (research answers questions, product ships them). If you finish Phase 6, you have a genuinely differentiated project — most portfolio CV work stops at Phase 1.
