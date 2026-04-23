# Setup And Install Guide

This guide describes how to set up the runtime environment for `md_pipeline`,
how external tools are discovered, and how to run a verified first workflow.

## Repository Execution Model

This repository is currently run from a source checkout with:

```bash
python -m md_pipeline <command> ...
```

There is no project packaging file in the repository root at present, so the
supported execution model is:

1. clone the repository,
2. activate the environment,
3. run commands from the repository root.

## Primary Environment

The main environment definition is `environment.yml`.

Create and activate it with:

```bash
mamba env create -f environment.yml -n mdpipe
mamba activate mdpipe
```

The declared stack includes:

- Python `>=3.10,<3.13`
- AmberTools
- GROMACS
- OpenMM
- Gemmi
- RDKit
- ParmEd
- MDAnalysis
- PDBFixer

## `gmx_MMPBSA` Installation Note

`environment.yml` explicitly notes that `gmx_MMPBSA` may not be installable in
the same environment as the main conda-forge stack.

Supported approaches:

### Option A: Separate Tool Environment

Install `gmx_MMPBSA` in another environment and point the pipeline at that
environment's `bin/` directory:

```bash
python -m md_pipeline full \
  --run-dir /path/to/boltz_results_example \
  --tool-bin-dir /path/to/other/env/bin
```

### Option B: Explicit Tool Paths Or PATH Management

If the relevant executables are already on `PATH`, or you want to override them
individually, use flags such as:

- `--gmx-command`
- `--gmx-mmgbsa-command`
- `--mpirun-command`
- `--propka3-command`
- `--pdb2pqr-command`
- `--epik-command`
- `--structconvert-command`

## External Tool Requirements

The current implementation may call the following external tools depending on
which options are enabled:

- `pdb4amber`
- `cpptraj`
- `tleap`
- `antechamber`
- `parmchk2`
- `gmx`
- `gmx_MMPBSA`
- `mpirun` when MM/GBSA MPI mode is requested
- `propka3` and `pdb2pqr` for protein protonation
- `epik` and `structconvert` for ligand protonation

If protonation is left at its defaults, the protonation-specific tools are not
required for a basic run.

## OpenMM Preflight

Before running production work, validate the OpenMM installation and platform:

```bash
python -m md_pipeline check-openmm --platform auto
```

Useful variants:

```bash
python -m md_pipeline check-openmm --platform CUDA
python -m md_pipeline check-openmm --platform CPU
```

The preflight verifies:

- `openmm` imports successfully,
- the requested platform can be selected,
- a minimal OpenMM `Context` can be created on that platform.

## Input Requirements

Each run targets one `boltz_results_*` directory with the expected layout:

- `predictions/<boltz_run>/<boltz_run>_model_*.cif`
- `vina_analysis/<boltz_run>/batch_results.tsv`
- `vina_analysis/<boltz_run>/batch_manifest.tsv`
- `vina_analysis/<boltz_run>/batch_summary.json`
- `vina_analysis/<boltz_run>/<boltz_run>_model_N/receptor.pdb`
- `vina_analysis/<boltz_run>/<boltz_run>_model_N/ligand.sdf`

The implementation is strict about this layout and expects the Boltz/Vina batch
selection metadata to be internally consistent.

## Quick Start

Run the full workflow from the repository root:

```bash
python -m md_pipeline full \
  --run-dir /path/to/boltz_results_example \
  --platform auto \
  --verbose
```

If you need to point at a different output directory root:

```bash
python -m md_pipeline full \
  --run-dir /path/to/boltz_results_example \
  --output-root /path/to/output_dir
```

By default the pipeline writes under:

```text
<run_dir>/openmm_md/<boltz_run>/<model_name>/
```

## Quick Start With Optional Protonation

Protein protonation:

```bash
python -m md_pipeline full \
  --run-dir /path/to/boltz_results_example \
  --protein-protonation-method propka-pdb2pqr \
  --protein-protonation-ph 7.4
```

Ligand protonation:

```bash
python -m md_pipeline full \
  --run-dir /path/to/boltz_results_example \
  --ligand-protonation-method epik \
  --ligand-protonation-ph 7.4 \
  --ligand-protonation-pht 2.0
```

These workflows require their corresponding external tools to be resolvable.

## Quick Start With MM/GBSA Variants

GB only:

```bash
python -m md_pipeline full \
  --run-dir /path/to/boltz_results_example \
  --mmgbsa-solvent-models gb
```

GB and PB:

```bash
python -m md_pipeline full \
  --run-dir /path/to/boltz_results_example \
  --mmgbsa-solvent-models gb,pb
```

Per-residue decomposition:

```bash
python -m md_pipeline full \
  --run-dir /path/to/boltz_results_example \
  --mmgbsa-run-per-residue-decomp
```

Pairwise decomposition:

```bash
python -m md_pipeline full \
  --run-dir /path/to/boltz_results_example \
  --mmgbsa-run-pairwise-decomp
```

## Development And Test Verification

Run the repository test suite with:

```bash
python -m pytest tests
```

If your environment differs substantially from `environment.yml`, validate both
`check-openmm` and at least one representative `full` or staged run before
trusting scientific outputs.
