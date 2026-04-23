# Setup And Install Guide

This guide covers the implemented local setup path for the Affinity Pipeline.
The current codebase is designed around native Linux execution with a real
Rosetta installation and Python 3.11+.

## Supported Baseline

- Ubuntu `22.04` `x86_64` is the qualified release baseline
- Ubuntu `24.04` may be acceptable for `research` mode, but it is not treated
  as the released baseline
- Python `3.11+`
- `gemmi`
- `rdkit`
- a real Rosetta build with:
  - `rosetta_scripts.default.linuxgccrelease`
  - `database/`
  - `source/scripts/python/public/molfile_to_params.py`

## Editable Install

For a simple local developer install:

```bash
python3 -m pip install -e .
```

This installs the Python package only. Real pipeline execution still requires
the Rosetta and scientific runtime dependencies described below.

## Native Research Bootstrap

The fastest supported bootstrap path is:

```bash
bash scripts/bootstrap_native_research.sh
```

By default this script:

1. installs `micromamba` if needed
2. creates a runtime environment via `scripts/create_runtime_env.sh`
3. builds a local wheelhouse via `scripts/build_wheelhouse.sh`
4. verifies the Python dependency set

You can optionally pass a Rosetta source tree, tarball, or clone URL:

```bash
bash scripts/bootstrap_native_research.sh /path/to/rosetta_source
```

Useful environment variables:

- `ENV_PREFIX`: target environment prefix
- `MAMBA_ROOT_PREFIX`: micromamba root
- `ROSETTA_PREFIX`: installed Rosetta prefix
- `INSTALL_APT=1`: install common Ubuntu packages first

## Create The Runtime Environment Manually

To create the Python environment without the full bootstrap:

```bash
bash scripts/create_runtime_env.sh
```

This script:

- creates a `micromamba` environment from `env/conda/base-linux-64.yml`
- upgrades `pip`, `wheel`, and `setuptools`
- installs runtime and dev requirements from `env/pip/`
- installs the repo itself in editable mode

## Install Rosetta

To build and stage Rosetta locally:

```bash
bash scripts/install_rosetta_from_bundle.sh /path/to/rosetta_source /path/to/install_prefix
```

The source argument may be:

- an existing Rosetta source tree
- a Git URL
- a tarball containing a Rosetta source tree

The installer:

- builds `rosetta_scripts.default.linuxgccrelease`
- stages a wrapper under `bin/`
- links the Rosetta `database/`
- links the Rosetta `source/` tree for ligand params generation

Useful environment variables:

- `ROSETTA_JOBS`: parallel build jobs
- `ROSETTA_TARGET`: Rosetta build target, defaults to `rosetta_scripts`

## Offline Runtime Install

For offline-capable runtime installation:

```bash
bash scripts/install_offline_runtime.sh
```

This path expects:

- `env/conda/base-linux-64.conda.lock`
- prebuilt wheels under `env/wheelhouse/runtime`

## Required Environment Variables For Real Runs

Integration tests and local real-toolchain runs expect:

- `REAL_ROSETTA_BIN`
- `REAL_ROSETTA_DB`
- `REAL_ROSETTA_SOURCE_ROOT`

Example:

```bash
export REAL_ROSETTA_BIN=/opt/rosetta/bin/rosetta_scripts.default.linuxgccrelease
export REAL_ROSETTA_DB=/opt/rosetta/database
export REAL_ROSETTA_SOURCE_ROOT=/opt/rosetta/source
```

## Qualify The Environment

To qualify a Rosetta environment for a runtime mode:

```bash
PYTHONPATH=src python3 -m affinity_pipeline.cli qualify-env \
  --rosetta-bin "$REAL_ROSETTA_BIN" \
  --rosetta-db "$REAL_ROSETTA_DB" \
  --mode research
```

The helper wrapper does the same and writes a Markdown verification report:

```bash
bash scripts/qualify_env_native.sh "$REAL_ROSETTA_BIN" "$REAL_ROSETTA_DB" research
```

The environment manifest is written to `work/environment_manifest.json`.

## Export Schemas

The pipeline can export its current JSON Schemas into `configs/schemas/`:

```bash
PYTHONPATH=src python3 -m affinity_pipeline.cli export-schemas
```

## Run Tests

The checked-in test entrypoint is:

```bash
make test
```

This runs the `unittest` suite under `tests/`.

Some tests require the real toolchain and are skipped unless the Rosetta
environment variables are set and the required scientific packages are present.

## Notes

- The implementation starts from completed Boltz outputs on disk; it does not
  install or run Boltz as part of the default setup path.
- The checked-in fixtures under `tests/fixtures/` are synthetic and intended
  for engineering verification, not scientific validation.
- Operational or regulated deployment requires the broader governance,
  validation, and licensing controls described elsewhere in `docs/`.
