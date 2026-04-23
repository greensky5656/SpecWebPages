# Scientist Guide

This document explains the workflow in scientific terms for users who want to
understand what the code is doing and how to interpret the outputs.

## What This Pipeline Does

For one `boltz_results_*` directory, the pipeline:

1. selects one best Boltz/Vina pose,
2. checks that the receptor and ligand files match that selected pose,
3. builds an Amber/OpenMM-compatible solvated system,
4. runs restrained equilibration and production MD,
5. computes stability-style trajectory metrics,
6. runs MM/GBSA for relative ranking,
7. writes a combined `run_summary.json`.

This is a post-processing and ranking workflow. It is not de novo docking and
not an absolute free-energy method.

## One-Sentence Scientific Summary

For each Boltz/Vina run, the workflow chooses one pose, validates that the
structure assets are self-consistent, relaxes that pose with explicit-solvent
OpenMM MD, and reports relative endpoint energetics with MM/GBSA.

## Expected Inputs

Each run depends on:

- Boltz prediction CIFs,
- Vina batch ranking outputs,
- a receptor PDB per candidate model,
- a ligand SDF per candidate model.

The expected filesystem layout is documented in
`docs/setup_install_guide.md` and `docs/one_page_workflow.md`.

## Model Selection

The workflow chooses one model based on the unique minimum of
`score_only_affinity_kcal_per_mol`.

Important implications:

- the chosen model is not selected by `docking_affinity_kcal_per_mol`,
- ties for best score are treated as an error,
- the workflow is designed around a single selected pose rather than ensemble
  ranking across all candidate poses.

## Pose Validation

Before simulation, the code checks that the selected:

- CIF,
- receptor PDB,
- ligand SDF

describe the same bound complex.

Scientifically, this protects against a common failure mode where model
selection metadata and downstream structure files drift out of sync.

## Preparation

The preparation stage builds a solvated protein-ligand system suitable for
OpenMM simulation.

Current defaults:

- protein force field: `ff14SB`
- ligand parameterization backend: `gaff2`
- water model: `TIP3P`
- solvent padding: `10 A`
- ionic strength: `0.15 M`

Preparation is intended to produce one explicit-solvent screening system, not to
search multiple protonation or force-field alternatives automatically.

## Optional Protonation

The current codebase supports opt-in protonation workflows:

- protein protonation through PROPKA/PDB2PQR
- ligand protonation through Epik

These are disabled by default.

They are useful when protonation-state handling is a known scientific risk, but
they also introduce additional external-tool dependencies and should be treated
as explicit experimental choices.

## Molecular Dynamics

The MD workflow uses OpenMM with restrained equilibration followed by unrestrained
production.

Default stage schedule:

1. `nvt_heavy` for `1.0 ns`
2. `npt_heavy` for `0.5 ns`
3. `npt_backbone_strong` for `0.75 ns`
4. `npt_backbone_light` for `0.75 ns`
5. `production` for `20.0 ns`

Interpretation:

- the early restrained stages are for relaxation and density/box equilibration,
- the production stage is the main trajectory used for analysis and MM/GBSA,
- this is a short screening protocol, not an exhaustive sampling strategy.

## Trajectory Analysis

The trajectory-analysis stage provides quick stability diagnostics, including:

- protein backbone RMSD
- ligand heavy-atom RMSD
- pocket RMSD
- ligand-protein contact counts
- minimum ligand-protein distances
- protein-ligand hydrogen-bond metrics

These metrics are intended as a sanity check before interpreting MM/GBSA.

## MM/GBSA

The endpoint energetic stage uses GROMACS + `gmx_MMPBSA`.

Current supported modes:

- GB
- PB
- GB and PB together
- optional per-residue decomposition
- optional pairwise decomposition

Important defaults:

- entropy method defaults to `none`
- the default MM/GBSA window skips the first `2 ns` of the `20 ns` production
  trajectory

Scientific interpretation:

- the most stable first-pass outputs are typically the enthalpy-like MM/GBSA
  terms,
- MM/GBSA is most useful here for relative ranking within closely related
  systems,
- single-trajectory endpoint methods remain sensitive to trajectory quality and
  sampling variance.

## Random Seed And Replica Guidance

The code supports deterministic random seeds through:

- `--prep-random-seed`
- `--md-random-seed`

Use a fixed seed when reproducibility of one run matters.

For comparative ranking, replica thinking is still important: different
trajectories can shift MM/GBSA results even when the protocol is unchanged.

## Main Caveats

The current scientific caveats include:

- one selected pose is carried through the workflow,
- the workflow assumes the Boltz/Vina tree is internally consistent,
- the MD protocol is short and screening-oriented,
- MM/GBSA remains an endpoint approximation,
- outputs are most reliable for relative comparison inside a matched series.

## Reading Order For Completed Runs

For a fast scientific review of one completed run, read:

1. `selected_model.json`
2. `pose_validation.json`
3. `prep_summary.json`
4. `md_summary.json`
5. `analysis/trajectory/trajectory_summary.json`
6. `analysis/mmgbsa/FINAL_RESULTS_MMPBSA.dat`
7. `run_summary.json`
