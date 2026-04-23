# Requirements

This document records the current implemented requirements for the
Boltz-to-Meeko-to-AutoDock Vina workflow in this repository in a stricter,
review-friendly format. It describes the maintained docking path as it exists
now rather than a hypothetical future packaged platform.

## Intended use

This repository is intended to dock completed Boltz structure outputs through a
Meeko → AutoDock Vina workflow, including the maintained minimized-receptor path
that performs protein preparation and restrained OpenMM minimization before
scoring.

## System requirements

- Linux `x86_64` as the expected operating environment for the maintained setup
  path.
- `micromamba` available on `PATH` for the documented environment bootstrap.
- A maintained environment including Python 3.11, AmberTools, `pdbfixer`,
  OpenMM, RDKit, Meeko 0.7.1, and Gemmi.
- A working AutoDock Vina binary, either from the default repository-local
  `vina_1.2.7_linux_x86_64` path or through an explicit `VINA=/path/to/vina`
  override.

## Input requirements

- Completed Boltz run directories already present on disk.
- For the direct CLI path, a specific Boltz run folder supplied as input to the
  scoring script.
- Receptor and ligand artifacts compatible with the maintained Meeko/Vina
  preparation and docking path.
- Inputs compatible with the current script-driven repository layout rather than
  a packaged multi-command CLI contract.

## Functional requirements

- The repository must support the maintained minimized Boltz → Meeko → Vina path
  documented in `boltz_run_minimized_vina_spec.md`.
- It must prepare the selected structure inputs, perform Meeko/Vina docking, and
  write run outputs under the configured output root.
- Wrapper scripts must support overriding the environment path, Vina binary, and
  output root through environment variables.
- The maintained path must emit per-run logs and the tabular summary files
  described in the repository README.

## Output requirements

- Per-run outputs under the configured output root, currently centered on
  `vina_boltz_minimized/<boltz_run_name>/...`.
- A run log for each processed run.
- Summary outputs including `per_cif_affinities.tsv` and
  `run_affinity_summary.tsv` for the maintained workflow path.

## Safety and fail-closed requirements

- The maintained path is explicit about the environment and Vina binary it uses;
  missing tooling should fail the run rather than silently choose an
  unqualified substitute.
- The repository should be treated as a docking/minimized-scoring workflow and
  not silently promoted to a stronger thermodynamic evidence class.
- The current codebase does not require or expose a polished installed package /
  CLI surface comparable to the other reviewed repos.

## Non-requirements / out-of-scope behavior

- No regulated or benchmark-facing reporting layer is required by the current
  repository.
- No native web application, release bundle, or signed-report surface is part of
  the current implemented contract.
- No claim of rigorous absolute affinity prediction is required or supported by
  this docking workflow alone.
