# AutoDock Vina RMSD Pipeline

This repository implements a strict Boltz-to-docking and affinity-analysis
workflow built around `meeko`, `AutoDock Vina`, optional restrained OpenMM
minimization, feature engineering, Platinum joins, ridge calibration, optional
ABFE feature merge, and prediction export.

Pipeline flowchart:

- Rendered SVG: `docs/pipeline_flowchart.svg`
- Editable Mermaid source: `docs/pipeline_flowchart.md`

![Pipeline flowchart](docs/pipeline_flowchart.svg)

## Documentation

The files below are the current maintained entrypoints. Older exploratory notes
and historical markdown files at the repository root may not reflect the
present implementation.

Start with these current documents:

- [Setup and install guide](docs/setup_install_guide.md): micromamba
  environment setup, tool prerequisites, quick starts, and verification checks
- [AutoDock Vina RMSD implementation specification](docs/autodockvina_rmsd_implementation_spec.md):
  refreshed specification entrypoint for the current implementation
- [Design specification](docs/design_specification.md): current implemented
  architecture, execution styles, validation rules, and runtime assumptions
- [CLI reference](docs/cli_reference.md): command-by-command usage, arguments,
  outputs, and failure behavior
- [Runtime and artifacts reference](docs/runtime_artifacts_reference.md): output
  directory structure, important files, and artifact meanings
- [Per-model docking and minimization method](docs/per_model_docking_and_minimization_method.md):
  the literal per-CIF preparation, score-only, and docking method
- [Pipeline scoring and calibration chain](docs/pipeline_scoring_and_calibration_chain.md):
  the whole data chain from docking summaries to calibrated `dg_predictions`
- [Pipeline flowchart](docs/pipeline_flowchart.md): editable Mermaid source for
  the current end-to-end workflow
- [Rendered pipeline flowchart](docs/pipeline_flowchart.svg): visual diagram for
  environments that do not render Mermaid

For machine-readable data contracts, use the current files under
`specs/schemas/` directly.

The current codebase supports:

- Batch docking across top-level `p0*` and `p1*` Boltz run folders
- Single-run minimized scoring through `scripts/score_boltz_run_minimized.py`
- Single-run direct scoring through `scripts/score_boltz_run.py`
- Ligand identity enforcement from Boltz YAML via CCD or SMILES definitions
- Optional cofactor-aware processing and deterministic preflight routing
- Per-CIF and aggregated docking feature tables
- Platinum dataset parsing, docking-to-Platinum joins, ridge calibration, and
  calibrated `dg_predictions.tsv`
- Optional ABFE TSV merge as an additional training feature

The minimized batch runner records three per-CIF dispositions:

- `run`
- `skip`
- runtime `error`

## What The Pipeline Does

For Boltz `*.cif` inputs, the current repository:

1. Parses Boltz run folders, CIF models, and optional Boltz results YAML files.
2. Identifies the target ligand and optional retained cofactors.
3. Builds receptor and ligand artifacts from each CIF model.
4. Optionally performs protein-only preparation and restrained OpenMM
   minimization before docking.
5. Runs Vina score-only on the retained Boltz pose and Vina docking or
   redocking.
6. Aggregates per-CIF results into run-level and batch-level TSV or JSON
   outputs.
7. Computes ligand descriptors and aggregated docking features by
   `(target_key, ligand_key)`.
8. Parses Platinum reference data and joins it to the docking aggregates.
9. Trains ridge calibration models and writes calibrated `dg_predictions.tsv`.

## Repository Layout

- `affinity/`: docking parsing, feature aggregation, Platinum parsing, joins,
  calibration, prediction, and ABFE merge logic
- `affinity/abfe/`: OpenMM ABFE pilot helpers
- `scripts/`: maintained orchestration CLIs and helper commands
- `tests/`: focused `unittest` coverage for Boltz YAML routing, score-only
  parsing, pose guards, and minimized preflight logic
- `docs/`: maintained documentation entrypoints for this repository
- `specs/schemas/`: machine-readable row or object contracts for key generated
  artifacts
- `meeko_cif_to_vina.py`: primary per-CIF docking executor
- `run_all_p_runs.py`: low-level batch driver for top-level `p0*` and `p1*`
  folders
- `run_all_cifs.py`: lower-level driver for all CIFs under one run folder

## Installation

This repository does not currently declare `pyproject.toml`, `setup.py`, or a
pip-installable package. The implemented workflow is run from the repository
root via a pinned micromamba environment.

```bash
micromamba create -y -p .micromamba/envs/amber311 \
  --override-channels -c conda-forge \
  python=3.11 ambertools pdbfixer openmm rdkit meeko=0.7.1 gemmi \
  scikit-learn joblib
```

The current scripts assume Python 3.11 tooling and external binaries such as
`vina`, plus AmberTools executables for the minimized path.

## CLI

### Run The Minimized Single-Run Pipeline

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/score_boltz_run_minimized.py \
  p10721kd_b49_wt_3g0e-c971d8454c63 \
  --vina-bin ./vina_1.2.7_linux_x86_64 \
  --overwrite
```

This command writes a structured per-run artifact tree under either
`vina_boltz_minimized/<run>/` or the input Boltz results root's
`vina_analysis/<run>/` directory, depending on input layout.

### Run Batch Docking Across `p0*` And `p1*`

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/run_cif_batch.py \
  --out-root vina_batch_test \
  --out-tsv vina_batch_test/vina_batch_summary.tsv \
  --vina-bin ./vina_1.2.7_linux_x86_64 \
  --overwrite
```

### Build Docking Features

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/build_docking_features.py \
  --docking-tsv vina_batch_test/vina_batch_summary.tsv \
  --out-per-cif vina_batch_test/docking_per_cif_features.tsv \
  --out-agg vina_batch_test/docking_agg_features.tsv
```

### Parse Platinum And Build The Training Dataset

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/parse_platinum.py \
  --in-md platinumresults.md \
  --out-tsv vina_batch_test/platinum_parsed.tsv

micromamba run -p .micromamba/envs/amber311 \
  python scripts/build_dataset.py \
  --docking-agg vina_batch_test/docking_agg_features.tsv \
  --platinum-tsv vina_batch_test/platinum_parsed.tsv \
  --out-dataset vina_batch_test/training_dataset.tsv
```

### Train Calibration Models

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/train_models.py \
  --dataset vina_batch_test/training_dataset.tsv \
  --out-dir vina_batch_test/models \
  --mode global
```

### Predict Calibrated `ΔG`

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/predict_dg.py \
  --docking-agg vina_batch_test/docking_agg_features.tsv \
  --model-dir vina_batch_test/models \
  --out-tsv vina_batch_test/dg_predictions.tsv
```

### Run The End-To-End CIF-Keyed Workflow

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/run_pipeline_from_cif.py \
  p10721kd_b49_wt_3g0e-c971d8454c63/run-0000-cifs/p10721kd_b49_wt_3g0e-c971d8454c63_model_0.cif \
  --vina-bin ./vina_1.2.7_linux_x86_64 \
  --train-mode global \
  --overwrite
```

This six-step command prints the matched `dg_pred_kcal_per_mol` row for the CIF
input after regenerating the docking, features, Platinum parse, dataset, model,
and predictions.

## Configuration

The current implementation uses CLI flags rather than a standalone YAML runtime
config. Important configured behaviors are:

- `scripts/run_cif_batch.py` and `run_all_p_runs.py` scan top-level `p0*` and
  `p1*` folders under the repository root
- `target_key` is inferred from the leading run token such as
  `p10721... -> P10721`
- `ligand_key` is the selected CCD `comp_id` written via `ligand_comp_id.txt`
- `scripts/train_models.py --mode global` requires at least two training rows
  total
- `scripts/train_models.py --mode per-target` requires at least two rows per
  `target_key`
- `scripts/score_boltz_run_minimized.py` changes its default output root based
  on whether the input is a legacy run folder or a Boltz results root

## Execution Styles

### Standard Docking Path

The standard path runs `meeko_cif_to_vina.py` directly from
`run_all_p_runs.py`, `scripts/run_cif_batch.py`, or
`scripts/score_boltz_run.py` without the Option A protein minimization stage.

### Minimized Option A Path

The minimized path is exposed by `scripts/score_boltz_run_minimized.py` and
adds:

- `pdb4amber` normalization
- `cpptraj prepareforleap`
- `tleap` topology construction
- restrained OpenMM minimization
- Vina score-only on the retained Boltz pose
- Vina docking or redocking plus batch manifests and failure summaries

### End-To-End Calibration Path

The end-to-end calibration path is exposed by
`scripts/run_pipeline_from_cif.py` and chains:

- batch docking
- feature generation
- Platinum parsing
- dataset join
- model training
- prediction emission

## Inputs And Supported Scope

The repository currently expects one of these input shapes:

- top-level Boltz run folders at the repository root with names starting `p0*`
  or `p1*`
- a single Boltz run folder passed to `scripts/score_boltz_run.py` or
  `scripts/score_boltz_run_minimized.py`
- a `run-*-cifs/` subfolder passed to the same single-run scorers
- a Boltz results root passed to `scripts/score_boltz_run_minimized.py`,
  optionally with `--boltz-yaml`

Current ligand-selection behavior includes:

- direct `ligand_comp_id` forcing
- Boltz YAML CCD-based ligand routing
- Boltz YAML SMILES-based ligand reconstruction
- optional auxiliary comp-id filtering for non-target ligands
- optional cofactor-aware routing when Boltz YAML reports retained non-target
  ligands

## Compute Notes

The real-compute path depends on environment tooling beyond the Python standard
library, including:

- AutoDock Vina
- AmberTools executables such as `pdb4amber`, `cpptraj`, and `tleap`
- OpenMM
- RDKit
- Meeko
- gemmi
- scikit-learn
- joblib

The code is intentionally strict and does not introduce fallback paths when
required chemistry, routing, parsing, or external tooling is missing.

## Output Artifacts

Important generated paths include:

- `vina_batch_test/vina_batch_summary.tsv`
- `vina_batch_test/docking_per_cif_features.tsv`
- `vina_batch_test/docking_agg_features.tsv`
- `vina_batch_test/platinum_parsed.tsv`
- `vina_batch_test/training_dataset.tsv`
- `vina_batch_test/models/manifest.json`
- `vina_batch_test/models/*/meta.json`
- `vina_batch_test/dg_predictions.tsv`
- `vina_boltz_minimized/<run>/per_cif_affinities.tsv`
- `vina_boltz_minimized/<run>/run_affinity_summary.tsv`
- `vina_boltz_minimized/<run>/batch_manifest.tsv`
- `vina_boltz_minimized/<run>/batch_results.tsv`
- `vina_boltz_minimized/<run>/batch_failures.tsv`
- `vina_boltz_minimized/<run>/batch_summary.json`

The per-CIF minimized outputs also preserve receptor, ligand, docking, logging,
and RMSD guard artifacts needed for debugging and reproducibility.

## Testing

The current test suite is `unittest`-based.

```bash
python3 -m unittest discover tests
```

Coverage is concentrated in:

- `tests/test_boltz_yaml_and_score_only.py`
- `tests/test_score_boltz_run_minimized_preflight.py`

## Current Safety Model

The implementation is intentionally fail-fast and artifact-oriented:

- invalid or ambiguous ligand identity is classified and skipped in the
  minimized batch preflight path
- missing tools, malformed inputs, failed chemistry reconstruction, and model
  training contract violations stop the active command
- the code does not add backup execution routes when required criteria are not
  met
