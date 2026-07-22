<div align="center">

# Neuraa

**An AI-powered computer vision annotation engine that auto-labels image datasets — including the classes off-the-shelf models have never seen.**

[![License: MIT](https://img.shields.io/badge/License-MIT-informational.svg)](LICENSE)
[![Python 3.10+](https://img.shields.io/badge/python-3.10%2B-blue.svg)](https://www.python.org/)
[![Docs](https://img.shields.io/badge/docs-mkdocs-teal.svg)](https://your-username.github.io/neuraa/)
[![Status](https://img.shields.io/badge/status-Phase%201%20research-orange.svg)](docs/roadmap.md)

</div>

---

## The problem

Manual annotation is the bottleneck in every computer-vision project. Drawing bounding boxes and segmentation masks by hand across thousands of images is slow, expensive, and error-prone.

Modern vision foundation models can annotate common objects (`car`, `person`, `dog`) zero-shot. But real datasets are full of the *long tail* — a folder of 10,000 images of **VTOL aircraft**, or a specific manufacturing defect, or a rare species. No pretrained model knows those classes, and that's exactly where annotation effort is most painful.

## What Neuraa does

Neuraa turns annotation from *creation* into *verification*. You point it at a folder of images and describe the classes you want. For common classes, it labels them automatically. For novel classes, you label a tiny seed (~2–5% of the dataset) and Neuraa bootstraps a specialist model that labels the rest — routing only the uncertain cases back to you, with a measured quality guarantee.

The result: **label a few hundred images, get a few thousand back**, exported in whatever format your training pipeline expects.

```
Images  →  AI detection  →  AI segmentation  →  Human review  →  Automatic export
```

## How it works

The core is an active-learning bootstrap loop. Rather than expecting a zero-shot model to magically understand a novel class, a small human seed teaches a lightweight specialist, which labels the full dataset. A confidence gate decides what is trustworthy enough to keep, and human corrections flow back to sharpen the specialist each round.

The full design — the general-vs-novel router, the confidence gate, the model-adapter interface, and the annotation schema — is documented in **[docs/architecture.md](docs/architecture.md)**.

## Project status

Neuraa is built in two deliberately separate layers:

| Layer | Purpose | Status |
| --- | --- | --- |
| **Layer 1 — Research** | Benchmark vision foundation models and validate the auto-labeling loop | 🟢 In progress |
| **Layer 2 — Product** | Annotation pipeline, backend, frontend, dataset management | ⚪ Not started |

We are currently in **Phase 1: Model Research**. The goal is not to build the product yet — it is to understand how each foundation model behaves and to prove the auto-labeling loop converges. See the **[roadmap](docs/roadmap.md)** for the notebook-by-notebook research plan.

## Supported models (planned)

Neuraa is model-agnostic by design. Every model sits behind a common adapter interface, so detectors and segmenters are interchangeable.

- **Detection:** Grounding DINO, YOLO-World, Florence-2
- **Segmentation:** SAM2, EfficientSAM
- **OCR:** PaddleOCR
- **Vision-language:** Qwen2.5-VL, Molmo

## Export formats (planned)

YOLO · COCO · Pascal VOC · LabelMe · CVAT · custom JSON

## Repository structure

```
neuraa/
├── README.md               # You are here
├── LICENSE
├── CONTRIBUTING.md
├── CHANGELOG.md
├── pyproject.toml
├── mkdocs.yml              # Documentation site config
├── docs/                   # Documentation (published to GitHub Pages)
│   ├── index.md
│   ├── architecture.md
│   ├── roadmap.md
│   ├── annotation-schema.md
│   ├── research-log.md     # Running benchmark results
│   └── notebooks/          # One write-up per research notebook
├── notebooks/              # Layer 1 — research experiments
│   └── 01_grounding_dino_detection.ipynb
├── benchmarks/             # Eval harness + result artifacts
├── neuraa/                 # Layer 2 — the Python package
│   ├── adapters/           # Model adapters (detection, segmentation, OCR)
│   ├── schema/             # Unified annotation object
│   ├── export/             # Export-format writers
│   └── pipeline/           # Auto-labeling loop
├── tests/
└── .github/
    ├── workflows/          # CI
    ├── ISSUE_TEMPLATE/
    └── PULL_REQUEST_TEMPLATE.md
```

## Getting started

> ⚠️ Neuraa is in early research. The package API does not exist yet — the current work lives in `notebooks/`.

```bash
git clone https://github.com/your-username/neuraa.git
cd neuraa
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
jupyter lab notebooks/
```

## Documentation

Full documentation is published at **[your-username.github.io/neuraa](https://your-username.github.io/neuraa/)**. Key pages:

- [Architecture](docs/architecture.md) — how the auto-labeling loop works
- [Roadmap](docs/roadmap.md) — research plan and progress
- [Annotation schema](docs/annotation-schema.md) — the unified annotation format
- [Research log](docs/research-log.md) — benchmark results as they land

## Contributing

Contributions are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md) for development setup, notebook conventions, and the pull-request process.

## License

Released under the [MIT License](LICENSE).
