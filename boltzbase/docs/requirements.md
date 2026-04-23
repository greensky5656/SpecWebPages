# Requirements

This document records the current implemented requirements for the BoltzBase
platform in a stricter, review-friendly format. It is intended to summarize the
present combined CLI + UI + queue platform rather than detached or future-facing
evaluation ambitions.

## Intended use

BoltzBase is intended to support native Boltz structure prediction and optional
Boltz-2 affinity outputs through both the installed `boltz` CLI and the
Affinitas web application, with queue-backed submission, run inspection,
downloads, and optional post-processing helpers.

## System requirements

- A Python environment compatible with the root package metadata
  (`>=3.10,<3.13`), with Python 3.12 currently required by the maintained UI
  bootstrap path.
- The Python dependencies declared in `pyproject.toml`, including the Boltz
  runtime stack, PyTorch, Hydra, RDKit, Gemmi, and pandas.
- Optional extras when needed for testing, linting, or CUDA acceleration.
- Node.js 18+ and npm for the Next.js Affinitas UI.
- Writable local paths for caches, result roots, queue state, logs, and the
  SQLite-backed run metadata used by the UI.

## Input requirements

- Direct CLI runs require a YAML, FASTA, or directory input accepted by
  `boltz predict`.
- Affinity outputs require input manifests that explicitly request
  `properties.affinity`.
- UI-submitted runs must satisfy the web layer's path-safety and payload-size
  checks.
- The current implementation expects cache and output directories that resolve
  correctly under the configured environment.

## Functional requirements

- The platform must support direct CLI prediction through `boltz predict`.
- It must support web-based submission, queue-backed execution, run inspection,
  and downloadable artifacts through the Affinitas UI.
- It must create a `boltz_results_<input_stem>` run root beneath the selected
  output parent.
- It must support Boltz-1 and Boltz-2 structure prediction, with Boltz-2
  affinity outputs only when affinity-enabled inputs are requested.
- It may optionally emit RMSD, TM-score, Vina docking, and CSV-export surfaces
  through the current post-processing helpers.

## Output requirements

- CLI result trees rooted under the selected `--out_dir` parent and named as
  `boltz_results_<input_stem>`.
- Web-facing run records, logs, queue metadata, and downloadable artifacts for
  UI-submitted jobs.
- Native structure and optional affinity outputs under the standard Boltz output
  layout, with additional optional post-processing outputs when those helpers
  are invoked.

## Safety and fail-closed requirements

- The primary shipped operator surface remains `boltz predict`; evaluation and
  benchmark scripts remain separate from the installed product CLI.
- The UI queue currently exposes only the four primary run states `queued`,
  `running`, `completed`, and `failed`.
- Users must distinguish the native product surface from detached or non-
  productized evaluation code and reports in the wider repository.

## Non-requirements / out-of-scope behavior

- The current installed package does not require a first-class evaluation CLI as
  part of the product surface.
- The platform does not require a claim of calibrated thermodynamic authority
  for all affinity-like outputs; native structure + affinity predictions and
  evaluation artifacts remain separate evidence classes.
- Optional downstream RMSD/Vina helpers are not required for every successful
  CLI or web-submitted prediction.
