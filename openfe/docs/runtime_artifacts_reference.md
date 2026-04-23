# Runtime And Artifacts Reference

This document summarizes the current runtime outputs written by the pipeline and
what they represent.

## Output Directory Structure

A pipeline run writes a structured output directory rooted at the `--out` path
provided to `mutation-rbfe run`.

Important subdirectories and files include:

- `manifests/paired_manifest.json`
- `selection/model_selection.json`
- `fep/complex_leg/replicate_*.json`
- `fep/apo_leg/replicate_*.json`
- `fep/replicate_observability.json`
- `inference/ddg_result.json`
- `calibration/wt_anchor.json`
- `reports/mutation_affinity_report.json`
- `reports/mutation_affinity_report.signature.json`
- `logs/state_ledger.json`
- `logs/stage_timings.json`
- `logs/pipeline.log`
- `provenance/input_hashes.json`
- `provenance/software_sbom.json`

## Manifest Artifacts

### `manifests/paired_manifest.json`

The paired manifest is the central case-local runtime contract. It captures:

- WT and MUT run roots
- selected WT and MUT model paths
- ligand and receptor paths
- mutation metadata
- provenance input hashes
- selection extensions and model-pair metadata

This manifest is required by the production execution path before external leg
execution is launched.

### `selection/model_selection.json`

This file records the selected model indices and summary selection metrics,
including:

- WT and MUT model index
- paired ligand RMSD after alignment
- selected confidence and clash-related scores

## FEP Artifacts

### `fep/complex_leg/replicate_*.json`

Per-replicate complex-leg payloads. Depending on execution mode, these contain:

- deterministic synthetic values in `research_mode`, or
- external workflow output plus timing and protocol fields in `production_mode`

### `fep/apo_leg/replicate_*.json`

Per-replicate apo-leg payloads following the same pattern as complex-leg outputs.

### `fep/replicate_observability.json`

Case-level observability summary over all expected legs and replicates. It tracks:

- execution mode
- expected replicate count
- persisted complex-leg payloads
- persisted apo-leg payloads

## Analysis Artifacts

### `inference/ddg_result.json`

This file records:

- estimated `ddG`
- `sigma_ddg_kcal_per_mol`
- summarized complex-leg statistics
- summarized apo-leg statistics

### `calibration/wt_anchor.json`

This file records:

- WT anchor `delta_g_wt_kcal_per_mol`
- WT anchor uncertainty
- the set of anchor records actually used

## Report Artifacts

### `reports/mutation_affinity_report.json`

This is the final output contract for a pipeline run. It is written for both
successful and fail-closed runs.

The report includes:

- terminal status
- mutation metadata
- `delta_delta_g_kcal_per_mol`
- uncertainty values and 95% intervals
- calibrated mutant affinity outputs
- WT anchor summary
- QC gate results
- provenance

For fail-closed outcomes, the written report may be a reduced fallback payload
centered on terminal status, error details, timestamps, and provenance pointers.

Terminal status is always one of:

- `SUCCESS`
- `NO_CALL`
- `FAILED_INFRA`

### `reports/mutation_affinity_report.signature.json`

This file contains the signature payload generated for the final report.

## Logging And Provenance

### `logs/state_ledger.json`

Monotonic state transitions for the case, including stage-associated artifacts and
terminal outcome.

### `logs/stage_timings.json`

Per-stage elapsed wall-clock timing plus total runtime summary.

### `logs/pipeline.log`

Append-only pipeline log written during orchestration.

### `provenance/input_hashes.json`

SHA-256 hashes for the selected WT and MUT ligand and receptor inputs.

### `provenance/software_sbom.json`

Runtime software provenance emitted by the pipeline, including:

- project identifier
- config hash
- high-level runtime engine metadata
- declared dependency metadata

## Failure Behavior

The pipeline is fail-closed. When a run ends in `NO_CALL` or `FAILED_INFRA`, it
still writes:

- timing data
- state ledger data
- a final report JSON
- a final report signature

That behavior is intentional and allows downstream monitoring and audit even for
non-successful runs.

## Related Contracts

- Intake schema: `specs/schemas/boltz_mutation_pair_intake.schema.json`
- Report schema: `specs/schemas/mutation_affinity_report.schema.json`
- Orchestration: `src/pipeline.py`
- Report writing: `src/reporting/report_writer.py`
