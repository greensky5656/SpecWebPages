# CLI Reference

The main entrypoint is:

```bash
python -m md_pipeline <command> [options]
```

If `<command>` is omitted, the default command is `full`.

## Commands

### `check-openmm`

Validates that OpenMM imports successfully, the requested platform can be
selected, and a minimal OpenMM `Context` can be created.

Example:

```bash
python -m md_pipeline check-openmm --platform CUDA
```

### `select`

Reads the Boltz/Vina batch outputs, selects the best model, and writes:

- `selected_model.json`
- `tool_config.json`
- `protocol.json`

Example:

```bash
python -m md_pipeline select --run-dir /path/to/boltz_results_example
```

### `prepare`

Runs selection, pose validation, and system preparation.

Additional stage output:

- `pose_validation.json`
- `prep_summary.json`

Example:

```bash
python -m md_pipeline prepare --run-dir /path/to/boltz_results_example
```

### `simulate`

Runs `prepare` and then OpenMM minimization + MD stages.

Additional stage output:

- `md_summary.json`

Example:

```bash
python -m md_pipeline simulate --run-dir /path/to/boltz_results_example
```

### `analyze`

Reuses existing `prep_summary.json` and `md_summary.json` from the output root,
then runs:

- trajectory analysis
- MM/GBSA
- final summary assembly

Example:

```bash
python -m md_pipeline analyze --run-dir /path/to/boltz_results_example
```

### `full`

Runs the complete workflow end to end.

Example:

```bash
python -m md_pipeline full --run-dir /path/to/boltz_results_example
```

## Core Arguments

### `--run-dir`

Path to the `boltz_results_*` directory. Required for all commands except
`check-openmm`.

### `--output-root`

Override the default output root.

Default output root when omitted:

```text
<run_dir>/openmm_md/<boltz_run>/<model_name>/
```

### `--python-executable`

Override the Python executable used when inferring tool locations.

### `--tool-bin-dir`

Explicit tool `bin/` directory for AmberTools, GROMACS, `gmx_MMPBSA`, and
related external tools.

### `--cpu-threads`

CPU thread count for CPU-bound tools and the OpenMM CPU platform.

### `--verbose`

Enables more detailed console and log output.

## External Tool Overrides

These options allow per-tool resolution overrides:

- `--gmx-command`
- `--gmx-mmgbsa-command`
- `--mpirun-command`
- `--propka3-command`
- `--pdb2pqr-command`
- `--epik-command`
- `--structconvert-command`

## Ligand Parameterization Options

### `--ligand-parameterization-backend`

Allowed values:

- `gaff2`
- `openff-sage`

Default:

- `gaff2`

### `--openff-forcefield`

OpenFF force-field identifier used when `openff-sage` is selected.

Default:

- `openff_unconstrained-2.2.1.offxml`

## Protonation Options

### Protein Protonation

- `--protein-protonation-method`: `none` or `propka-pdb2pqr`
- `--protein-protonation-ph`: default `7.4`
- `--protein-protonation-flag-window`: default `2.0`
- `--protein-protonation-pdb2pqr-chain-flag`: `keep-chain` or `chain`

### Ligand Protonation

- `--ligand-protonation-method`: `none` or `epik`
- `--ligand-protonation-ph`: default `7.4`
- `--ligand-protonation-pht`: default `2.0`
- `--ligand-protonation-max-states`: default `1`

Current deterministic constraint:

- `--ligand-protonation-max-states` must remain `1` when using Epik

## OpenMM Platform Options

- `--platform`: `auto`, `CUDA`, `OpenCL`, or `CPU`
- `--cuda-precision`: `single`, `mixed`, or `double`
- `--cuda-device-index`: default `0`

## MM/GBSA Options

### Core MM/GBSA Configuration

- `--entropy-method`: `none`, `c2`, or `ie`
- `--mmgbsa-solvent-models`: `gb`, `pb`, or `gb,pb`
- `--mmgbsa-interval`
- `--mmgbsa-mpi-procs`

### Time-Window Controls

Time-based mode is the default:

- `--mmgbsa-skip-initial-ns` defaults to `2.0`
- `--mmgbsa-end-ns` defaults to the end of production

Frame-based mode is enabled with:

- `--no-mmgbsa-skip-initial-ns`
- `--mmgbsa-startframe`
- `--mmgbsa-endframe`

### Decomposition Controls

- `--mmgbsa-run-per-residue-decomp`
- `--mmgbsa-run-pairwise-decomp`
- `--mmgbsa-per-residue-idecomp`
- `--mmgbsa-pairwise-idecomp`
- `--mmgbsa-print-res-per-residue`
- `--mmgbsa-print-res-pairwise`
- `--mmgbsa-dec-verbose`

## System-Building And Thermodynamic Parameters

- `--solvent-padding-angstrom` default `10.0`
- `--ionic-strength-molar` default `0.15`
- `--temperature-k` default `300.0`
- `--pressure-bar` default `1.0`
- `--prep-random-seed`
- `--md-random-seed`

## MD Schedule Parameters

- `--production-frame-interval-ps` default `10.0`
- `--nvt-heating-ns` default `1.0`
- `--npt-heavy-ns` default `0.5`
- `--npt-backbone-strong-ns` default `0.75`
- `--npt-backbone-light-ns` default `0.75`
- `--production-ns` default `20.0`

## Restraint Parameters

- `--nvt-restraint-k` default `1000.0`
- `--npt-heavy-restraint-k` default `250.0`
- `--npt-backbone-strong-k` default `100.0`
- `--npt-backbone-light-k` default `50.0`

## Common Usage Patterns

### Minimal End-To-End Run

```bash
python -m md_pipeline full \
  --run-dir /path/to/boltz_results_example
```

### CUDA Run

```bash
python -m md_pipeline full \
  --run-dir /path/to/boltz_results_example \
  --platform CUDA \
  --cuda-precision double \
  --cuda-device-index 0
```

### Reuse Existing Prep And MD For Analysis

```bash
python -m md_pipeline analyze \
  --run-dir /path/to/boltz_results_example
```

### Enable Protein Protonation

```bash
python -m md_pipeline full \
  --run-dir /path/to/boltz_results_example \
  --protein-protonation-method propka-pdb2pqr
```

### Enable Ligand Protonation

```bash
python -m md_pipeline full \
  --run-dir /path/to/boltz_results_example \
  --ligand-protonation-method epik
```

### Run GB And PB MM/GBSA

```bash
python -m md_pipeline full \
  --run-dir /path/to/boltz_results_example \
  --mmgbsa-solvent-models gb,pb
```
