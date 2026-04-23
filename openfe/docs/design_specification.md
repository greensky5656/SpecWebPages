# Design Specification

This document describes the current implemented design of the mutation RBFE
pipeline as reflected in the codebase today.

## Purpose

The repository implements a fail-closed WT-referenced residue-mutation RBFE
pipeline. Given a WT and MUT `boltz_results_*` pair, it builds a paired manifest,
validates mutation scope and invariants, executes or simulates apo and complex
legs, estimates `ddG`, calibrates against WT anchor data, applies QC gates, and
emits a signed mutation affinity report.

## Supported Scope

The current implementation supports:

- single-site residue substitutions
- same-chain multi-site residue substitutions
- unchanged ligand identity between WT and MUT
- deterministic `research_mode` runs for dry-run and CI use
- external-workflow-backed `production_mode` runs

The current implementation does not support:

- multi-site substitutions across multiple chains
- ligand changes between WT and MUT
- fallback execution paths when required checks fail

Unsupported or invalid cases resolve to fail-closed terminal outcomes rather than
alternative execution paths.

## Inputs

The pipeline expects two Boltz-style run roots:

- one WT run root
- one MUT run root

Each run root must contain:

- exactly one run YAML at the root
- prediction/model metadata under `predictions/<run_name>/`
- selected model ligand and receptor artifacts under
  `vina_analysis/<run_name>/<run_name>_model_<idx>/`
- optional `receptor_with_cofactors.pdb` artifacts when cofactor-inclusive model
  selection is enabled

From these inputs, the intake path derives:

- protein sequences by chain
- binder CCD identity
- selected WT and MUT model artifacts
- ligand chemistry hashes
- receptor and ligand paths for the selected model pair

## Top-Level Architecture

Main implemented modules:

- `src/cli.py`: CLI entrypoint
- `src/pipeline.py`: orchestration and terminal report handling
- `src/intake/`: run parsing, candidate discovery, paired manifest creation
- `src/mutation/`: WT-to-MUT sequence difference validation and mutation typing
- `src/fep/`: research-mode synthetic legs and production-mode external leg execution
- `src/inference/`: `ddG` estimation
- `src/calibration/`: WT anchor harmonization and mutant affinity conversion
- `src/qc/`: gate evaluation
- `src/reporting/`: report writing and signing
- `src/validation/`: PLATINUM holdout validation
- `src/ops/`: monitoring summaries over generated reports
- `src/release/`: release-bundle assembly

## Execution Flow

For `mutation-rbfe run`, the implemented flow is:

1. Load the protocol YAML.
2. Resolve relative paths for schemas, WT anchor records, and workflow script.
3. Merge `qc_gates_path` content into the runtime gates config.
4. Build `manifests/paired_manifest.json`.
5. Validate the intake manifest against the intake schema.
6. Write provenance input hashes.
7. Write model-selection metadata.
8. Validate mutation mapping and ligand/cofactor invariants.
9. Run FEP legs:
   - synthetic deterministic values in `research_mode`
   - external workflow execution in `production_mode`
10. Summarize complex and apo legs and estimate `ddG`.
11. Harmonize WT anchor data and convert to mutant affinity.
12. Evaluate QC gates.
13. Write and sign the final report.
14. Validate the final report against the report schema.
15. Write timing, ledger, and software provenance artifacts.

## Execution Modes

### `research_mode`

`research_mode` is implemented as a deterministic pipeline-verification mode.
It does not execute real PMX/GROMACS mutation legs. Instead it produces
deterministic synthetic leg values while still exercising:

- intake
- mutation validation
- analysis
- calibration
- QC
- reporting
- signing
- artifact generation

### `production_mode`

`production_mode` requires:

- a resolvable `pmx` binary
- a resolvable `gmx` binary
- an existing `execution.workflow_script`
- a persisted `manifests/paired_manifest.json`
- a non-default signing key
- a pinned `execution.openfe_version`

Leg execution is delegated to the configured external workflow path. The current
default production config points to `scripts/pmx_gmx_workflow.py`.

## Mutation Validation Rules

The implemented mutation validation path enforces:

- WT and MUT chain sets must match
- WT and MUT sequence lengths must match chain-by-chain
- one or more sequence differences are required
- all supported mutation differences must be on the same chain
- WT and MUT ligand chemistry hashes must match
- if cofactors are retained, WT and MUT cofactor sets must match

Mutation metadata is propagated as:

- mutation descriptor
- site count
- per-site chemistry classification via `sites[].site_class`
- per-site chain and residue identity

## QC And Terminal Outcomes

The pipeline is intentionally fail-closed. Terminal report status is always one
of:

- `SUCCESS`
- `NO_CALL`
- `FAILED_INFRA`

Broadly:

- scientific, mapping, applicability-domain, and QC failures resolve to `NO_CALL`
- workflow and infrastructure failures resolve to `FAILED_INFRA`

The pipeline still writes a report artifact for fail-closed outcomes.

## State And Provenance

The implementation writes auditable runtime artifacts including:

- `logs/state_ledger.json`
- `logs/stage_timings.json`
- `logs/pipeline.log`
- `provenance/input_hashes.json`
- `provenance/software_sbom.json`
- signed final report artifacts

The final report also records:

- config hash
- execution mode
- random seed
- forcefield metadata
- software versions
- hardware profile
- input hashes

## Output Contract

Important output artifacts include:

- `manifests/paired_manifest.json`
- `selection/model_selection.json`
- `fep/complex_leg/replicate_*.json`
- `fep/apo_leg/replicate_*.json`
- `fep/replicate_observability.json`
- `inference/ddg_result.json`
- `calibration/wt_anchor.json`
- `reports/mutation_affinity_report.json`
- `reports/mutation_affinity_report.signature.json`

The final report schema includes:

- terminal status
- mutation metadata
- `delta_delta_g_kcal_per_mol`
- uncertainty fields
- calibrated affinity outputs
- WT anchor summary
- QC result payload
- provenance

## Operational Constraints

Current code-level constraints worth knowing:

- at least 3 replicates are required
- report signing is always performed
- production mode refuses the default research signing key
- production mode refuses an unpinned `execution.openfe_version`
- release-bundle assembly expects specific configs, schemas, and documentation
  artifacts to exist
- the current release-bundle implementation packages the maintained
  implementation/specification, setup, CLI, runtime-reference, and flowchart docs

## Source-Of-Truth References

- CLI: `src/cli.py`
- Orchestration: `src/pipeline.py`
- Manifest build: `src/intake/pair_builder.py`
- Model selection: `src/selection/model_selector.py`
- Mutation validation: `src/mutation/mapping_validator.py`
- FEP runner: `src/fep/pmx_runner.py`
- Report writer: `src/reporting/report_writer.py`
- Research config: `configs/openfe/mutation_rbfe_protocol.yaml`
- Production config: `configs/openfe/mutation_rbfe_protocol.production.yaml`
