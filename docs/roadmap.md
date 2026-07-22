# Roadmap

Neuraa is developed in two layers. Layer 1 (research) must be sufficiently mature before Layer 2 (product) begins. This page tracks progress and defines the research question each notebook answers.

Status legend: ⚪ not started · 🟡 in progress · 🟢 done

---

## Layer 1 — Research

The goal of Layer 1 is to understand how each foundation model behaves and to prove the auto-labeling loop converges. Every experiment produces measurable results. Nothing here concerns UI or deployment.

A guiding discipline: **work with a single image before scaling to thousands.** Once the single-image pipeline is stable, scaling to folders is an engineering task rather than a research problem.

### Research notebooks

Each notebook answers one specific research question and produces an entry in the [research log](research-log.md).

| # | Notebook | Research question | Status |
| --- | --- | --- | --- |
| 01 | Grounding DINO detection | Can Grounding DINO accurately detect user-defined objects in a single image using zero-shot prompting? | 🟡 |
| 02 | SAM2 segmentation | Given a bounding box, can SAM2 produce an accurate mask for arbitrary (including novel) classes? | ⚪ |
| 03 | Unified annotation object | Can detections, masks, and OCR from different models be normalized into one schema? | ⚪ |
| 04 | Seed + active-learning loop | How few seed labels does the loop need before auto-accept precision crosses target, and how many rounds to converge? | ⚪ |
| 05 | Confidence gate | How reliable is the usefulness gate — what is the precision of auto-accepted labels vs. ground truth? | ⚪ |
| 06 | Export engine | Can the unified annotation object be exported losslessly to YOLO, COCO, and Pascal VOC? | ⚪ |
| 07 | Folder pipeline | Does the single-image pipeline scale to a full folder with a dataset-level export? | ⚪ |

### Notebook 01 — Grounding DINO detection

**Question:** Can Grounding DINO accurately detect user-defined objects in a single image using zero-shot prompting?

**Input:** one RGB image + a text prompt (e.g. `person . bicycle . dog .`)

**Output:** for every detected object — class, confidence, bounding box, plus a saved visualization.

**Success criteria.** The notebook is successful if it can load a Grounding DINO model, run inference on a single image, detect arbitrary user-specified classes, display bounding boxes, report inference latency, and save the visualization. Nothing more.

**Why one image first?** Working with a single image lets us debug model behavior, understand preprocessing, measure inference speed, validate outputs, and experiment with thresholds before scaling.

### Cross-cutting research foundations

These are set up alongside Notebook 01 and reused by every later notebook:

- **Eval harness.** A shared set of test images, metrics (mAP/precision/recall, latency, GPU footprint), and logging, so every model is compared apples-to-apples. Benchmarking needs ground truth — decide the eval set early (COCO val, plus a small hand-labeled slice of the target domain).
- **Annotation schema.** Define the unified annotation object up front, even if only detection fields are populated at first, so later notebooks read and write it instead of ad-hoc dictionaries.
- **Adapter interface.** Sketch the `detect(image, prompt) -> List[Detection]` contract in Notebook 01 so subsequent models slot in behind it.

---

## Layer 2 — Product

Begun only after Layer 1 is sufficiently mature. Planned components:

- Annotation pipeline
- Backend
- Frontend
- Database
- Authentication
- Dataset management
- Team collaboration
- Active learning
- Cloud deployment

---

## Guiding principles

Every component should be modular, model-agnostic, reproducible, benchmark-driven, and scalable. See [architecture.md](architecture.md) for how these are realized.
