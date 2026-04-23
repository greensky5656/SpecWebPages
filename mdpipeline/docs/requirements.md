# Requirements

This document records the current implemented requirements for the MDPipeline
repository in a stricter, review-friendly format. It is source-grounded and is
meant to describe the present shipped workflow rather than aspirational future
scope.

## Intended use

MDPipeline is intended for single-run, single-selected-pose downstream molecular
simulation and MM/GBSA analysis of a completed `boltz_results_*` directory. Its
current role is comparative ranking support rather than direct rigorous absolute
thermodynamic affinity determination.

## System requirements

- A Linux-oriented scientific Python environment capable of running the
  repository as `python -m md_pipeline` from the checkout.
- Python dependencies consistent with the maintained environment definitions,
  especially `environment.yml` and `amber311_export.yml`.
- OpenMM, AmberTools-compatible preparation tooling, RDKit, Gemmi, ParmEd, and
  MDAnalysis in the active runtime.
- A working GROMACS installation and `gmx_MMPBSA` for MM/GBSA execution.
- Optional protonation tools only when the corresponding branches are enabled:
  `propka3`, `pdb2pqr`, `epik`, and `structconvert`.

## Input requirements

- One `boltz_results_*` run directory at a time.
- The expected Boltz/Vina batch layout, including `predictions/`,
  `vina_analysis/`, batch summary tables, and the per-model receptor/ligand
  artifacts used for pose validation.
- A selected CIF, receptor PDB, and ligand SDF that all describe the same bound
  pose.
- Inputs compatible with the current single-selected-pose workflow rather than a
  multi-pose ensemble contract.

## Functional requirements

- The pipeline must select a single best model from the reviewed Boltz/Vina
  batch outputs.
- It must validate pose-asset consistency before preparation proceeds.
- It must prepare a receptor-ligand system compatible with downstream OpenMM and
  GROMACS/gmx_MMPBSA stages.
- It must run restrained minimization/equilibration and production MD in
  OpenMM.
- It must perform trajectory analysis and emit structured stage summaries.
- When MM/GBSA is enabled, it must run GB and/or PB analysis through the
  configured GROMACS + `gmx_MMPBSA` path.

## Output requirements

- A structured output tree rooted under the current run's OpenMM/MD output path.
- Machine-readable summaries including `selected_model.json`, `tool_config.json`,
  `protocol.json`, `pose_validation.json`, `prep_summary.json`, `md_summary.json`,
  trajectory-analysis outputs, MM/GBSA summaries when enabled, and the combined
  `run_summary.json`.
- A run log capturing pipeline execution.

## Safety and fail-closed requirements

- Missing required files, broken invariants, or incompatible external tools must
  stop the run rather than trigger silent fallback behavior.
- Optional protonation branches remain opt-in and must validate their toolchain
  before use.
- The shipped CLI must not silently widen into benchmark/cohort orchestration
  when the required supporting surfaces are absent.

## Non-requirements / out-of-scope behavior

- No native benchmark/cohort command layer is required by the current shipped
  CLI.
- No automatic fallback to alternate pose sources, alternate preparation paths,
  or degraded MM/GBSA execution is supported.
- No direct claim of rigorous absolute thermodynamic affinity estimation is part
  of the current implemented contract.
