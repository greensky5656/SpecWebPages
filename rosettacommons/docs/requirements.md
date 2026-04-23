# Requirements

This document records the current implemented requirements for the
RosettaCommons affinity pipeline in a stricter, review-friendly format. It is
source-grounded and describes the current deterministic Boltz-to-Rosetta
mutation-effect rescoring workflow rather than a broader aspirational platform.

## Intended use

The RosettaCommons pipeline is intended for deterministic paired mutation-effect
rescoring and calibrated reporting using qualified Boltz outputs, authoritative
ligand chemistry, Rosetta score-only/local-refine protocols, and mode-dependent
QC/calibration controls.

## System requirements

- Python 3.11+ with the package installed from the repository checkout.
- Python dependencies from `pyproject.toml`, including `pydantic`, `PyYAML`,
  `numpy`, `pandas`, `scipy`, `gemmi`, and `rdkit`.
- A real Rosetta installation exposing
  `rosetta_scripts.default.linuxgccrelease`.
- A matching Rosetta `database/` directory.
- Rosetta source access for
  `source/scripts/python/public/molfile_to_params.py`.

## Input requirements

- Completed Boltz outputs already present on disk before this pipeline starts.
- Structured run manifests for either single-case or paired reference/mutant
  workflows.
- Authoritative ligand chemistry supplied separately from the Boltz coordinate
  files.
- Runtime profiles and schema/config paths that resolve correctly from the
  selected `configs/` files.
- Calibrated paired output only when an approved calibration bundle is present
  and the case remains inside the declared operating domain.

## Functional requirements

- The pipeline must normalize supported structure formats (`.cif`, `.mmcif`,
  `.pdb`) into one ingest path.
- It must canonicalize ligand chemistry, generate Rosetta params and atom-name
  mappings, and build Rosetta-ready complexes.
- It must run score-only and local-refine Rosetta protocols for selected poses.
- It must aggregate branch summaries and paired delta features, then apply
  domain checks, calibration gates, and QC rules.
- It must emit JSON/Markdown reports and associated hash-linked audit artifacts.
- It must support the documented CLI surfaces for single-case runs, paired runs,
  and schema export.

## Output requirements

- Structured case/pair work roots containing manifests, Rosetta outputs,
  aggregation artifacts, reports, and audit files.
- JSON and Markdown reporting outputs for the current supported run surfaces.
- Hash-linked or provenance-linked artifacts sufficient for the current audit
  model.

## Safety and fail-closed requirements

- The only supported runtime modes are `research`, `shadow`, and `clinical`.
- Regulated modes require stricter cross-format and qualification controls.
- Clinical mode requires both `calibration_bundle` and
  `environment_manifest`.
- Unsupported, out-of-domain, or unqualified runs must fail closed rather than
  silently emit best-effort calibrated conclusions.

## Non-requirements / out-of-scope behavior

- The v1 path does not require general absolute single-structure affinity
  authority.
- The pipeline does not require silent chemistry repair for ambiguous ligand
  mapping, protonation, stereochemistry, or cross-format disagreements.
- The current contract is a mutation-effect rescoring and calibrated reporting
  workflow, not an alchemical RBFE engine.
