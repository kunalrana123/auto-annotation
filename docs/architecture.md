# Architecture

This document describes how Neuraa auto-annotates a folder of images with minimal human effort, while maintaining a measurable quality bar. It is the reference for the system's design decisions.

## Design goals

Neuraa is built to be:

- **Modular** — every component can be replaced or extended.
- **Model-agnostic** — no dependence on any single foundation model.
- **Reproducible** — experiments pin model versions and seeds; we target *statistical* reproducibility, since bit-exact output is unrealistic on GPUs.
- **Benchmark-driven** — every model is evaluated with measurable metrics before adoption.
- **Scalable** — prototypes evolve into production systems.

## The two layers

Neuraa is split into two independent layers so that research questions never get tangled with product plumbing.

- **Layer 1 — Research.** Evaluate and benchmark state-of-the-art vision models, and validate that the auto-labeling loop converges. Nothing here concerns UI or deployment.
- **Layer 2 — Product.** After Layer 1 identifies the best-performing components, build the annotation pipeline, backend, frontend, database, and deployment.

We are currently in Layer 1.

## The core insight: general vs. novel classes

Not all classes are equally hard, and the architecture branches on this.

- **General classes** (`car`, `person`, `dog`) are solved today: an open-vocabulary detector such as Grounding DINO or YOLO-World localizes them zero-shot, and SAM2 produces masks.
- **Novel / fine-grained classes** (`VTOL`, a specific defect, a rare species) are where zero-shot fails. The detector either misses the object, fires on a generic superclass, or confuses it with a neighbor.

A key simplification follows: **the novel-class problem is a localization problem, not a segmentation one.** Once a box exists around an object, SAM2 masks it perfectly regardless of how exotic the class is — SAM2 is class-agnostic. So the entire difficulty lives in *detection*, and segmentation comes for free downstream.

## The router

Before any labeling, Neuraa decides which regime a class belongs to — automatically, because a user cannot reliably tell you whether their class is "hard."

The router runs the open-vocabulary detector zero-shot on a random sample (~100 images) with the user's class prompt and inspects the resulting confidence distribution:

- Consistent, high-confidence detections → treat as **general**; auto-label the folder and gate the results.
- Sparse, low-confidence, or scattered detections → route into the **novel-class loop** below.

## The auto-labeling loop (novel classes)

For a novel class, Neuraa runs an active-learning bootstrap loop:

1. **Seed labels.** The user labels a small seed set (~2–5% of the dataset). For a cold start with zero prior, the cheapest viable seed is visual — the user boxes a handful of instances, and image-conditioned detection produces rough first labels that the user then corrects.
2. **Fine-tune a specialist.** A lightweight detector is fine-tuned on the seed. This turns a novel class into a "known" class for a small, fast model.
3. **Auto-label the full dataset.** The specialist predicts boxes across all images, each with a confidence score.
4. **Gate by usefulness** (see below). Predictions are routed to auto-accept, human review, or flag.
5. **Feed corrections back.** Human review corrections are added to the training set, and the specialist is retrained. Each round raises the auto-accept rate and lowers human load.

The loop stops when the auto-accept rate plateaus and validation quality holds above target.

### The honest constraint

For a genuinely novel class, **zero human input is impossible** — the system has no prior for what the object looks like. The achievable promise is not "fully automatic" but "label ~2–5% of the dataset, and the system does the other 95–98% at a measured quality bar." For 10,000 images that is roughly 200–400 human-labeled images producing 10,000 good annotations. Being explicit about this preserves user trust.

## The usefulness threshold

"Usefulness" is the confidence gate that decides which auto-labels are trustworthy. A single model's raw confidence is poorly calibrated, so the gate combines three cheap signals:

- **Calibrated model confidence** from the specialist.
- **Cross-model agreement** — does the open-vocabulary detector's box overlap the specialist's box at high IoU?
- **Test-time-augmentation stability** — does the detection survive a flip and a rescale?

The threshold itself is set empirically. A small human-labeled validation slice is held out, and the confidence cutoff is chosen where auto-accepted labels hit the user's precision target (for example, 95%). That same slice provides the **stopping criterion** — the dataset is done when auto-accept precision holds and the accept rate plateaus, so you never label the whole set to know when to stop.

## The model-adapter interface

"Model-agnostic" is only real if it is an actual interface. The supported models have incompatible I/O — Grounding DINO wants dot-separated text prompts, YOLO-World wants a class list, Florence-2 wants task tokens, and the vision-language models return unstructured text to be parsed.

Neuraa normalizes all of this behind a single contract:

```
detect(image, prompt) -> List[Detection]
```

Every detector adapter accepts an image and a prompt and returns a normalized list of detections. Segmentation adapters follow the same pattern: `segment(image, boxes) -> List[Mask]`. This adapter contract is the seam that keeps the rest of the system independent of any specific model.

## The unified annotation object

All components read and write one schema — the unified annotation object. A detection from Grounding DINO, a mask from SAM2, and OCR text from PaddleOCR all land in the same structure, so visualization, export, and batch processing depend on the schema rather than on any model. The schema is defined and versioned in [annotation-schema.md](annotation-schema.md).

Note on annotation guidelines: subjective conventions (do occluded instances count? one box per object or per part?) are captured *implicitly* by how the user labels the seed set. The specialist learns whatever convention the seed demonstrates, so user preference is enforced automatically downstream rather than through a settings panel.

## Export

Export is a deterministic transform from the unified annotation object to each target format (YOLO, COCO, Pascal VOC, LabelMe, CVAT, custom JSON). It is engineering, not research — there is no unknown to benchmark.
