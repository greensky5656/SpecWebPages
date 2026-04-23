# Setup And Install Guide

This guide reflects the current repository layout and runtime behavior.

## Modes

There are two practical setup targets:

- `research_mode`: local install for CLI use, tests, and deterministic dry runs
- `production_mode`: local install plus the external scientific toolchain used by
  the configured workflow path

If you only want to exercise the pipeline, tests, and report generation, start
with `research_mode`.

## Baseline Requirements

- Linux or another Unix-like environment
- `python3` 3.11+
- `pip`
- `bash`

The Python package metadata currently declares a small runtime dependency set and
installs the CLI entrypoint `mutation-rbfe`.

## Install For Research Mode

Create and activate a virtual environment:

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install --upgrade pip
```

Install the repository in editable mode:

```bash
python3 -m pip install -e .
```

Confirm the CLI is available:

```bash
mutation-rbfe --help
```

Run the test suite:

```bash
python3 -m pytest tests
```

## Research-Mode Quick Start

The default research config is:

- `configs/openfe/mutation_rbfe_protocol.yaml`

Run a dry-run pipeline execution:

```bash
mutation-rbfe run \
  --wt-run /path/to/boltz_results_wt \
  --mut-run /path/to/boltz_results_mut \
  --config configs/openfe/mutation_rbfe_protocol.yaml \
  --out /path/to/output_dir
```

In `research_mode`, the code does not run real PMX/GROMACS mutation legs. It
generates deterministic synthetic leg values while still exercising intake,
mapping, inference, calibration, QC, reporting, signing, and artifact output.

## Production-Mode Requirements

The production config is:

- `configs/openfe/mutation_rbfe_protocol.production.yaml`

Current code-level requirements for `production_mode` include:

- `pmx` available on `PATH`
- `gmx` available on `PATH`
- `execution.workflow_script` must exist
- `execution.openfe_version` must be pinned
- `security.signing_key` must be non-default
- at least `3` replicates per run

Production mode delegates leg execution through the configured workflow script,
which is currently:

- `scripts/pmx_gmx_workflow.py`

If your production workflow template calls `scripts/pmx_real_compute_leg.py`,
the surrounding environment will also need the external scientific stack that
script expects, including PMX, GROMACS, AmberTools, RDKit, and ParmEd.

## Production-Mode Example

```bash
mutation-rbfe run \
  --wt-run /path/to/boltz_results_wt \
  --mut-run /path/to/boltz_results_mut \
  --config configs/openfe/mutation_rbfe_protocol.production.yaml \
  --out /path/to/output_dir
```

## Other CLI Commands

Validate PLATINUM holdout metrics:

```bash
mutation-rbfe validate-platinum \
  --dataset configs/openfe/platinum_holdout_example.json \
  --qc-gates configs/openfe/mutation_qc_gates.yaml \
  --out /path/to/validation_summary.json
```

Monitor report populations:

```bash
mutation-rbfe monitor \
  --report-dir /path/to/report/root \
  --out /path/to/monitor_summary.json
```

Verify a signed report:

```bash
mutation-rbfe verify-report \
  --report /path/to/mutation_affinity_report.json \
  --signature /path/to/mutation_affinity_report.signature.json \
  --key your-signing-key
```

Build a release bundle:

```bash
mutation-rbfe build-release-bundle \
  --repo-root /path/to/OpenFE \
  --bundle-dir /path/to/release_bundle \
  --validation-summary /path/to/validation_summary.json
```

## Input Expectations

The pipeline expects WT and MUT inputs that look like Boltz run roots with:

- exactly one run YAML at the root
- prediction/model artifacts under `predictions/<run_name>/`
- selected model ligand and receptor artifacts under
  `vina_analysis/<run_name>/<run_name>_model_<idx>/`
- optional `receptor_with_cofactors.pdb` artifacts when cofactor-inclusive
  selection is enabled

Current supported mutation scope:

- one or more substitutions on the same chain

Unsupported cases fail closed to `NO_CALL` rather than following a fallback path.

## Where To Go Next

- Design and implementation: `docs/design_specification.md`
- CLI usage: `docs/cli_reference.md`
- Runtime outputs: `docs/runtime_artifacts_reference.md`
- Pipeline diagram: `docs/pipeline_flowchart.md` and `docs/pipeline_flowchart.svg`
- Schemas: `specs/schemas/boltz_mutation_pair_intake.schema.json` and
  `specs/schemas/mutation_affinity_report.schema.json`
- Configs: `configs/openfe/mutation_rbfe_protocol.yaml`,
  `configs/openfe/mutation_rbfe_protocol.production.yaml`, and
  `configs/openfe/mutation_qc_gates.yaml`
