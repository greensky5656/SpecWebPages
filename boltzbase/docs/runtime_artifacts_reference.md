# Runtime And Artifacts Reference

This document describes the main directories and files produced by the current
CLI and web workflows.

## Run Root Naming

For an input named `example.yaml` and `--out_dir ./results`, the runtime writes
to:

```text
./results/boltz_results_example/
```

For UI-managed runs, the saved YAML filename usually includes the generated run
id, so the run root becomes:

```text
<out_dir>/boltz_results_<yaml-base>-<run-id>/
```

## Typical Run Layout

```text
boltz_results_<name>/
├── <name>.yaml
├── params.json
├── ui-config.json
├── <run-id>.log
├── processed/
│   ├── manifest.json
│   ├── records/
│   ├── structures/
│   ├── msa/
│   ├── constraints/
│   ├── templates/
│   └── mols/
├── msa/
└── predictions/
    ├── <name>/
    │   ├── <name>_model_0.cif
    │   ├── confidence_<name>_model_0.json
    │   ├── affinity_<name>.json
    │   ├── pae_<name>_model_0.npz
    │   ├── pde_<name>_model_0.npz
    │   └── plddt_<name>_model_0.npz
    ├── rmsd_results.json
    └── tmscore_results.json
```

Not every file appears for every run. For example:

- `ui-config.json` is a UI artifact, not a direct CLI artifact
- `affinity_<name>.json` only appears when affinity is requested
- `rmsd_results.json` and `tmscore_results.json` are post-processing outputs

## Processed Artifacts

The `processed/` tree is the normalized input state used by prediction:

- `manifest.json`: processed manifest of records
- `records/*.json`: serialized record metadata
- `structures/*.npz`: processed structure arrays
- `msa/*.npz`: processed MSA arrays
- `constraints/*.npz`: processed constraints
- `templates/*.npz`: processed structural templates
- `mols/*.pkl`: extra molecule payloads

## Prediction Artifacts

The `predictions/<name>/` tree contains:

- predicted structures in `.cif` or `.pdb`, depending on output format
- confidence summaries in `confidence_*.json`
- optional affinity outputs in `affinity_<name>.json`
- optional dense score arrays such as PAE and PDE
- optional embeddings when `--write_embeddings` is enabled

## UI Queue Artifacts

The web application persists queue state in:

- `boltz-ui/boltz-runs.db`

The `runs` table stores run status, command text, timestamps, logs, and
discovered artifact URLs.

## Log Artifacts

UI-managed runs can write logs in multiple places:

- repository-root `logs/<run-id>.log`
- `<run-root>/<run-id>.log`
- SQLite `runs.log` column

The web runtime continuously updates these locations during execution.

## Post-Processing Artifacts

When a UI run completes successfully, the runtime may additionally create:

- `predictions/rmsd_results.json`
- `predictions/tmscore_results.json`
- Vina-related outputs inside the run directory, depending on Vina config

These artifacts are best-effort and depend on the relevant scripts and runtime
configuration being available.

## Global Artifacts

Outside individual run roots, the repository may also contain:

- `runhistory.csv`: appended summary metrics for completed UI runs
- `~/.boltz/` or the `BOLTZ_CACHE` directory: model weights and cached assets

## Download Surface

The UI exposes these artifacts through:

- `/api/files/[...path]` for direct file serving
- `/api/runs/[id]/download` for zip export of a run root

## Operational Notes

- Existing predictions are skipped unless `--override` is set.
- The run root naming convention is stable and derived from the input stem.
- The web app uses the stored `config` JSON to recover run paths for later
  analysis and download routes.
