# autodockvina Specification

Updated: 2026-04-20 21:13:35 PDT

Codebase-grounded scientific review of the current autodockvina execution path: Boltz-CIF parsing, Meeko/Vina scoring, feature engineering, calibration-model training, prediction, and optional ABFE pilot surfaces.

Repository: `/home/joethomas/Documents/Github/autodockVinaRMSD/autodockvina`

Authority policy: this page is grounded primarily in current source code, tests, and shipped output-contract logic. README/docs were treated as secondary context only.

## Executive view for signoff

<p>autodockvina is not just a single docking wrapper. The actual current codebase contains a layered pipeline: Boltz-CIF parsing, Meeko/Vina scoring, aggregated docking features, Platinum joins, Ridge-regression calibration, and optional ABFE pilot hooks. Signoff requires keeping these layers separate instead of treating every downstream number as one undifferentiated affinity claim.</p>

## Actual shipped operator surface

- `score_boltz_run_minimized.py`: primary modern run-folder driver
- `meeko_cif_to_vina.py`: per-CIF parsing/prep/Vina/artifact path
- `affinity/*.py`: learned calibration and prediction surfaces
- `affinity/abfe/*`: optional pilot ABFE layer

## Scientific workflow actually implemented

1. Parse Boltz mmCIF and identify protein/ligand entities
2. Require and consume Boltz YAML in practice
3. Prepare and optionally minimize receptor structure
4. Reconstruct ligand chemistry via SMILES or CCD route
5. Prepare receptor/ligand PDBQT through Meeko
6. Define docking box from ligand coordinate envelope
7. Run Vina score-only and docking modes
8. Aggregate outputs into manifests/results/summaries
9. Optionally engineer features, join to Platinum, train Ridge model(s), and predict calibrated ΔG

## What scientific reviewers should separate

- Raw Vina score-only / docking score: direct AutoDock Vina outputs
- Docking-derived feature tables: engineered summaries plus descriptors
- Calibrated ΔG prediction: statistical model layer
- ABFE pilot: optional pilot physics layer, not default authority

## Current scientific and operational limits

- Mainline path depends on YAML in practice; some wrappers are stale
- Current learned calibration evidence is weak
- Feature set is narrow
- ABFE is pilot-level, not default production endpoint

## Recommended signoff questions

### For genetics leadership
- Is a docking-plus-calibration evidence layer acceptable for the intended use case?
- Do you want raw docking and learned calibration clearly separated in every report?

### For thermodynamics review
- Should calibrated ΔG be treated only as a statistical correction layer?
- Should ABFE remain pilot-only until stronger evidence exists?

## Evidence appendix

- `/home/joethomas/Documents/Github/autodockVinaRMSD/autodockvina/meeko_cif_to_vina.py`
- `/home/joethomas/Documents/Github/autodockVinaRMSD/autodockvina/scripts/score_boltz_run_minimized.py`
- `/home/joethomas/Documents/Github/autodockVinaRMSD/autodockvina/scripts/score_boltz_run.py`
- `/home/joethomas/Documents/Github/autodockVinaRMSD/autodockvina/affinity/parse_docking.py`
- `/home/joethomas/Documents/Github/autodockVinaRMSD/autodockvina/affinity/dataset_build.py`
- `/home/joethomas/Documents/Github/autodockVinaRMSD/autodockvina/affinity/join_dataset.py`
- `/home/joethomas/Documents/Github/autodockVinaRMSD/autodockvina/affinity/train_calibrator.py`
- `/home/joethomas/Documents/Github/autodockVinaRMSD/autodockvina/affinity/predict.py`
- `/home/joethomas/Documents/Github/autodockVinaRMSD/autodockvina/affinity/abfe/abfe_pilot.py`
- `/home/joethomas/Documents/Github/autodockVinaRMSD/autodockvina/affinity/abfe/decouple.py`

## Top current clarifications

- The real mainline is the minimized/cofactor-aware Boltz-CIF to Meeko/Vina path, not just a thin docking wrapper.
- The repo contains a distinct learned calibration surface on top of docking outputs, and that layer must be reviewed separately from raw Vina scores.
- The currently shipped trained model artifact is extremely small and should not be mistaken for production-grade calibration evidence.