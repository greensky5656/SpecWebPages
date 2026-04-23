# Design Specification

## Architecture Overview

The repository is a layered prediction platform with a web orchestration surface
on top of Boltz inference code.

The main layers are:

- UI layer: `boltz-ui/`
- web API layer: `boltz-ui/app/api/`
- queue and metadata layer: `boltz-ui/lib/db.ts`
- command-construction and execution layer: `boltz-ui/lib/boltz-client.ts`
- inference layer: `src/boltz/` and `boltz-main/src/boltz/`
- post-processing layer: `RMSD/` and Vina helpers under `boltz-ui/lib/vina/`

## Execution Model

### Direct CLI

Direct CLI runs are synchronous from the user's shell and use the installed
`boltz` console entrypoint.

### Web Queue

Web-submitted runs use a serialized queue:

1. API routes write a queued row into SQLite.
2. `startNextRunIfIdle()` acquires an immediate transaction.
3. If no run is `running`, the oldest queued run is promoted.
4. The process is spawned with the repository root as the working directory.
5. Logs are streamed back into SQLite and persisted to disk.
6. Completion transitions the run to `completed` or `failed`.
7. The queue immediately attempts to start the next job.

The design intentionally avoids concurrent active predictions through the web
surface.

## Runtime Import Strategy

The web runtime forces Python imports from `boltz-main/src` by prepending that
path to `PYTHONPATH` before spawning `python -m boltz.main`.

This design choice ensures that web-managed runs execute against the `boltz-main`
tree and not the alternative root `src/boltz` tree.

## Data Flow

### Input Phase

Inputs enter the platform as:

- CLI file paths or directories
- UI-authored YAML content
- batch-upload job definitions

The CLI validates supported extensions and processes YAML or FASTA inputs into a
manifest.

### Preprocessing Phase

The inference pipeline writes processed state beneath `processed/`, including:

- `records/`
- `structures/`
- `msa/`
- `constraints/`
- `templates/`
- `mols/`
- `manifest.json`

MSA generation is optional and only happens when missing protein MSAs are
allowed through `--use_msa_server`.

### Prediction Phase

The prediction layer:

- downloads weights and molecule assets if absent
- filters already computed outputs unless `--override` is set
- runs structure inference
- runs affinity inference for eligible records
- writes outputs beneath `predictions/`

### Post-Processing Phase

For UI-managed runs, successful execution may trigger:

- CIF discovery for viewer attachment
- RMSD JSON generation
- TM-score JSON generation
- Vina docking analysis
- run-history CSV append

## Storage Design

### Durable Run Metadata

SQLite stores:

- run id
- queue status
- raw config JSON
- stored command
- log output
- timestamps
- batch metadata
- discovered CIF/PDB URLs

### Durable Filesystem Artifacts

Each run root stores:

- saved YAML
- parameter snapshots
- processed intermediate files
- model prediction outputs
- optional analysis files
- log files

## API Design

The primary API design is route-oriented and server-local:

- creation routes enqueue work
- list and detail routes read from SQLite
- analysis routes derive paths from stored run config
- file routes serve artifacts from the local repository filesystem

There is no separate worker service, broker, or remote task queue in the
current implementation. Queue control is embedded in the web app process.

## Operational Invariants

The current design relies on these invariants:

- only one web-managed run may actively execute at a time
- run directories are named from the input stem plus a `boltz_results_` prefix
- UI-managed YAML inputs are written into their own results directories
- batch-created result paths must remain inside the repository root
- MSA authentication modes cannot be mixed
- a successful UI run should generally produce a discoverable CIF file

## Known Design Boundaries

- The repository contains both `src/boltz` and `boltz-main/src/boltz`.
- The UI queue is process-local and SQLite-backed rather than distributed.
- Root-level markdown files include both current docs and historical notes.
- Training, validation, and exploratory science workflows exist, but they are
  not all first-class paths in the UI runtime.

## Testing Surface

The main automated test surfaces referenced by config are:

- backend tests under `tests/`
- UI tests under `boltz-ui/tests/`

The design assumes that documentation and operational behavior should be
verified against these source trees rather than against archival root notes.
