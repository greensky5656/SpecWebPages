# Affinity Pipeline Implementation Specification

This document is the curated implementation entrypoint for the current
Boltz-to-Rosetta codebase. It distills the live repository behavior into one
practical reference while preserving the broader canonical sources:

- `implementation_spec_dual_format.md`: detailed canonical build specification
- `plan_dual_format.md`: strategic plan, release posture, and governance context
- `implementation_working_knowledge.md`: precedence and implementation rules

## Purpose

The implemented product is a deterministic, fail-closed paired mutation-effect
pipeline that:

1. starts from completed Boltz outputs already present on disk
2. accepts `*.cif`, `*.mmcif`, and `*.pdb` Boltz structures
3. loads authoritative ligand chemistry separately from the structure files
4. prepares Rosetta-ready protein-ligand complexes
5. runs Rosetta score-only and local-refinement rescoring
6. constructs branch-level and paired delta features
7. emits a research evidence package or a calibrated mutation-affinity report
   depending on mode and approvals

## Implemented Scope

The current code implements these core boundaries:

- protein plus one non-covalent small-molecule ligand
- one case at a time or one paired reference/mutant workflow
- paired mutation-effect reporting for the same ligand identity across the pair
- research, shadow, and clinical runtime modes
- fail-closed handling for ambiguous chemistry, missing provenance, unsupported
  structure comparisons, or missing regulated-mode prerequisites

The current code does not treat raw Rosetta scores as experimental affinity.
Calibrated output is only considered in non-research modes when a calibration
bundle is present and the case remains inside the declared operating domain.

## Core Runtime Contracts

### Input Contracts

Single-case execution uses `RunManifest` from `configs/schemas/run_manifest.schema.json`.
Important fields:

- `run_id`
- `mode`
- `case_dir`
- `ligand_source`
- `top_k_boltz_samples`
- `rosetta_replicates_per_pose`
- `rosetta_seed_base`
- `preferred_structure_format`
- `require_cross_format_equivalence`
- `config_profile`
- `calibration_bundle`
- `environment_manifest`

Pair execution uses `PairRunManifest` from
`configs/schemas/pair_run_manifest.schema.json` and replaces `case_dir` with:

- `reference_case_dir`
- `mutant_case_dir`
- `mutation_descriptor`

### Runtime Profiles

The code loads defaults from `configs/pipeline.defaults.yaml` and overlays one
of:

- `configs/pipeline.research.yaml`
- `configs/pipeline.shadow.yaml`
- `configs/pipeline.clinical.yaml`

These profiles define:

- baseline platform expectations
- Rosetta naming and protocol defaults
- cross-format comparison thresholds
- QC thresholds
- reporting toggles

### Output Contracts

The code exports schemas for:

- `run_manifest.schema.json`
- `pair_run_manifest.schema.json`
- `case_manifest.schema.json`
- `prep_manifest.schema.json`
- `rosetta_run_manifest.schema.json`
- `environment_manifest.schema.json`
- `score_report.schema.json`
- `pair_score_report.schema.json`
- `calibration_bundle.schema.json`

## Implemented Component Layout

- `src/affinity_pipeline/cli.py`: top-level command dispatch
- `src/affinity_pipeline/ingest/`: Boltz discovery, provenance handling, sample
  selection, and structure parsing
- `src/affinity_pipeline/chemistry/`: ligand loading, canonicalization, atom
  mapping, Rosetta params generation, and complex preparation
- `src/affinity_pipeline/rosetta/`: protocol rendering, command assembly,
  execution, and score parsing
- `src/affinity_pipeline/aggregation/`: per-pose summaries, cross-pose metrics,
  QC, and paired feature construction
- `src/affinity_pipeline/calibration/`: domain checks, feature vectors, and
  bundle-backed calibration
- `src/affinity_pipeline/reporting/`: JSON report writing, Markdown rendering,
  and audit logging
- `src/affinity_pipeline/qualification/`: environment qualification and
  regulated-mode guards

## End-To-End Behavior

### Single Case

1. Load and validate the run manifest.
2. Discover Boltz provenance, confidence JSONs, structure files, and affinity
   JSON.
3. Select the top confidence Boltz samples.
4. Compare `.mmcif` and `.pdb` structures when both are present.
5. Copy the selected source artifacts into the working tree and write
   `ingest/case_manifest.json`.
6. Canonicalize the authoritative ligand and generate `LIG.params`.
7. Map the authoritative ligand onto Boltz coordinates and Rosetta atom names.
8. Build `prepared/complex_rosetta.pdb` plus per-pose prepared complexes.
9. Run a Rosetta load test, then score-only and local-refine protocols.
10. Summarize Rosetta outputs, apply domain checks and QC, and write JSON and
    Markdown reports.

### Paired Workflow

1. Load and validate the pair run manifest.
2. Ingest the reference branch into `branches/reference/`.
3. Ingest the mutant branch into `branches/mutant/`.
4. Prepare both branches independently.
5. Run Rosetta for both branches.
6. Summarize branch-level reports.
7. Construct paired delta features.
8. Apply calibration and QC gates at the pair level.
9. Write a paired `reports/score_report.json` and `reports/score_report.md`.

## Runtime Modes

- `research`: deterministic feature-generation mode; calibrated output is
  disabled
- `shadow`: calibrated output allowed only when bundle and domain checks pass
- `clinical`: same calibrated path plus mandatory environment qualification

For regulated modes, cross-format equivalence remains enabled and a missing
calibration bundle is a fail-closed condition.

## Fail-Closed Rules

The current implementation is intentionally conservative:

- ambiguous or unsupported chemistry is rejected
- missing provenance becomes a hard failure in stricter modes
- dual-format disagreement can block execution
- missing calibration in regulated modes prevents numeric output
- an unqualified clinical environment blocks execution
- result statuses fall back to `PASS_WITH_WARNINGS` or `FAIL_NO_RESULT`
  instead of silently emitting best-effort claims

## Real Toolchain Expectations

The live preparation and execution path expects:

- `gemmi`
- `rdkit`
- a real Rosetta executable
- a Rosetta `database/`
- Rosetta source access for `molfile_to_params.py`

See `docs/setup_install_guide.md` for install details and
`docs/cli_reference.md` for command usage.
