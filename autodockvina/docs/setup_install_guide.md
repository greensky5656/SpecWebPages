# Setup And Install Guide

This guide reflects the current repository layout and runtime behavior.

## Workflow Families

There are two practical setup targets:

- `standard_docking_and_calibration`: direct docking, feature generation,
  Platinum joins, model training, and prediction export
- `minimized_single_run`: the same docking core plus Option A protein-only
  preparation, restrained OpenMM minimization, score-only evaluation, and
  preflight classification

If you only want to exercise the maintained docking and calibration chain, start
with `standard_docking_and_calibration`.

## Baseline Requirements

- Linux or another Unix-like environment
- `micromamba`
- Python 3.11
- AutoDock Vina
- AmberTools executables for the minimized path
- OpenMM
- RDKit
- Meeko `0.7.1`
- gemmi
- scikit-learn
- joblib

This repository currently runs from source and does not expose a packaged
install such as `pip install -e .`.

## Create The Working Environment

From the repository root:

```bash
micromamba create -y -p .micromamba/envs/amber311 \
  --override-channels -c conda-forge \
  python=3.11 ambertools pdbfixer openmm rdkit meeko=0.7.1 gemmi \
  scikit-learn joblib
```

If a stale environment from another operating system already exists, remove it
before recreating the environment.

## Sanity Checks

Confirm that the core Python dependencies import cleanly:

```bash
micromamba run -p .micromamba/envs/amber311 \
  python -c "import openmm, rdkit, gemmi, meeko, sklearn, joblib; print('imports ok')"
```

For the minimized path, also verify AmberTools executables are available inside
the environment:

```bash
micromamba run -p .micromamba/envs/amber311 \
  python -c "import shutil; tools=['pdb4amber','cpptraj','tleap']; print({t: bool(shutil.which(t)) for t in tools})"
```

## Vina Binary Expectations

The repository already contains Vina binaries, but several scripts still default
to checked-in macOS paths:

- `scripts/score_boltz_run_minimized.py` defaults to `vina_1.2.7_mac_aarch64`
- `scripts/run_cif_batch.py`, `scripts/score_boltz_run.py`, and
  `scripts/run_pipeline_from_cif.py` default to
  `autodock_vina_1_1_2_mac_catalina_64bit/bin/vina`

For consistent Linux usage, pass:

- `--vina-bin ./vina_1.2.7_linux_x86_64`

## Quick Start: Minimized Single-Run Workflow

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/score_boltz_run_minimized.py \
  p10721kd_b49_wt_3g0e-c971d8454c63 \
  --vina-bin ./vina_1.2.7_linux_x86_64 \
  --overwrite
```

This writes a per-run artifact tree under `vina_boltz_minimized/<run>/` for
legacy run folders, or under `<boltz_results_root>/vina_analysis/<run>/` when
the input is a Boltz results root.

## Quick Start: Batch Docking And Calibration

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/run_cif_batch.py \
  --out-root vina_batch_test \
  --out-tsv vina_batch_test/vina_batch_summary.tsv \
  --vina-bin ./vina_1.2.7_linux_x86_64 \
  --overwrite

micromamba run -p .micromamba/envs/amber311 \
  python scripts/build_docking_features.py \
  --docking-tsv vina_batch_test/vina_batch_summary.tsv \
  --out-per-cif vina_batch_test/docking_per_cif_features.tsv \
  --out-agg vina_batch_test/docking_agg_features.tsv

micromamba run -p .micromamba/envs/amber311 \
  python scripts/parse_platinum.py \
  --in-md platinumresults.md \
  --out-tsv vina_batch_test/platinum_parsed.tsv

micromamba run -p .micromamba/envs/amber311 \
  python scripts/build_dataset.py \
  --docking-agg vina_batch_test/docking_agg_features.tsv \
  --platinum-tsv vina_batch_test/platinum_parsed.tsv \
  --out-dataset vina_batch_test/training_dataset.tsv

micromamba run -p .micromamba/envs/amber311 \
  python scripts/train_models.py \
  --dataset vina_batch_test/training_dataset.tsv \
  --out-dir vina_batch_test/models \
  --mode global

micromamba run -p .micromamba/envs/amber311 \
  python scripts/predict_dg.py \
  --docking-agg vina_batch_test/docking_agg_features.tsv \
  --model-dir vina_batch_test/models \
  --out-tsv vina_batch_test/dg_predictions.tsv
```

## Other Maintained Commands

Run the end-to-end CIF-keyed wrapper:

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/run_pipeline_from_cif.py \
  p10721kd_b49_wt_3g0e-c971d8454c63/run-0000-cifs/p10721kd_b49_wt_3g0e-c971d8454c63_model_0.cif \
  --vina-bin ./vina_1.2.7_linux_x86_64 \
  --train-mode global \
  --overwrite
```

Run the ABFE pilot helper:

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/run_abfe.py \
  --complex-prmtop complex.prmtop \
  --complex-inpcrd complex.inpcrd \
  --solvent-prmtop solvent.prmtop \
  --solvent-inpcrd solvent.inpcrd \
  --ligand-resname LIG \
  --r0-A 3.0 \
  --thetaA0-rad 1.5 \
  --thetaB0-rad 1.5 \
  --phiA0-rad 0.0 \
  --phiB0-rad 0.0 \
  --phiC0-rad 0.0 \
  --kr 10.0 \
  --kthetaA 10.0 \
  --kthetaB 10.0 \
  --kphiA 10.0 \
  --kphiB 10.0 \
  --kphiC 10.0 \
  --out-json abfe_result.json
```

## Test Command

```bash
python3 -m unittest discover tests
```

## Input Expectations

The maintained workflows expect one of these input shapes:

- top-level `p0*` and `p1*` Boltz run folders at the repository root
- a single Boltz run folder
- a `run-*-cifs/` subfolder
- a Boltz results root, optionally paired with `--boltz-yaml`

Current supported ligand-routing behavior includes:

- direct `ligand_comp_id` forcing
- Boltz YAML CCD-based routing
- Boltz YAML SMILES-based routing
- optional cofactor-aware processing
- optional auxiliary comp-id filtering for non-target ligands

Unsupported chemistry or routing cases fail fast rather than following a
fallback path.

## Where To Go Next

- Design and implementation: `docs/design_specification.md`
- CLI usage: `docs/cli_reference.md`
- Runtime outputs: `docs/runtime_artifacts_reference.md`
- Per-model method: `docs/per_model_docking_and_minimization_method.md`
- Pipeline chain: `docs/pipeline_scoring_and_calibration_chain.md`
- Flowchart: `docs/pipeline_flowchart.md` and `docs/pipeline_flowchart.svg`
- Schemas: `specs/schemas/vina_batch_summary_row.schema.json`,
  `specs/schemas/run_affinity_summary.schema.json`,
  `specs/schemas/dg_predictions_row.schema.json`, and
  `specs/schemas/model_manifest.schema.json`
