# AutoDock Vina RMSD Pipeline Specification


















This review document is grounded in current code, tests, and inspected artifacts, with targeted regression checks confirming that the reviewed docking and preflight surfaces remain stable.

- Last commit inspected: `2026-03-12 15:35:03 -0700` — `updated vina numbers and switched to .json`
- Current scientific status: **No material change identified**
- Basis priority used here: **code/tests > docs**

## Table of contents

1. [Executive summary](#1-executive-summary)
2. [The stages of the pipeline from a thermodynamic view](#2-the-stages-of-the-pipeline-from-a-thermodynamic-view)
3. [Supporting documentation](#3-supporting-documentation)
4. [Scientific workflow actually implemented](#4-scientific-workflow-actually-implemented)
5. [What scientific reviewers should separate](#5-what-scientific-reviewers-should-separate)
6. [Current scientific and operational limits](#6-current-scientific-and-operational-limits)
7. [Recommended signoff questions](#7-recommended-signoff-questions)
8. [Top current clarifications](#8-top-current-clarifications)
9. [Actual shipped operator surface](#9-actual-shipped-operator-surface)
10. [Evidence appendix](#10-evidence-appendix)

## 1. Executive summary









### What this pipeline is
autodockvina is a strict Boltz-to-Meeko-to-AutoDock Vina docking workflow with a maintained minimized/cofactor-aware path, engineered docking features, and an optional classical calibration layer.

### Strongest justified claim
Its strongest justified claim is docking-based triage and feature generation, optionally improved by a downstream regression layer on top of docking-derived features.

### Main limiting factor
The main limiting factor is that the core evidence class remains docking plus calibration, not direct thermodynamic free-energy simulation.

### Appropriate present use
The appropriate present use is governed docking/scoring prioritization, not stand-alone thermodynamic affinity authority or mature ABFE-based decision support.









## 2. The stages of the pipeline from a thermodynamic view








- **Stage 1 — Structure handoff and optional receptor relaxation:** completed Boltz outputs provide a candidate bound geometry, and the maintained minimized path may refine receptor geometry before docking without turning the workflow into a thermodynamic simulation method.
- **Stage 2 — Docking evaluation:** Meeko/Vina generates docking poses and Vina score terms that act as heuristic interaction/ranking signals for the candidate bound state.
- **Stage 3 — Feature construction:** raw docking outputs are converted into engineered scoring features, summaries, and comparison tables.
- **Stage 4 — Statistical affinity mapping:** when the learned calibration layer is used, docking-derived features are mapped onto an experimental affinity scale by regression rather than by direct thermodynamic calculation.
- **Thermodynamic boundary:** neither the raw Vina scores nor the calibrated outputs should be interpreted as explicit free-energy simulations; this remains a docking-and-calibration evidence class.








## 3. Supporting documentation


















### Core documentation
- [AutoDock Vina RMSD implementation specification](../docs/autodockvina_rmsd_implementation_spec.md) — refreshed specification entrypoint for the current implementation.
- [Requirements](../docs/requirements.md): current intended use, system, input, output, functional, and fail-closed requirements.
- [Design specification](../docs/design_specification.md) — implemented architecture, execution styles, validation rules, and runtime assumptions.
- [CLI reference](../docs/cli_reference.md) — command-by-command usage, arguments, outputs, and failure behavior.
- [Runtime and artifacts reference](../docs/runtime_artifacts_reference.md) — output directory structure, important files, and artifact meanings.

### Thermodynamics and scoring documentation
- [Per-model docking and minimization method](../docs/per_model_docking_and_minimization_method.md) — literal per-CIF preparation, score-only, and docking method.
- [Pipeline scoring and calibration chain](../docs/pipeline_scoring_and_calibration_chain.md) — whole data chain from docking summaries to calibrated `dg_predictions`.

### Setup and operations
- [Setup and install guide](../docs/setup_install_guide.md) — micromamba environment setup, tool prerequisites, quick starts, and verification checks.
- [README](../docs/README.md) — high-level repository front door and current maintained documentation map.

### Flowchart
- [Rendered pipeline flowchart](../docs/pipeline_flowchart.svg) — quick visual workflow view.
- [Editable flowchart source](../docs/pipeline_flowchart.md) — Mermaid source for the workflow map.

### Operator entrypoints
- Main run surfaces remain `scripts/run_selected_boltz_runs.sh` and `scripts/score_boltz_run_minimized.py`.
- The simplified legacy wrapper `scripts/score_boltz_run.py` should be interpreted separately from the minimized/cofactor-aware mainline.


















## 4. Scientific workflow actually implemented

### A. Ligand and receptor extraction from Boltz outputs

The code reads Boltz mmCIF with `gemmi`, identifies polymer entities for the receptor, and identifies ligand atoms by selected `comp_id`. Ligand selection can come from:

- a forced `--ligand-comp-id`
- Boltz YAML ligand definitions when `--boltz-yaml` is provided
- CIF non-polymer inspection / auto-detection in the no-YAML path
- auxiliary-comp-id filtering in the minimized batch preflight path

### B. Ligand chemistry source is optional-YAML plus CCD fallback

The current code does **not** make `--boltz-yaml` strictly mandatory. Instead, ligand chemistry is sourced as follows:

- if Boltz YAML is provided and contains ligand SMILES, the code builds the ligand from that SMILES while preserving Boltz coordinates
- otherwise, it falls back to the CIF ligand `comp_id`, resolves that CCD entry to SMILES, and builds the ligand from the CCD-derived chemistry while preserving Boltz coordinates

In both cases the code writes `ligand.sdf` and enforces a **pose-preservation RMSD guard** between Boltz CIF coordinates and the RDKit/PDBQT representations.

### C. Receptor preparation and optional cofactor retention

The receptor starts as polymer-only PDB extracted from the CIF. If `--minimize-receptor` is enabled, the code runs:

- `pdb4amber`
- `cpptraj prepareforleap`
- `tleap`
- restrained OpenMM minimization

If cofactor-aware mode is enabled, YAML-listed **non-target ligands** are merged into the receptor as rigid cofactors, while the binder ligand remains excluded from the receptor.

### D. Two distinct Vina score pathways

The implemented workflow explicitly separates two Vina quantities:

1. **Raw original-pose Vina score**: `--score_only` against the prepared receptor, where the ligand pose does not move.
2. **Raw redocking Vina score**: a full docking search that writes `docked.pdbqt` and reports the best docked affinity.

The code also writes a director-style summary interpreting the difference `dock - score_only`.

### E. Engineered docking feature generation

`affinity/parse_docking.py` turns docking TSV rows into structured per-CIF rows and computes ligand descriptors from `ligand.sdf`, including:

- molecular weight
- logP
- TPSA
- H-bond donors/acceptors
- rotatable bonds
- ring count
- heavy atom count
- formal charge
- fraction Csp3

`affinity/dataset_build.py` then aggregates by `(target_key, ligand_key)` and computes:

- `vina_affinity_min`
- `vina_affinity_mean`
- `vina_affinity_median`
- `vina_affinity_std`

along with the ligand descriptors.

### F. Experimental join and calibration layer

`affinity/join_dataset.py` joins docking aggregates with Platinum data by **`(target_key, ligand_key)`** and, when multiple Platinum rows exist for a key, stores:

- mean `dg_exp_kcal_per_mol`
- `dg_exp_kcal_per_mol_std`
- `dg_exp_n`

`affinity/train_calibrator.py` then trains a **Pipeline(StandardScaler, Ridge(alpha=1.0))** in either:

- `global` mode
- `per-target` mode

Models are saved to disk with manifest and per-model metadata, including **training-set MAE**.

`affinity/predict.py` loads those saved models and emits **`dg_pred_kcal_per_mol`** for aggregated docking rows.

### G. ABFE pilot hook

`affinity/abfe/abfe_pilot.py` implements a separate **double-decoupling ABFE pilot**:

- complex-leg annihilation decoupling
- solvent-leg annihilation decoupling
- analytical Boresch standard-state correction
- uncertainty propagation from the two BAR legs

This ABFE path requires already-prepared AMBER `prmtop`/`inpcrd` files and is not used by the mainline Meeko→Vina operators.

## 5. What scientific reviewers should separate



### Native evidence
The native evidence is the direct docking workflow output: Vina score-only and docking scores, per-CIF results, run-level summaries, and the maintained script-driven scoring path. Those raw outputs are the first-class native evidence for this repository.

### Transformed / calibrated evidence
Engineered docking features and the Ridge-based affinity mapping layer are transformed evidence built on top of raw docking outputs. These transformed values can be useful for governed triage, but they are not a direct thermodynamic measurement layer.

### Detached analyst artifacts
ABFE pilot utilities, exploratory feature experiments, and analyst-side merged datasets are detached artifacts unless they are part of the maintained default scoring workflow. They should be discussed separately from the repository's mainline docking path.

### Non-authoritative surfaces
Raw Vina scores are not calibrated free energies. Calibrated regression outputs are not equivalent to a physics-based free- energy engine. Pilot ABFE hooks should not be presented as the default operator surface.



## 6. Current scientific and operational limits



### Scientific limits
The workflow remains a docking-and-calibration evidence class rather than a direct thermodynamic free-energy method. Model evaluation in the core training path is still narrow and does not by itself establish broad external validation or uncertainty calibration. ABFE utilities remain pilot surfaces and should not be treated as the default scientific contract.

### Operational / runtime limits
The main per-CIF operator effectively requires Boltz YAML even where older help text suggests optionality. Auto ligand routing is conservative and may skip ambiguous cases rather than guessing. Documentation and binary defaults still show some platform drift, including Linux-focused README examples alongside bundled macOS Vina binary defaults in code.

### Evidence / provenance limits
Tests mainly validate routing, parsing, cofactor handling, and batch preflight behavior rather than broad scientific accuracy. Raw docking affinity aliases and redocking summaries require careful filename reading to avoid misclassification of what quantity is being reported. Detached pilot and analyst-side surfaces can easily be mistaken for the main workflow unless explicitly separated.

### Signoff consequence
This repository is appropriate for governed docking-based triage and feature generation, but it should not be signed off as a mature thermodynamic affinity engine or as a benchmark-proven calibrated endpoint on its present evidence.



## 7. Recommended signoff questions



### Scientific validity question
Is the intended scientific claim explicitly limited to docking/scoring plus a classical calibration layer, rather than direct physical free-energy prediction?

### Operational readiness question
Is the current operator surface acceptable as deployed, including the Boltz YAML requirement, cofactor-aware routing rules, platform-specific binary defaults, and the distinction between the maintained minimized path and legacy wrappers?

### Evidence / provenance question
Before stronger affinity claims are made, is additional out-of-sample validation required beyond the current training-set-oriented evidence and routing/parsing test coverage?

### Deployment-decision question
Should the repository be signed off only as a governed docking/triage tool, with ABFE pilot utilities and legacy wrappers kept outside the main approved operator surface?



## 8. Top current clarifications


















1. **No material change identified.** This page continues to reflect the same current docking, minimization, and calibration surfaces.
2. **The minimized orchestrator plus `meeko_cif_to_vina.py` is the real signoff surface.** It is the only path here that cleanly exposes score-only vs redock separation, cofactor-aware handling, preflight routing, and structured failure accounting.
3. **The learned layer is still classical Ridge calibration on engineered docking features.** It is not an end-to-end learned docking-affinity platform.
4. **ABFE is still a pilot hook, not the mainline operator contract.** It requires prepared AMBER topologies and separate scientific validation.
5. **One newly emphasized clarification:** `--boltz-yaml` is operationally required in the main per-CIF tool even though some help/docs still imply optional use.
6. **One newly emphasized clarification:** repo docs and bundled default Vina binary targets are not fully aligned, so deployment signoff should rely on code-level operator paths rather than README wording alone.





## 9. Actual shipped operator surface


















### Primary per-CIF operator: `meeko_cif_to_vina.py`

The core CLI accepts a Boltz `.cif`, output directory, ligand routing controls, cofactor-aware toggle, receptor minimization controls, Vina controls, and logging controls. The implemented 10-step run is:

1. Parse CIF and detect/extract ligand and receptor entities.
2. Read Boltz YAML ligand roster/definition.
3. Write polymer receptor PDB.
4. Optionally minimize receptor with `pdb4amber`/`tleap`/OpenMM.
5. Build ligand chemistry from **Boltz YAML SMILES** or **Boltz YAML CCD** while preserving Boltz coordinates.
6. Convert ligand to PDBQT with Meeko.
7. Guard ligand pose with RMSD checks.
8. Convert receptor (polymer only or polymer+cofactors) to rigid receptor PDBQT.
9. Write Vina box/config.
10. Run both **Vina score-only** and **Vina redocking** unless `--no-docking` is used.

Key output files written by this operator include:

- `ligand_comp_id.txt`
- `retained_cofactors.txt`
- `ligand.sdf`, `ligand.pdbqt`, `receptor.pdbqt`
- `boltz_pose_rmsd_report.txt`
- `vina_config.txt`
- `score_only_affinity_kcal_per_mol.txt`
- `docking_affinity_kcal_per_mol.txt`
- `affinity_kcal_per_mol.txt` (same value as docking affinity)
- `pipeline.log` and optional JSONL/artifacts

### Main batch/run operator: `scripts/score_boltz_run_minimized.py`

This is the main shipped orchestrator for a Boltz run folder or Boltz results root. It:

- finds CIFs under the run
- can auto-route ligand selection per CIF
- can auto-enable cofactor-aware mode when non-target ligands exist in Boltz YAML
- calls `meeko_cif_to_vina.py` per CIF with receptor minimization enabled
- records preflight decisions, per-CIF results, failures, and run-level summaries

Key batch outputs are:

- `per_cif_affinities.tsv`
- `run_affinity_summary.tsv`
- `run_affinity_summary.json`
- `batch_manifest.tsv`
- `batch_results.tsv`
- `batch_failures.tsv`
- `batch_summary.json`

Each per-CIF row carries both:

- `docking_affinity_kcal_per_mol`
- `score_only_affinity_kcal_per_mol`

plus any retained cofactor comp IDs.

### Simpler legacy operator: `scripts/score_boltz_run.py`

This older run-level wrapper still exists, but it only reports **redocking affinity** across CIFs. It does **not** expose:

- score-only vs redock separation
- cofactor-aware receptor merging
- minimized receptor workflow
- structured batch preflight/failure accounting

### Downstream affinity/calibration modules

Implemented downstream modules are:

- `affinity/parse_docking.py`: parse docking TSV and compute RDKit ligand descriptors from `ligand.sdf`
- `affinity/dataset_build.py`: aggregate per-CIF docking rows by `(target_key, ligand_key)` and compute min/mean/median/std docking summaries
- `affinity/join_dataset.py`: join docking aggregates with Platinum experimental `dg_exp_kcal_per_mol`
- `affinity/train_calibrator.py`: train **global** or **per-target** StandardScaler+Ridge calibrators and save `model.joblib` and metadata
- `affinity/predict.py`: apply saved calibrator(s) to aggregated docking features and emit `dg_pred_kcal_per_mol`
- `affinity/abfe/abfe_pilot.py` and `affinity/merge_abfe.py`: separate ABFE pilot and merge hook




## 10. Evidence appendix


















### Git and run-state evidence

- Repository inspected: `/home/joethomas/Documents/Github/autodockVinaRMSD/autodockvina`
- Targeted review emphasis: docking entrypoints, minimized receptor path, and calibration-facing outputs.

### Test evidence

Command run:

```bash
python -m pytest -q tests/test_boltz_yaml_and_score_only.py tests/test_score_boltz_run_minimized_preflight.py
```

Observed result:

- `32 passed, 3 skipped in 0.13s`

### Consulted files

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
- `/home/joethomas/Documents/Github/autodockVinaRMSD/autodockvina/affinity/merge_abfe.py`
- `/home/joethomas/Documents/Github/autodockVinaRMSD/autodockvina/tests/test_boltz_yaml_and_score_only.py`
- `/home/joethomas/Documents/Github/autodockVinaRMSD/autodockvina/tests/test_score_boltz_run_minimized_preflight.py`
- Supporting context only: `/home/joethomas/Documents/Github/autodockVinaRMSD/autodockvina/README.md`
- Supporting context only: `/home/joethomas/Documents/Github/autodockVinaRMSD/autodockvina/PIPELINE_SPEC.md`

### No-material-change note

No material change identified: the docking, minimized-receptor, and calibration-facing surfaces remain unchanged in material scientific terms, and targeted tests still pass.
