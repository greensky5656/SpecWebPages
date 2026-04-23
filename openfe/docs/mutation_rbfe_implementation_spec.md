# OpenFE Mutation RBFE Implementation Specification

This file has been refreshed to reflect the current implemented code path.

The canonical maintained narrative spec is now:

- `docs/design_specification.md`

Use that file as the primary implementation-facing document. This file is kept as
an alternate specification entrypoint because older references may still point
here.

## Current Locked Decisions

- Production mutation engine is `pmx + GROMACS`.
- The pipeline is WT-referenced and fail-closed.
- Supported mutation scope is one or more substitutions on the same chain.
- WT and MUT ligand chemistry must match.
- Terminal outcomes are restricted to `SUCCESS`, `NO_CALL`, and `FAILED_INFRA`.

## Current Runtime Contract

Primary CLI:

```bash
mutation-rbfe run --wt-run <path> --mut-run <path> --config <yaml> --out <dir>
```

Primary outputs:

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
- `provenance/input_hashes.json`
- `provenance/software_sbom.json`

Expected selected model artifacts come from Boltz-style run roots and include:

- `predictions/<run_name>/...`
- `vina_analysis/<run_name>/<run_name>_model_<idx>/ligand.sdf`
- `vina_analysis/<run_name>/<run_name>_model_<idx>/receptor.pdb`
- optional `receptor_with_cofactors.pdb` when cofactor-inclusive selection is enabled

## Current Execution Modes

### `research_mode`

- deterministic synthetic complex and apo leg values
- used for dry-run and CI-style pipeline verification
- still exercises intake, mapping, analysis, calibration, QC, and reporting

### `production_mode`

- requires `pmx` and `gmx` on `PATH`
- requires an existing `execution.workflow_script`
- requires a non-default signing key
- requires a pinned `execution.openfe_version`
- delegates leg execution through the configured external workflow script

## Current Validation Rules

- one or more WT-to-MUT sequence differences are allowed
- multi-site support currently requires all substitutions on the same chain
- WT and MUT protein chain sets must match
- WT and MUT sequence lengths must match per chain
- WT and MUT ligand hashes must match
- retained cofactor sets must match when cofactors are enabled

## Current Provenance And Security Model

- SHA-256 hashes are written for selected ligand and receptor inputs
- final reports are always signed
- stage timing and state ledger artifacts are always written
- fail-closed outcomes still emit report and signature artifacts
- the current release-bundle implementation packages the maintained
  implementation/specification, setup, CLI, runtime-reference, and flowchart docs

## Related Current Documents

- `docs/design_specification.md`
- `docs/setup_install_guide.md`
- `docs/cli_reference.md`
- `docs/runtime_artifacts_reference.md`
- `docs/pipeline_flowchart.md`
