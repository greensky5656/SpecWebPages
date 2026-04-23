# Pipeline Scoring And Calibration Chain

This document describes the full repository-level data chain after per-model
docking artifacts exist.

This is a **pipeline-level** view. It explains how per-CIF docking outputs
become aggregate features, Platinum-aligned training data, trained calibration
models, and final `dg_predictions.tsv` rows.

## Scope

This document covers the higher-level chain implemented across:

- `run_all_p_runs.py`
- `scripts/build_docking_features.py`
- `scripts/parse_platinum.py`
- `scripts/build_dataset.py`
- `scripts/train_models.py`
- `scripts/predict_dg.py`
- `scripts/run_pipeline_from_cif.py`
- `affinity/`

It does not describe the literal per-CIF preparation, RMSD-guard, and docking
method. That lower-level method is documented separately.

## Stage 1: Build The Batch Docking Summary

The docking layer scans supported run folders and writes one row per CIF model.

What happens:

- `run_all_p_runs.py` discovers top-level `p0*` and `p1*` folders
- `scripts/run_cif_batch.py` wraps that driver for the maintained CLI surface
- each completed model contributes a row to `vina_batch_summary.tsv`

Pipeline meaning:

- this is the first unified tabular contract after per-model docking

## Stage 2: Build Per-CIF Docking Features

The feature layer enriches each docking row with canonical identifiers and
descriptors.

What happens:

- `scripts/build_docking_features.py` loads `vina_batch_summary.tsv`
- `affinity/parse_docking.py` normalizes `target_key`
- the parser extracts model index from CIF naming
- the parser loads `ligand.sdf` and computes RDKit ligand descriptors

Pipeline meaning:

- this yields `docking_per_cif_features.tsv`, which still contains one row per
  CIF

## Stage 3: Aggregate By `(target_key, ligand_key)`

The aggregation layer collapses per-CIF rows into one record per modeling key.

What happens:

- `affinity/dataset_build.py` groups rows by `(target_key, ligand_key)`
- the code computes `n_models`
- the code summarizes Vina affinity values with min, mean, median, and standard
  deviation
- the ligand descriptor block is copied forward for modeling

Pipeline meaning:

- this yields `docking_agg_features.tsv`, the main model-input contract

## Stage 4: Parse Experimental Platinum Data

The experimental-reference layer normalizes `platinumresults.md`.

What happens:

- `scripts/parse_platinum.py` calls `affinity/parse_platinum.py`
- the parser reads tab-separated rows
- the parser expects ligand identifiers and a final
  `dg_exp_kcal_per_mol` column

Pipeline meaning:

- this yields `platinum_parsed.tsv`, the normalized experimental dataset

## Stage 5: Join Docking To Platinum

The join layer combines aggregate docking rows with experimental reference data.

What happens:

- `scripts/build_dataset.py` calls `affinity/join_dataset.py`
- rows are joined on `(target_key, ligand_key)`
- duplicate Platinum rows are summarized by mean experimental `ΔG`, population
  standard deviation, and contributing row count

Pipeline meaning:

- this yields `training_dataset.tsv`, the direct training input for calibration
  models

## Stage 6: Train Calibration Models

The training layer fits ridge regression models over the joined dataset.

What happens:

- `scripts/train_models.py` calls `affinity/train_calibrator.py`
- supported modes are `global` and `per-target`
- the training output includes `manifest.json`, `model.joblib`, and `meta.json`

Pipeline meaning:

- this persists the reusable calibration models and their training metadata

## Stage 7: Predict Calibrated `ΔG`

The prediction layer applies the saved model set to aggregated docking features.

What happens:

- `scripts/predict_dg.py` calls `affinity/predict.py`
- the script loads `manifest.json`
- the script applies the matching model(s) to `docking_agg_features.tsv`

Pipeline meaning:

- this yields `dg_predictions.tsv`, where `dg_pred_kcal_per_mol` is prepended to
  the original aggregate feature row for provenance

## Core Pipeline Equation

At the repository workflow level, the current data chain is:

```text
per-CIF docking rows
plus
descriptor extraction
= per-CIF feature rows

per-CIF feature rows
grouped by (target_key, ligand_key)
= aggregate docking feature rows

aggregate docking feature rows
joined with Platinum rows
= training dataset rows

training dataset rows
fit ridge models
= reusable calibration models

aggregate docking feature rows
plus trained models
= final dg_predictions rows
```

## Optional ABFE Branch

The repository also contains:

- `scripts/run_abfe.py`
- `scripts/merge_abfe.py`
- `affinity/abfe/`

These files support a pilot path where `dg_abfe_kcal_per_mol` can be merged into
the training dataset and used as an extra feature via `--use-abfe-feature`.

## Where This Lives In Code

- `run_all_p_runs.py`
- `affinity/parse_docking.py`
- `affinity/dataset_build.py`
- `affinity/parse_platinum.py`
- `affinity/join_dataset.py`
- `affinity/train_calibrator.py`
- `affinity/predict.py`
- `scripts/run_pipeline_from_cif.py`

## Practical Summary

If someone asks, "what is the scoring and calibration chain for the whole
repository?", the current answer is:

1. build the batch docking summary
2. build per-CIF docking features
3. aggregate by `(target_key, ligand_key)`
4. parse experimental Platinum data
5. join docking to Platinum
6. train calibration models
7. predict calibrated `ΔG`

## Related Documents

- `docs/per_model_docking_and_minimization_method.md`
- `docs/runtime_artifacts_reference.md`
- `docs/design_specification.md`
- `docs/pipeline_flowchart.md`
