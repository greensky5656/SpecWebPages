# Boltzbase Platform Implementation Specification

## Scope

The current repository implements a combined prediction platform with two main
surfaces:

- a Python CLI exposed as `boltz`
- a Next.js web application (`boltz-ui`) for managing queued prediction runs

The platform is centered on Boltz structure prediction, with optional Boltz-2
affinity prediction and several web-managed post-processing steps.

## Product Surfaces

### Direct CLI Surface

The root `pyproject.toml` publishes the `boltz` console script and installs the
root `src/boltz` package. The primary implemented command is:

- `boltz predict`

This path is the direct local inference interface.

### Web Surface

The Affinitas UI in `boltz-ui/` provides:

- run configuration and submission
- queued execution state
- run history
- downloadable results
- CIF-backed structure viewing
- optional batch submission

The UI persists queue metadata in `boltz-ui/boltz-runs.db`.

## Implemented Runtime Boundary

The repository contains two in-repo Boltz source trees:

- `src/boltz`
- `boltz-main/src/boltz`

The current web runtime explicitly uses `boltz-main/src/boltz` by:

1. selecting `../.venv/bin/python`
2. setting `PYTHONPATH` to `boltz-main/src`
3. invoking `python -m boltz.main`

This is an important implementation boundary. Direct CLI usage and web-managed
usage do not necessarily import the same in-repo source tree unless the local
environment is arranged to do so deliberately.

## Prediction Capabilities

From the currently inspected code and docs, the implemented platform supports:

- YAML and FASTA inputs for prediction
- directory inputs containing supported files
- Boltz-1 and Boltz-2 structure prediction
- optional MSA generation with `--use_msa_server`
- optional constraints and templates from YAML input
- optional affinity prediction for Boltz-2 when the manifest requests affinity
- optional method conditioning for Boltz-2
- optional affinity molecular-weight correction

The detailed input schema is documented in `docs/prediction.md`.

## Web Workflow Contract

For a single UI-submitted run, the implemented behavior is:

1. receive config and YAML content through `POST /api/runs/new`
2. generate a run id
3. create `boltz_results_<yaml-name-with-run-id>/`
4. write the YAML into that directory
5. write `params.json` and, for single runs, `ui-config.json`
6. create a queued row in `boltz-ui/boltz-runs.db`
7. start execution if no other run is currently marked `running`
8. stream command output into the database and log files
9. mark the run `completed` or `failed`
10. on success, attempt CIF discovery and optional post-processing

## Batch Workflow Contract

For `POST /api/runs/batch`, the current implementation additionally:

- accepts up to 100 jobs
- rejects payloads above 50 MB
- rejects individual YAML files above 2 MB
- sanitizes the result path to stay under the repository root
- optionally fetches one or more RCSB PDB files for each job
- inserts one queued row per expanded job

## Post-Processing Contract

After a successful web-managed run, the platform can attempt:

- RMSD generation through `RMSD/batch_rmsd_to_json.py`
- TM-score generation through `RMSD/batch_tmscore_to_json.py`
- Vina docking analysis through `boltz-ui/lib/vina/server`
- run-history CSV aggregation at repository-root `runhistory.csv`

These are best-effort add-ons around the core prediction path. They do not
replace the primary Boltz success criteria.

## Storage Contract

The current platform writes to these main storage locations:

- repository-root results directories: `boltz_results_*`
- repository-root `runhistory.csv`
- repository-root `logs/` for UI-generated logs
- `boltz-ui/boltz-runs.db` for queue metadata
- cache directory resolved from `BOLTZ_CACHE` or `~/.boltz`

## Queue State Contract

The UI queue uses these primary run states:

- `queued`
- `running`
- `completed`
- `failed`

Only one run is allowed to hold the `running` state at a time in the current
SQLite-backed execution loop.

## Non-Goals Of This Spec

This specification does not claim:

- that training docs are fully up to date for Boltz-2
- that every root-level markdown note is part of the maintained product surface
- that Vina, PSI4, or custom validation workspaces are fully configured by
  default in a fresh install

Those areas exist in the repository, but they are adjacent to the primary
prediction platform rather than being guaranteed turnkey paths.
