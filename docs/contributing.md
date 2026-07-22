# Contributing to Neuraa

Thanks for your interest in Neuraa. This guide covers how to set up a development environment and the conventions that keep the project clean and reviewable.

## Development setup

```bash
git clone https://github.com/your-username/neuraa.git
cd neuraa
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
pip install -r requirements-dev.txt   # linting, tests, pre-commit
pre-commit install
```

## Project layers

Neuraa has two layers, and contributions should respect the separation:

- **Layer 1 (research)** lives in `notebooks/` and `benchmarks/`. This is where new models are evaluated.
- **Layer 2 (product)** lives in `neuraa/`. Code moves here only after Layer 1 has validated it.

See the [roadmap](docs/roadmap.md) for what stage the project is in.

## Notebook conventions (Layer 1)

Research notebooks are the heart of the current phase, so they follow a strict format:

- **One question per notebook.** The first markdown cell states the research question, the input, the expected output, and the success criteria.
- **Every experiment produces a measurable result** — latency, accuracy, GPU footprint. No "it seems to work" conclusions.
- **Pin versions and seeds.** Record model versions and random seeds at the top so results are reproducible.
- **Write up results.** Each notebook has a companion page in `docs/notebooks/` and an entry in `docs/research-log.md` with the numbers.
- **Clear outputs before committing.** Notebook output cells bloat diffs; a pre-commit hook strips them.

## Documentation is part of the change

Neuraa uses **docs-as-code**: documentation lives in the repo and is reviewed in the same pull request as the code it describes. A change that alters behavior without updating the relevant doc is incomplete. Docs are published with MkDocs; preview locally with `mkdocs serve`.

## Commit conventions

We use [Conventional Commits](https://www.conventionalcommits.org/). This keeps the history readable and lets the changelog be generated automatically.

```
feat: add YOLO-World detection adapter
fix: correct COCO bbox coordinate order in export
docs: document the confidence gate signals
research: benchmark SAM2 mask latency on 512px images
chore: bump grounding-dino pin to 1.5
```

Common types: `feat`, `fix`, `docs`, `research`, `test`, `refactor`, `chore`.

## Pull-request process

1. Branch from `main` using a descriptive name (`feat/yolo-world-adapter`).
2. Keep PRs focused — one logical change per PR.
3. Ensure `pre-commit`, tests, and (for Layer 2) type checks pass locally.
4. Fill out the PR template, linking any related issue.
5. Update the relevant docs and, for research, the research log.
6. A maintainer reviews and merges.

## Reporting issues

Use the issue templates in `.github/ISSUE_TEMPLATE/`. For research questions or model requests, include the model, the task, and — where possible — a link to the paper or repository.

## Code of conduct

Be respectful and constructive. This is a public, welcoming project.
