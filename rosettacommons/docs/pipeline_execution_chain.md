# Pipeline Execution Chain

This document describes the current end-to-end execution chain implemented in
the repository. It follows the live CLI and the current on-disk artifact model.

## High-Level Chain

```text
RunManifest or PairRunManifest
-> environment qualification when required
-> ingest selected Boltz inputs
-> prepare Rosetta-ready complexes
-> execute Rosetta protocols
-> aggregate branch summaries
-> construct pair deltas when applicable
-> apply domain and calibration gates
-> write reports and audit trail
```

## Single-Case Chain

### 1. Load The Run Manifest

The CLI validates a `RunManifest` and resolves the active config profile.

Important fields:

- mode
- case location
- ligand source
- sample count
- replicate count
- structure-format preference
- calibration bundle
- environment manifest

### 2. Enforce Environment Qualification When Required

- `clinical` requires a passing environment manifest
- `shadow` validates the environment manifest when one is supplied
- `research` does not require the manifest

### 3. Ingest The Boltz Case

The ingest stage:

- discovers provenance metadata
- locates confidence JSON files and structure files
- selects the top confidence samples
- optionally compares `.mmcif` and `.pdb`
- copies the retained source files into `ingest/copied_inputs/`
- writes `ingest/case_manifest.json`

### 4. Prepare The Chemistry And Complex

The preparation stage:

- loads and canonicalizes the authoritative ligand
- writes `prepared/authoritative_ligand.sdf`
- generates `prepared/LIG.params`
- builds `prepared/complex_rosetta.pdb`
- builds per-pose prepared complexes
- performs a Rosetta load test
- writes `prepared/prep_manifest.json`

### 5. Execute Rosetta

The Rosetta stage:

- renders XML into `rosetta/rendered/`
- runs score-only once per selected pose
- runs local-refine once per pose and replicate
- writes `score.sc`, logs, and Rosetta run manifests

### 6. Aggregate Results

The aggregation stage:

- parses all `score.sc` files
- computes baseline and refined summaries
- computes per-pose summaries
- computes cross-pose disagreement metrics
- checks the operating domain
- optionally applies calibration
- applies QC status rules

### 7. Write Reports

The reporting stage:

- writes `reports/score_report.json`
- renders `reports/score_report.md`
- appends `reports/audit_log.jsonl`

## Pair Chain

The paired workflow uses the same per-branch stages, then adds a pair layer.

### 1. Load The Pair Manifest

The CLI validates a `PairRunManifest` containing:

- reference case location
- mutant case location
- mutation descriptor
- ligand source
- mode
- calibration and environment inputs

### 2. Build Branch Work Trees

The CLI creates:

- `branches/reference/`
- `branches/mutant/`

Each branch receives its own ingest, preparation, Rosetta, and reporting path.

### 3. Aggregate Branch Reports

The pair summarizer reuses the branch report logic for both branches, then
constructs pair deltas.

### 4. Build Paired Features

The current paired feature layer computes:

- delta best refined interface score
- delta median refined interface score
- delta ligand IPTM
- delta Boltz affinity predictor value
- combined uncertainty

### 5. Apply Pair-Level Gates

Pair-level QC adds checks such as:

- ligand identity mismatch
- ligand charge mismatch
- branch QC carry-forward
- calibration availability
- domain support

### 6. Write The Pair Report

The paired workflow writes:

- `reports/score_report.json`
- `reports/score_report.md`
- `reports/audit_log.jsonl`

The top-level pair report references the reference and mutant branch outputs
through hashes and summarized sections rather than duplicating all raw files.

## Execution Properties

The implemented execution chain is designed to be:

- deterministic through manifest-driven inputs and seeded Rosetta runs
- auditable through hash-linked manifests and audit events
- fail-closed when critical requirements are not met
- mode-aware so research and regulated behaviors remain distinct

## Where To Look In Code

- `src/affinity_pipeline/cli.py`
- `src/affinity_pipeline/ingest/boltz_case.py`
- `src/affinity_pipeline/chemistry/complex_builder.py`
- `src/affinity_pipeline/rosetta/runner.py`
- `src/affinity_pipeline/aggregation/summarize.py`
- `src/affinity_pipeline/reporting/markdown_report.py`
