# Runtime And Artifacts Reference

This document summarizes the current runtime outputs written by the repository
and what they represent.

## Output Directory Structure

The maintained workflows write artifacts into two main families:

- batch docking and calibration outputs under `vina_batch_test/`
- minimized single-run outputs under `vina_boltz_minimized/<run>/` or
  `<boltz_results_root>/vina_analysis/<run>/`

Important subdirectories and files include:

- `vina_batch_test/vina_batch_summary.tsv`
- `vina_batch_test/docking_per_cif_features.tsv`
- `vina_batch_test/docking_agg_features.tsv`
- `vina_batch_test/platinum_parsed.tsv`
- `vina_batch_test/training_dataset.tsv`
- `vina_batch_test/models/manifest.json`
- `vina_batch_test/dg_predictions.tsv`
- `vina_boltz_minimized/<run>/per_cif_affinities.tsv`
- `vina_boltz_minimized/<run>/run_affinity_summary.tsv`
- `vina_boltz_minimized/<run>/run_affinity_summary.json`
- `vina_boltz_minimized/<run>/batch_manifest.tsv`
- `vina_boltz_minimized/<run>/batch_results.tsv`
- `vina_boltz_minimized/<run>/batch_failures.tsv`
- `vina_boltz_minimized/<run>/batch_summary.json`

## Batch Docking And Calibration Artifacts

### `vina_batch_summary.tsv`

Produced by `run_all_p_runs.py` and `scripts/run_cif_batch.py`.

Columns:

- `boltz_run`
- `cif`
- `ligand_comp_id`
- `affinity_kcal_per_mol`
- `out_dir`
- `ligand_pdbqt`
- `receptor_pdbqt`
- `config`
- `docked_pdbqt`
- `log`

The row shape is documented by
`specs/schemas/vina_batch_summary_row.schema.json`.

### `docking_per_cif_features.tsv`

Produced by `scripts/build_docking_features.py`.

It expands each batch docking row with:

- canonical target identifiers
- parsed model index
- the generated `ligand.sdf` path
- RDKit ligand descriptors such as `ligand_mw`, `ligand_logp`, and
  `ligand_fraction_csp3`

### `docking_agg_features.tsv`

Produced by `scripts/build_docking_features.py`.

Each row represents one `(target_key, ligand_key)` pair and includes:

- `n_models`
- `vina_affinity_min`
- `vina_affinity_mean`
- `vina_affinity_median`
- `vina_affinity_std`
- the ligand descriptor set copied forward for modeling

### `platinum_parsed.tsv`

Produced by `scripts/parse_platinum.py`.

Columns include:

- `target_key`
- `uniprot`
- `pdb_id`
- `ligand_key`
- `ligand_name`
- `assay`
- `dg_exp_kcal_per_mol`

### `training_dataset.tsv`

Produced by `scripts/build_dataset.py`.

It begins with all aggregated docking columns and appends:

- `dg_exp_kcal_per_mol`
- `dg_exp_kcal_per_mol_std`
- `dg_exp_n`
- `platinum_pdb_id`
- `platinum_assay`
- `platinum_ligand_name`

### `models/`

Produced by `scripts/train_models.py`.

Contents:

- `manifest.json`
- one subdirectory per trained model, such as `global/` or `P10721/`
- inside each model directory:
  - `model.joblib`
  - `meta.json`

The object shape of `manifest.json` is documented by
`specs/schemas/model_manifest.schema.json`.

### `dg_predictions.tsv`

Produced by `scripts/predict_dg.py`.

The first column is `dg_pred_kcal_per_mol`, followed by the entire original
aggregated docking row. The documented row contract lives at
`specs/schemas/dg_predictions_row.schema.json`.

## Minimized Single-Run Artifacts

### Run-Level Files

The minimized pipeline writes:

- `per_cif_affinities.tsv`
- `run_affinity_summary.tsv`
- `run_affinity_summary.json`
- `batch_manifest.tsv`
- `batch_results.tsv`
- `batch_failures.tsv`
- `batch_summary.json`
- combined log output, default `pipeline.log`

### `per_cif_affinities.tsv`

Columns include:

- `boltz_run`
- `cif`
- `model_index`
- `ligand_comp_id`
- `docking_affinity_kcal_per_mol`
- `score_only_affinity_kcal_per_mol`
- `retained_cofactors_comp_ids`
- `out_dir`

### `run_affinity_summary.tsv`

Columns include:

- `boltz_run`
- `ligand_comp_id`
- `n_models`
- `vina_docking_affinity_min_kcal_per_mol`
- `vina_docking_affinity_mean_kcal_per_mol`
- `vina_docking_affinity_median_kcal_per_mol`
- `vina_docking_affinity_std_kcal_per_mol`
- `vina_score_only_affinity_min_kcal_per_mol`
- `vina_score_only_affinity_mean_kcal_per_mol`
- `vina_score_only_affinity_median_kcal_per_mol`
- `vina_score_only_affinity_std_kcal_per_mol`

The row contract is documented by
`specs/schemas/run_affinity_summary.schema.json`.

### `batch_manifest.tsv`

Documents the preflight decision for each CIF:

- found `comp_id` values
- selected ligand
- decision
- reason code
- reason detail
- cofactor-aware mode

### `batch_results.tsv`

Copies the successful row set used for the run-level summary.

### `batch_failures.tsv`

Provides one row per failed or skipped CIF with:

- `phase`
- `step`
- `error_code`
- `error_message`
- optional subprocess `returncode`

### `batch_summary.json`

Provides rolled-up counts for:

- total CIFs
- preflight `run` versus `skip` counts
- successful rows
- failed rows
- failure code counts
- ligand-routing policy flags

## Per-CIF Artifact Directory

Each per-CIF docking directory written by `meeko_cif_to_vina.py` or the
minimized runner may include:

- `ligand_comp_id.txt`
- `ligand.sdf`
- `ligand.pdbqt`
- `receptor.pdb`
- `receptor.pdbqt`
- `vina_config.txt`
- `docked.pdbqt`
- `docked.log`
- `affinity_kcal_per_mol.txt`
- minimized-path receptor preparation artifacts
- `boltz_pose_rmsd_report.txt`
- step metadata under `logs/` or script-specific log outputs

The exact set depends on the caller and whether minimization, cofactor-aware
mode, and optional debug outputs were enabled.

## Failure Behavior

The repository is intentionally strict. When a command fails because ligand
identity, chemistry reconstruction, external tooling, parsing, or training
contracts are not satisfied, it stops rather than switching to a fallback path.

The minimized runner is the main exception in terms of artifact persistence: it
still writes preflight manifests, failure rows, and rolled-up summaries even
when some CIFs are skipped or fail at runtime.

## Related Contracts

- Batch summary schema: `specs/schemas/vina_batch_summary_row.schema.json`
- Run summary schema: `specs/schemas/run_affinity_summary.schema.json`
- Prediction row schema: `specs/schemas/dg_predictions_row.schema.json`
- Model manifest schema: `specs/schemas/model_manifest.schema.json`
- Per-CIF docking executor: `meeko_cif_to_vina.py`
- Minimized runner: `scripts/score_boltz_run_minimized.py`
