# One-Page Workflow

This is the shortest maintained summary of what `md_pipeline` does for one
`boltz_results_*` directory.

## Purpose

Starting from Boltz predictions plus Vina pose-ranking outputs, the workflow:

1. selects one best model,
2. validates that the receptor and ligand files match that model,
3. prepares an Amber/OpenMM-compatible solvated system,
4. optionally applies protonation-state workflows,
5. runs restrained equilibration and production MD,
6. analyzes trajectory stability,
7. runs MM/GBSA and writes one final combined summary.

## Command

```bash
python -m md_pipeline full --run-dir /path/to/boltz_results_example
```

Other commands:

- `check-openmm`: OpenMM runtime preflight
- `select`: selection only
- `prepare`: selection + validation + system build
- `simulate`: preparation + MD
- `analyze`: reuse prep/MD summaries, then analysis + MM/GBSA
- `full`: complete workflow

## Inputs

The run directory must contain:

- `predictions/<boltz_run>/<boltz_run>_model_*.cif`
- `vina_analysis/<boltz_run>/batch_results.tsv`
- `vina_analysis/<boltz_run>/batch_manifest.tsv`
- `vina_analysis/<boltz_run>/batch_summary.json`
- `vina_analysis/<boltz_run>/<boltz_run>_model_N/receptor.pdb`
- `vina_analysis/<boltz_run>/<boltz_run>_model_N/ligand.sdf`

## Step-By-Step Pipeline

### 1. Select The Best Model

- reads the batch ranking files
- chooses the unique minimum of `score_only_affinity_kcal_per_mol`
- writes `selected_model.json`

### 2. Validate The Pose Assets

- checks the chosen CIF, receptor PDB, and ligand SDF for consistency
- writes `pose_validation.json`

### 3. Prepare The System

- cleans the receptor for AmberTools
- parameterizes the ligand
- builds a solvated, neutralized, salted system
- writes `prep_summary.json`

### 4. Optional Protonation

- protein mode: PROPKA/PDB2PQR
- ligand mode: Epik
- protonation artifacts are referenced from `prep_summary.json` and
  `run_summary.json` when enabled

### 5. Run Molecular Dynamics

Default MD schedule:

1. `nvt_heavy` - `1.0 ns`
2. `npt_heavy` - `0.5 ns`
3. `npt_backbone_strong` - `0.75 ns`
4. `npt_backbone_light` - `0.75 ns`
5. `production` - `20.0 ns`

Writes `md_summary.json` plus the production trajectory.

### 6. Analyze The Production Trajectory

Measures:

- protein backbone RMSD
- ligand RMSD
- pocket RMSD
- contacts
- minimum distances
- hydrogen-bond metrics

Writes `analysis/trajectory/trajectory_summary.json`.

### 7. Run MM/GBSA

- prepares GROMACS-compatible inputs
- runs GB and/or PB MM/GBSA through `gmx_MMPBSA`
- supports optional per-residue and pairwise decomposition

Important outputs:

- `analysis/mmgbsa/FINAL_RESULTS_MMPBSA.dat`
- `analysis/mmgbsa/FINAL_RESULTS_MMPBSA.csv`
- `analysis/mmgbsa/mmgbsa_summary.json`

### 8. Final Summary

The final combined record is:

- `run_summary.json`

## Bottom Line

This pipeline is a strict, single-pose, explicit-solvent MD + MM/GBSA screening
workflow for relative comparison of related protein-ligand systems.
