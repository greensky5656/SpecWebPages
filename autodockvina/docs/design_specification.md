# Design Specification

This document describes the current implemented design of the AutoDock Vina RMSD
pipeline as reflected in the codebase today.

## Purpose

The repository implements a strict Boltz-to-docking and affinity-analysis
workflow. Given top-level Boltz run folders, a single run folder, or a Boltz
results root, it resolves ligand identity, prepares receptor and ligand docking
artifacts, runs score-only or docking calculations, aggregates features, joins
experimental Platinum data, trains ridge calibration models, and emits
calibrated `dg_predictions.tsv` outputs.

## Supported Scope

The current implementation supports:

- top-level batch discovery through `run_all_p_runs.py` and
  `scripts/run_cif_batch.py`
- single-run direct scoring through `scripts/score_boltz_run.py`
- single-run minimized scoring through `scripts/score_boltz_run_minimized.py`
- ligand routing through CCD-native or Boltz-YAML SMILES-native definitions
- optional retained cofactor awareness in the minimized batch path
- deterministic preflight classification into `run`, `skip`, or runtime `error`
- `global` and `per-target` ridge calibration
- optional ABFE feature merge before model training

The current implementation does not support:

- permissive recovery when ligand identity or chemistry reconstruction fails
- automatic repair of malformed run naming or inconsistent ligand routing inputs
- automatic schema validation of emitted artifacts
- a packaged `pip install -e .` Python package install surface

Unsupported or invalid cases fail fast rather than following an alternate
execution path.

## Inputs

The maintained entrypoints accept one of these input families:

- top-level repository folders named `p0*` or `p1*`
- a single Boltz run folder
- a `run-*-cifs/` subfolder under a run folder
- a Boltz results root, optionally paired with `--boltz-yaml`

At the model level, the docking path expects Boltz `*.cif` structures and then
derives:

- protein atoms and chain content
- non-polymer component identifiers
- the selected target ligand
- optional retained cofactor identifiers
- receptor and ligand artifacts for Vina preparation

## Top-Level Architecture

Main implemented modules and entrypoints:

- `meeko_cif_to_vina.py`: primary per-CIF docking executor
- `run_all_p_runs.py`: low-level batch driver for top-level `p0*` and `p1*`
  folders
- `run_all_cifs.py`: lower-level per-run CIF driver
- `scripts/score_boltz_run.py`: single-run direct scorer
- `scripts/score_boltz_run_minimized.py`: single-run minimized scorer with
  preflight manifests and failure summaries
- `scripts/build_docking_features.py`: per-CIF and aggregate feature generation
- `scripts/parse_platinum.py`: Platinum markdown parser
- `scripts/build_dataset.py`: docking-to-Platinum join step
- `scripts/train_models.py`: ridge calibration training
- `scripts/predict_dg.py`: calibrated prediction emission
- `scripts/run_pipeline_from_cif.py`: end-to-end wrapper over the maintained
  calibration chain
- `affinity/`: library modules for feature parsing, aggregation, joining,
  training, prediction, and ABFE merge helpers

## Execution Flow

For the maintained docking-to-calibration chain, the implemented flow is:

1. Discover run folders, CIFs, or a Boltz results root.
2. Resolve ligand identity from direct arguments, Boltz YAML CCD entries, or
   Boltz YAML SMILES definitions.
3. Parse each CIF and write receptor or ligand preparation artifacts.
4. Optionally run Option A protein-only preparation and restrained OpenMM
   minimization.
5. Run Vina score-only on the retained Boltz pose when requested and run Vina
   docking or redocking.
6. Write per-CIF artifacts and per-run or per-batch summaries.
7. Parse docking outputs into `docking_per_cif_features.tsv`.
8. Aggregate features by `(target_key, ligand_key)` into
   `docking_agg_features.tsv`.
9. Parse `platinumresults.md` into `platinum_parsed.tsv`.
10. Join aggregated docking rows to Platinum rows into `training_dataset.tsv`.
11. Train ridge models and write `models/manifest.json` plus per-model outputs.
12. Apply the saved model(s) to produce `dg_predictions.tsv`.

## Execution Styles

### Standard Docking

The standard docking path runs `meeko_cif_to_vina.py` directly from
`run_all_p_runs.py`, `scripts/run_cif_batch.py`, or `scripts/score_boltz_run.py`
without the Option A minimization stage.

Typical outputs land under `vina_batch_test/` or `vina_boltz_run_scores/`.

### Minimized Option A

The minimized path is exposed by `scripts/score_boltz_run_minimized.py`. It
adds:

- input normalization for legacy run folders and Boltz results roots
- deterministic preflight ligand routing decisions per CIF
- optional cofactor-aware behavior from Boltz YAML ligand rosters
- `pdb4amber`, `cpptraj`, `tleap`, and restrained OpenMM minimization
- run logging, batch manifests, failure classification, and summary JSON

Typical outputs land under `vina_boltz_minimized/<run>/` or a Boltz results
root's `vina_analysis/<run>/`.

### End-To-End Calibration

The end-to-end calibration path is exposed by `scripts/run_pipeline_from_cif.py`
and chains batch docking, feature generation, Platinum parsing, dataset join,
model training, and prediction emission.

## Ligand And Routing Validation Rules

The implemented docking path enforces:

- one selected ligand per docking record
- strict ligand reconstruction from CCD chemistry or Boltz-YAML SMILES
- pose-guard checks between CIF, RDKit, and PDBQT representations
- optional retained cofactor accounting only when Boltz YAML reports it
- explicit failure when the selected ligand route cannot be reconstructed

Canonical modeling keys are:

- `target_key`
- `ligand_key`

`target_key` is inferred from the leading run token such as
`p10721... -> P10721`. `ligand_key` is the selected ligand CCD `comp_id`.

## Failure And Outcome Model

The implementation is intentionally strict:

- minimized preflight can classify a CIF as `skip` before any docking is run
- missing tools, malformed inputs, failed chemistry reconstruction, and parsing
  violations stop the active command
- model training refuses insufficient row counts
- prediction refuses missing manifests, missing required feature columns, or
  unavailable matching models
- no backup execution route is introduced when required criteria are not met

## State, Provenance, And Output Contract

The implementation writes four broad artifact classes:

- per-CIF preparation and docking outputs
- per-run or per-batch summaries
- feature, dataset, and prediction TSVs
- model directories containing `model.joblib`, `meta.json`, and
  `manifest.json`

Maintained machine-readable contracts currently exist for:

- `specs/schemas/vina_batch_summary_row.schema.json`
- `specs/schemas/run_affinity_summary.schema.json`
- `specs/schemas/dg_predictions_row.schema.json`
- `specs/schemas/model_manifest.schema.json`

These schemas document the row or object shapes consumed by downstream
analysis, even though the current code does not yet auto-validate emitted files
against them.

## Operational Constraints

Current code-level constraints worth knowing:

- Linux runs should override `--vina-bin` explicitly because several scripts
  still default to checked-in macOS Vina paths
- descriptor calculation requires RDKit and a sanitizable `ligand.sdf`
- `scripts/train_models.py --mode global` requires at least two rows total
- `scripts/train_models.py --mode per-target` requires at least two rows per
  `target_key`
- the maintained workflow assumes execution from the repository root

## Source-Of-Truth References

- Per-CIF docking executor: `meeko_cif_to_vina.py`
- Top-level batch driver: `run_all_p_runs.py`
- Minimized runner: `scripts/score_boltz_run_minimized.py`
- Direct single-run scorer: `scripts/score_boltz_run.py`
- Feature parser: `affinity/parse_docking.py`
- Dataset aggregation: `affinity/dataset_build.py`
- Platinum parser: `affinity/parse_platinum.py`
- Dataset join: `affinity/join_dataset.py`
- Model training: `affinity/train_calibrator.py`
- Prediction: `affinity/predict.py`
