# Requirements

This document records the current implemented requirements for the OpenFE
mutation RBFE pipeline in a stricter, review-friendly format. It is intended to
describe the current shipped pipeline behavior, not future benchmark or platform
ambitions.

## Intended use

OpenFE is intended for WT-referenced mutation-effect estimation using a fail-
closed mutation RBFE workflow built around paired WT/MUT Boltz inputs, complex
and apo mutation legs, aggregation, calibration, QC gating, signed reporting,
and operational release surfaces.

## System requirements

- Python 3.11+ with the package installed from the repository checkout.
- The declared Python dependency set from `pyproject.toml`, plus the required
  repository configs, schemas, and scripts.
- In `production_mode`, resolvable `pmx` and `gmx` executables.
- In `production_mode`, a real workflow script at the configured
  `execution.workflow_script` path.
- A non-default signing key and pinned `execution.openfe_version` for regulated
  production surfaces.

## Input requirements

- One WT and one MUT Boltz run root for each scientific case.
- Inputs that support paired model selection, mutation mapping, and ligand /
  cofactor consistency checks.
- Runtime configuration with valid protocol, QC-gate, schema, and auxiliary
  paths that resolve correctly relative to the chosen config file.
- `execution.replicates >= 3`.
- Mutation scope limited to the currently supported single-site substitutions
  and same-chain multi-site substitutions.

## Functional requirements

- The pipeline must build a paired intake manifest from WT/MUT Boltz runs.
- It must validate mutation mapping and reject unsupported or inconsistent
  pairings.
- It must execute complex and apo mutation legs or deterministic synthetic legs,
  depending on mode.
- It must aggregate leg data into `inference/ddg_result.json`, harmonize WT
  anchors, apply QC gates, and write a signed final report.
- It must support the documented CLI surfaces for running cases, validating
  holdout summaries, monitoring reports, verifying signatures, and building
  release bundles.

## Output requirements

- A structured output tree including manifests, inference artifacts, reports,
  signatures, provenance artifacts, and timing/monitoring outputs.
- Final report artifacts centered on `reports/mutation_affinity_report.json` and
  its corresponding signature artifact.
- Terminal outputs that reflect only the current allowed terminal states.

## Safety and fail-closed requirements

- The pipeline is limited to the three terminal states `SUCCESS`, `NO_CALL`, and
  `FAILED_INFRA`.
- Unsupported mappings, missing production prerequisites, or invalid regulated-
  mode settings must fail closed rather than degrade to best-effort execution.
- `research_mode` must still exercise intake, mapping, aggregation,
  calibration, QC, reporting, signing, and artifact generation, even though it
  does not run real PMX/GROMACS legs.

## Non-requirements / out-of-scope behavior

- The current shipped package does not require a benchmark harness as part of
  the primary CLI surface.
- The implemented contract is WT-referenced mutation-effect estimation, not a
  general absolute binding-affinity workflow.
- The code does not require or promise support for arbitrary mutation classes
  beyond the current validated intake and mapping rules.
