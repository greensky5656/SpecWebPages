# Runtime And Artifacts Reference

This document describes the output structure written by the current
`md_pipeline` implementation.

## Default Output Root

Unless `--output-root` is provided, the CLI writes to:

```text
<run_dir>/openmm_md/<boltz_run>/<model_name>/
```

## Top-Level Output Files

The workflow writes the following top-level files as stage execution proceeds:

- `selected_model.json`
- `tool_config.json`
- `protocol.json`
- `pose_validation.json`
- `prep_summary.json`
- `md_summary.json`
- `run_summary.json`
- `pipeline.log`

## Top-Level Directories

The current implementation also writes stage-specific directories such as:

- `prep/`
- `md/`
- `analysis/trajectory/`
- `analysis/mmgbsa/`

## `selected_model.json`

Selection output recorded immediately after the best model is chosen.

Purpose:

- identifies the selected Boltz run and model,
- records the selected CIF/model directory paths,
- captures ligand and retained-cofactor metadata needed by later stages.

## `tool_config.json`

Resolved external tool configuration used for the run.

Purpose:

- records the Python executable and tool `bin/` directory,
- captures the effective commands for AmberTools, GROMACS, MM/GBSA, MPI, and
  optional protonation tools.

## `protocol.json`

Serialized `ProtocolConfig` for the run.

Purpose:

- records scientific defaults and user overrides,
- captures the effective stage schedule and MM/GBSA configuration,
- makes the run configuration inspectable even after execution finishes.

## `pose_validation.json`

Pose-validation report written after selection.

Purpose:

- records whether the chosen CIF, receptor PDB, and ligand SDF agree,
- captures ligand and receptor validation details,
- preserves fail-fast structural checks in machine-readable form.

## `prep/`

Preparation artifacts written by `md_pipeline.prep`.

Common artifacts include:

- `receptor_source.pdb`
- `receptor_pdb4amber.pdb`
- `receptor_prepareforleap.pdb`
- `ligand_source.sdf`
- `ligand_gaff2.mol2` when GAFF2 is used
- `ligand.frcmod` when GAFF2 is used
- `complex_reference.pdb`
- `complex_unsalted.prmtop`
- `complex_unsalted.inpcrd`
- `complex.prmtop`
- `complex.inpcrd`
- `complex_solvated.pdb`

Optional protonation-related artifacts may also appear here or be referenced by
`prep_summary.json`.

## `prep_summary.json`

Preparation summary for the built system.

Purpose:

- records the preparation working directory,
- points to the prepared topology and coordinate files,
- records ligand charge and ion-pair count,
- records protonation-related artifact paths when protonation is enabled.

This file is the reuse contract for later analysis workflows.

## Protonation Artifacts

When protonation workflows are enabled, `prep_summary.json` and
`run_summary.json` may point to additional files such as:

- protein protonation summaries
- flagged-residue TSV outputs
- PDB2PQR-generated files
- PROPKA pKa outputs
- Epik-selected ligand files
- ligand protonation summary JSON

The exact set depends on the enabled protonation method.

## `md/`

Molecular-dynamics artifacts written by `md_pipeline.openmm_runner`.

Common artifacts include:

- stage PDB snapshots
- stage CSV state reports
- `production.dcd`
- `production.xtc`
- `production.csv`
- `equilibrated.pdb`
- `final.pdb`
- `final_state.xml`
- `system.xml`

## `md_summary.json`

MD summary output.

Purpose:

- records the MD directory,
- points to the final trajectory and structure artifacts,
- records the selected OpenMM platform,
- captures each stage result, including duration, restraint mode, and output
  paths.

This file is required when reusing MD outputs through the `analyze` command.

## `analysis/trajectory/`

Trajectory-analysis outputs written by `md_pipeline.analysis`.

Common artifacts:

- `trajectory_metrics.csv`
- `trajectory_summary.json`

The summary provides a compact view of RMSD, contact, distance, and hydrogen
bond metrics computed from the production trajectory.

## `analysis/mmgbsa/`

MM/GBSA outputs written by `md_pipeline.gmx_mmgbsa`.

Common artifacts include:

- `complex.gro`
- `topol.top`
- `analysis.tpr`
- `index.ndx`
- `production_pbc.xtc`
- `production_fit.xtc`
- `mmpbsa.in`
- `FINAL_RESULTS_MMPBSA.dat`
- `FINAL_RESULTS_MMPBSA.csv`
- `mmgbsa_summary.json`

Optional decomposition outputs:

- `FINAL_DECOMP_MMPBSA_PER_RESIDUE.dat`
- `FINAL_DECOMP_MMPBSA_PER_RESIDUE.csv`
- `FINAL_DECOMP_MMPBSA_PAIRWISE.dat`
- `FINAL_DECOMP_MMPBSA_PAIRWISE.csv`

## `run_summary.json`

Combined final summary assembled by the CLI.

Purpose:

- aggregates the selected-model output,
- aggregates pose-validation results,
- aggregates preparation and protonation outputs,
- aggregates MD results,
- aggregates trajectory-analysis outputs,
- aggregates MM/GBSA outputs.

This is the primary machine-readable output for a completed run.

## Reuse Contract For `analyze`

The `analyze` command does not rebuild the system or rerun MD. It loads:

- `prep_summary.json`
- `md_summary.json`

from the chosen output root and expects both files to contain valid path and
summary data. Missing or malformed summary files stop execution.

## Logging

`pipeline.log` is the main human-readable execution log for the workflow.

It should be consulted together with the machine-readable JSON summaries when a
run fails or produces unexpected scientific results.
