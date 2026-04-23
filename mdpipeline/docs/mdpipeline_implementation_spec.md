# MDPipeline Implementation Specification

This document is the maintained implementation entrypoint for the current
`md_pipeline` codebase.

## Scope

The repository implements a single-run Boltz-to-OpenMM workflow for one
`boltz_results_*` directory at a time. Its purpose is to select one
protein-ligand pose, verify the pose assets, build a solvated simulation system,
run restrained molecular dynamics, analyze trajectory stability, and estimate
relative binding behavior with MM/GBSA.

This is:

- a post-processing and ranking workflow,
- a structured MD + MM/GBSA workflow for one selected pose,
- suitable for matched target/ligand or mutant-series comparisons.

This is not:

- a de novo docking engine,
- an absolute binding free-energy implementation,
- a fallback-heavy workflow that silently continues past broken invariants.

## Supported Workflow Contract

For one run directory, the implementation does the following:

1. Selects exactly one model from the Boltz/Vina batch.
2. Validates the selected CIF, receptor PDB, and ligand SDF against each other.
3. Prepares receptor and ligand artifacts compatible with Amber/OpenMM.
4. Builds a solvated, neutralized, salted protein-ligand system.
5. Runs minimization, restrained equilibration, and production MD in OpenMM.
6. Computes trajectory metrics with MDAnalysis.
7. Converts the prepared system for GROMACS/gmx_MMPBSA.
8. Runs GB and/or PB MM/GBSA, with optional decomposition.
9. Writes machine-readable stage summaries and one combined final summary.

## Input Contract

The run directory must contain a Boltz/Vina-style layout including:

- `predictions/<boltz_run>/<boltz_run>_model_*.cif`
- `vina_analysis/<boltz_run>/batch_results.tsv`
- `vina_analysis/<boltz_run>/batch_manifest.tsv`
- `vina_analysis/<boltz_run>/batch_summary.json`
- `vina_analysis/<boltz_run>/<boltz_run>_model_N/receptor.pdb`
- `vina_analysis/<boltz_run>/<boltz_run>_model_N/ligand.sdf`

The implementation is strict about this structure and expects it to be
internally consistent.

## Selection Rules

Selection is implemented in `md_pipeline.selection`.

Current behavior:

- reads the Vina batch result tables and summary metadata,
- selects the single unique minimum of
  `score_only_affinity_kcal_per_mol`,
- expects the usual Boltz batch completeness checks to pass,
- requires the chosen model to be runnable,
- errors if the best score is tied.

There is no alternate ranking heuristic when these assumptions fail.

## Pose Validation Rules

Pose validation is implemented in `md_pipeline.pose`.

The selected:

- Boltz CIF,
- receptor PDB,
- ligand SDF

must describe the same bound system.

Validation checks include:

- protein polymer presence in the CIF,
- target ligand identity,
- receptor atom-order and identity consistency,
- receptor coordinate agreement,
- ligand heavy-atom count and element agreement,
- ligand coordinate agreement,
- retained cofactor handling when present in the selected model metadata.

If these checks fail, the workflow stops.

## Preparation Contract

Preparation is implemented primarily in `md_pipeline.prep`.

Current preparation responsibilities:

- clean the receptor with AmberTools-friendly preprocessing,
- parameterize the ligand with either GAFF2 or OpenFF Sage,
- build Amber topology/coordinate artifacts,
- solvate the complex,
- neutralize and salt the system,
- record a machine-readable `prep_summary.json`.

Scientific defaults encoded in `md_pipeline.protocol.ProtocolConfig` include:

- protein force field: `ff14SB`
- ligand force field: `gaff2`
- water model: `TIP3P`
- water box: `TIP3PBOX`
- solvent padding: `10.0 A`
- ionic strength: `0.15 M`

## Protonation Support

Protonation support is implemented in `md_pipeline.protonation` and exposed
through CLI arguments.

Current supported modes:

- protein protonation: `none`, `propka-pdb2pqr`
- ligand protonation: `none`, `epik`

Current constraints:

- protein protonation defaults to pH `7.4`
- ligand protonation defaults to pH `7.4`
- `ligand_protonation_max_states` must remain exactly `1` when `epik` is used
  so the workflow stays deterministic
- protonation is opt-in; no fallback path is used if protonation is requested but
  the required tooling is absent or invalid

## MD Contract

Molecular dynamics is implemented in `md_pipeline.openmm_runner`.

Current default stage schedule:

1. `nvt_heavy` for `1.0 ns`
2. `npt_heavy` for `0.5 ns`
3. `npt_backbone_strong` for `0.75 ns`
4. `npt_backbone_light` for `0.75 ns`
5. `production` for `20.0 ns`

Core MD defaults:

- electrostatics: PME
- nonbonded cutoff: `1.0 nm`
- constraints: H-bonds
- timestep: `2 fs`
- friction: `1 ps^-1`
- NPT pressure: `1.0 bar`
- temperature: `300 K`

The production stage is the only stage that writes the main trajectory.

## MM/GBSA Contract

MM/GBSA is implemented in `md_pipeline.gmx_mmgbsa`.

Current supported behavior:

- solvent models: `gb`, `pb`, or `gb,pb`
- entropy modes: `none`, `c2`, `ie`
- optional per-residue decomposition
- optional pairwise decomposition
- optional MPI execution through `mpirun`

Current defaults:

- solvent model: `gb`
- entropy method: `none`
- MM/GBSA time window: skip the first `2.0 ns` of production by default
- frame interval: `1`

The implementation uses GROMACS + `gmx_MMPBSA` and does not silently degrade to
another endpoint method when those tools are unavailable.

## Final Output Contract

The final combined machine-readable output is:

- `run_summary.json`

It aggregates:

- selected-model metadata,
- pose-validation results,
- preparation summary,
- protonation summary pointers,
- MD outputs,
- trajectory-analysis outputs,
- MM/GBSA outputs.

Additional stage outputs are documented in
`docs/runtime_artifacts_reference.md`.

## Testing Surface

The repository includes focused tests for:

- CLI behavior
- protocol validation
- preparation and protonation behavior
- MM/GBSA configuration and output handling
- retained cofactors and structure normalization
- general utility functions

The maintained test entrypoint is:

```bash
python -m pytest tests
```

## Safety Model

The implementation is intentionally strict:

- invariants are enforced early,
- unsupported or inconsistent input structures stop execution,
- required tools must be resolvable for enabled features,
- deterministic protonation behavior is enforced where the code requires it,
- there are no fallback execution paths when required criteria are not met.
