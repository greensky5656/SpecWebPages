## Purpose

This repository now contains an end-to-end workflow that starts from **Boltz `.cif`** outputs and produces:

- **Raw docking scores** from **Meeko → AutoDock Vina**
- **Docking-derived feature tables** (including RDKit ligand descriptors)
- A **joined training dataset** aligned to **Platinum** experimental \(\Delta G\) (kcal/mol)
- **Calibration models** mapping docking features → Platinum scale
- **Predicted \(\Delta G\)** outputs (kcal/mol) for all docked target/ligand pairs

This file is the up-to-date **spec** for what is implemented in code. The original plan file is intentionally not modified.

## Canonical keys

- **`target_key`**: **UniProt** (e.g. `P10721`, `P00533`)
  - Rationale: some Boltz folder names do not consistently contain a 4-char PDB id token.
- **`ligand_key`**: ligand **CCD `comp_id`** detected from the CIF / written to `ligand_comp_id.txt` (e.g. `B49`, `IRE`)

All joins (docking ↔ Platinum, models ↔ predictions) occur on:

- `(target_key, ligand_key)`

## Data products (written under `vina_batch_test/` by default)

- **Batch docking summary**: `vina_batch_test/vina_batch_summary.tsv`
  - Produced by `run_all_p_runs.py` (called via wrapper `scripts/run_cif_batch.py`)
- **Per-CIF feature table**: `vina_batch_test/docking_per_cif_features.tsv`
- **Aggregated feature table** (across CIF models per target/ligand): `vina_batch_test/docking_agg_features.tsv`
- **Parsed Platinum TSV**: `vina_batch_test/platinum_parsed.tsv`
- **Joined training dataset**: `vina_batch_test/training_dataset.tsv`
- **Models**: `vina_batch_test/models/`
  - `manifest.json`
  - per-model `model.joblib` + `meta.json`
- **Predictions**: `vina_batch_test/dg_predictions.tsv`

## Implemented CLIs

### Run Meeko→Vina docking across all `p0*` and `p1*` runs

```bash
/Users/hansparsons/Github/autodockvina/.venv/bin/python scripts/run_cif_batch.py --overwrite
```

This wraps `run_all_p_runs.py` and produces `vina_batch_test/vina_batch_summary.tsv`.

### Build docking features (per-CIF + aggregated)

```bash
/Users/hansparsons/Github/autodockvina/.venv/bin/python scripts/build_docking_features.py \
  --docking-tsv vina_batch_test/vina_batch_summary.tsv \
  --out-per-cif vina_batch_test/docking_per_cif_features.tsv \
  --out-agg vina_batch_test/docking_agg_features.tsv
```

### Parse Platinum

```bash
/Users/hansparsons/Github/autodockvina/.venv/bin/python scripts/parse_platinum.py \
  --in-md platinumresults.md \
  --out-tsv vina_batch_test/platinum_parsed.tsv
```

### Join docking ↔ Platinum into a training dataset

```bash
/Users/hansparsons/Github/autodockvina/.venv/bin/python scripts/build_dataset.py \
  --docking-agg vina_batch_test/docking_agg_features.tsv \
  --platinum-tsv vina_batch_test/platinum_parsed.tsv \
  --out-dataset vina_batch_test/training_dataset.tsv
```

Platinum duplicates for the same key are aggregated:

- `dg_exp_kcal_per_mol` = mean across Platinum rows for that `(target_key, ligand_key)`
- `dg_exp_n`, `dg_exp_kcal_per_mol_std` are recorded

### Train calibration model(s)

```bash
/Users/hansparsons/Github/autodockvina/.venv/bin/python scripts/train_models.py \
  --dataset vina_batch_test/training_dataset.tsv \
  --out-dir vina_batch_test/models \
  --mode global
```

`--mode per-target` is supported, but it requires at least 2 training rows per target.

### Predict calibrated \(\Delta G\)

```bash
/Users/hansparsons/Github/autodockvina/.venv/bin/python scripts/predict_dg.py \
  --docking-agg vina_batch_test/docking_agg_features.tsv \
  --model-dir vina_batch_test/models \
  --out-tsv vina_batch_test/dg_predictions.tsv
```

## ABFE pilot (implemented hook)

The ABFE pilot expects **prepared AMBER** `prmtop/inpcrd` inputs for:

- solvated complex (protein+ligand+solvent)
- solvated ligand (ligand+solvent)

It provides:

- decoupling via OpenMM + BAR across a lambda schedule
- analytical Boresch standard-state correction
- a merge path (`scripts/merge_abfe.py`) to include `dg_abfe_kcal_per_mol` as a model feature (`--use-abfe-feature`).

