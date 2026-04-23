# Web API Reference

This document summarizes the main API routes implemented under `boltz-ui/app/api/`
that support run submission, queue inspection, analysis, and artifact access.

## Primary Run Routes

### `POST /api/runs/new`

Creates a single queued run.

Current behavior:

- accepts UI config, YAML path/name, YAML content, and run parameters
- generates a unique run id
- creates a results directory under the repository root
- writes the submitted YAML into that directory
- writes `params.json` and `ui-config.json`
- builds the underlying `boltz predict` command
- inserts the run into SQLite with status `queued`
- attempts to start execution immediately if the queue is idle

### `GET /api/runs`

Returns all runs ordered by descending creation time.

### `PATCH /api/runs/[id]`

Updates batch metadata for an existing run.

The current implementation supports updating:

- `batchTitle`
- `batchDescription`

### `DELETE /api/runs/[id]`

Deletes a run record and attempts to remove its associated results directory.

## Batch Route

### `POST /api/runs/batch`

Creates multiple queued runs from a single request.

Current safeguards and behaviors:

- maximum 100 jobs
- maximum 50 MB combined payload
- maximum 2 MB per YAML file
- path normalization to keep outputs under the repository root
- optional RCSB PDB download when a structure id is supplied
- one SQLite row inserted per expanded batch job

## Export And Artifact Routes

### `GET /api/run-history`

Returns repository-root `runhistory.csv` if present. If not present, it returns
a default CSV header row.

### `GET /api/runs/[id]/download`

Creates a zip archive of a run results directory and streams it back to the
caller.

### `GET /api/files/[...path]`

Serves result artifacts from the repository filesystem through the web app.

## Analysis Routes

### `POST /api/runs/[id]/rmsd`

Runs RMSD post-processing for a completed run. The route resolves the run's
results directory and then invokes `RMSD/batch_rmsd_to_json.py`.

### `POST /api/runs/[id]/tmscore`

Runs TM-score post-processing for a completed run by invoking
`RMSD/batch_tmscore_to_json.py`.

## Queue And Runtime Model

The API layer is not backed by a separate worker service. Instead:

- run metadata is stored in `boltz-ui/boltz-runs.db`
- queue control happens inside the web app process
- only one run is promoted to `running` at a time
- prediction execution is delegated to `../.venv/bin/python -m boltz.main`
- the UI runtime forces imports from `boltz-main/src`

## Persisted Run Fields

The `runs` table currently stores:

- `id`
- `status`
- `config`
- `command`
- `results`
- `log`
- `ligand`
- `target`
- `batchId`
- `batchTitle`
- `batchDescription`
- `cifUrl`
- `pdbUrl`
- `createdAt`
- `startedAt`
- `completedAt`

## Additional Route Families

The repository also contains additional route families for:

- auth under `/api/auth/*`
- run file helpers under `/api/runs/[id]/files`
- CIF and metrics helpers under `/api/runs/[id]/cifs` and `/api/run-metrics/[id]`
- batch and template helpers

Those routes are implementation details adjacent to the primary run lifecycle
documented above.
