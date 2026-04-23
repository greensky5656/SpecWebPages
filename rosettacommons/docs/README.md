# Affinity Pipeline

This repository implements a deterministic, fail-closed paired mutation-effect
pipeline built around qualified Boltz outputs and Rosetta rescoring. It ingests
reference/wild-type and mutant protein-small-molecule cases, prepares
Rosetta-ready complexes from authoritative ligand chemistry, runs score-only and
local-refinement protocols, aggregates branch and paired delta features, and
emits research evidence or calibrated mutation-affinity reports depending on
mode and approvals.

Pipeline flowchart:

- Rendered SVG: `docs/pipeline_flowchart.svg`
- Editable Mermaid source: `docs/pipeline_flowchart.md`

![Pipeline flowchart](docs/pipeline_flowchart.svg)

## Documentation

The files below are the current maintained entrypoints. Legacy planning and
governance files outside this curated set may provide broader background, but
the documents listed here are the fastest way to understand the implemented
codebase.

Start with these current documents:

- [Setup and install guide](docs/setup_install_guide.md): native research setup,
  real Rosetta prerequisites, environment variables, schema export, and test
  entrypoints
- [Affinity pipeline implementation specification](docs/affinity_pipeline_implementation_spec.md):
  implementation-scoped specification for the current Boltz-to-Rosetta pipeline
- [Design specification](docs/design_specification.md): system architecture,
  runtime modes, fail-closed boundaries, and component responsibilities
- [CLI reference](docs/cli_reference.md): command-by-command usage, arguments,
  outputs, and guardrails
- [Runtime and artifacts reference](docs/runtime_artifacts_reference.md): case
  and pair directory structure, manifests, Rosetta outputs, reports, and audit
  artifacts
- [Paired mutation scoring method](docs/paired_mutation_scoring_method.md):
  branch scoring, paired delta construction, calibration gating, and QC logic
- [Pipeline execution chain](docs/pipeline_execution_chain.md): end-to-end state
  progression from case intake to report emission
- [Pipeline flowchart](docs/pipeline_flowchart.md): editable Mermaid source for
  the current end-to-end pipeline
- [Rendered pipeline flowchart](docs/pipeline_flowchart.svg): visual diagram for
  environments that do not render Mermaid

For machine-readable contracts, use `configs/schemas/` and the active pipeline
profiles under `configs/` directly.

The current codebase supports:

- paired reference/wild-type and mutant mutation-effect workflows
- single-case research rescoring workflows
- dual-format Boltz structure ingest from `.cif`, `.mmcif`, and `.pdb`
- authoritative ligand intake from `sdf`, `mol`, `mol2`, `smiles`, or `ccd`
- `research`, `shadow`, and `clinical` runtime modes
- paired calibrated output only when an approved calibration bundle is present

## What The Pipeline Does

For a qualified Boltz case or paired reference/mutant case, the pipeline:

1. validates structured Boltz provenance and selects the top confidence samples
2. normalizes `.mmcif` and `.pdb` structure inputs into one ingest path
3. loads and canonicalizes the authoritative ligand definition
4. generates Rosetta ligand params and atom-name mappings
5. builds Rosetta-ready complex PDBs with normalized ligand chain and residue
   naming
6. runs Rosetta score-only and local-refine protocols for the selected poses
7. aggregates branch-level Rosetta summaries and paired delta features
8. applies operating-domain checks, calibration gates, and QC rules
9. writes JSON and Markdown reports plus hash-linked audit artifacts

## Repository Layout

- `src/affinity_pipeline/`: main package and CLI
- `src/affinity_pipeline/ingest/`: Boltz provenance loading, sample selection,
  structure parsing, and manifest creation
- `src/affinity_pipeline/chemistry/`: authoritative ligand loading,
  canonicalization, atom mapping, params generation, and complex preparation
- `src/affinity_pipeline/rosetta/`: command construction, XML rendering,
  execution, and scorefile parsing
- `src/affinity_pipeline/aggregation/`: branch summaries, cross-pose metrics,
  uncertainty, and QC
- `src/affinity_pipeline/calibration/`: domain checks, feature vectors, linear,
  isotonic, and paired `ΔΔG` bundle loaders
- `src/affinity_pipeline/reporting/`: JSON report writing, Markdown rendering,
  and audit log emission
- `src/affinity_pipeline/qualification/`: environment and release guardrails
- `configs/`: runtime profiles, schema exports, Rosetta templates, and example
  manifests
- `scripts/`: environment bootstrap, Rosetta installation, qualification, and
  bundle-building helpers
- `tests/`: `unittest` coverage for ingest, mapping, Rosetta execution guards,
  aggregation, calibration, qualification, and reporting
- `docs/`: curated implementation and operational documentation
- `data/`: checked-in calibration examples and bundle templates
- `work/`: local generated outputs, benchmark runs, and environment manifests

## Installation

The Python package targets Python 3.11+ and exposes the CLI module
`affinity_pipeline.cli`.

```bash
python3 -m pip install -e .
```

For native research setup and Rosetta prerequisites, see
`docs/setup_install_guide.md`.

## CLI

### Inspect The CLI

```bash
PYTHONPATH=src python3 -m affinity_pipeline.cli --help
```

### Run A Single Case

```bash
PYTHONPATH=src python3 -m affinity_pipeline.cli run-case \
  --run-manifest /path/to/run_manifest.json \
  --out /path/to/work_root \
  --rosetta-bin /path/to/rosetta_scripts.default.linuxgccrelease \
  --rosetta-db /path/to/database \
  --rosetta-source-root /path/to/rosetta/source
```

### Run A Paired Reference/Mutant Workflow

```bash
PYTHONPATH=src python3 -m affinity_pipeline.cli run-pair \
  --run-manifest /path/to/pair_run_manifest.json \
  --out /path/to/work_root \
  --rosetta-bin /path/to/rosetta_scripts.default.linuxgccrelease \
  --rosetta-db /path/to/database \
  --rosetta-source-root /path/to/rosetta/source
```

### Export The Current JSON Schemas

```bash
PYTHONPATH=src python3 -m affinity_pipeline.cli export-schemas
```

Additional commands are documented in `docs/cli_reference.md`.

## Configuration

Runtime profiles currently shipped in `configs/`:

- `pipeline.defaults.yaml`: shared defaults and QC thresholds
- `pipeline.research.yaml`: research-mode override
- `pipeline.shadow.yaml`: shadow-mode override
- `pipeline.clinical.yaml`: clinical-mode override

Important behaviors in the current implementation:

- `research`, `shadow`, and `clinical` are the only supported modes
- regulated modes require cross-format equivalence checks
- clinical mode requires both `calibration_bundle` and
  `environment_manifest`
- Rosetta ligand identity is normalized to residue `LIG` on chain `X`
- Rosetta commands run in isolated work directories with thread-count pins

## Execution Modes

### `research`

Research mode allows the implemented Boltz-to-Rosetta workflow without
requiring calibration or qualified environment gating. It still enforces
fail-closed chemistry, provenance, and file-contract checks.

### `shadow`

Shadow mode keeps the same deterministic pipeline but requires stricter input
and cross-format controls. Calibrated output is allowed only when an approved
bundle exists and the case remains inside the declared domain.

### `clinical`

Clinical mode adds mandatory environment qualification and calibration-bundle
requirements on top of the shadow controls. Unsupported, out-of-domain, or
unqualified runs return fail-closed statuses rather than best-effort results.

## Inputs And Supported Scope

The implemented v1 path expects:

- completed Boltz outputs already present on disk
- one qualified protein-small-molecule ligand per branch
- authoritative ligand chemistry supplied separately from the Boltz coordinates
- paired reference/wild-type and mutant cases for mutation-effect reporting

Current v1 boundaries:

- protein targets only
- same ligand identity across a reference/mutant pair
- mutation-effect estimation rather than absolute single-structure affinity
- no silent chemistry repair for ambiguous ligand mapping, protonation,
  stereochemistry, or cross-format disagreements

## Real Compute Notes

The runtime code expects:

- `gemmi`
- `rdkit`
- a real Rosetta build exposing
  `rosetta_scripts.default.linuxgccrelease`
- a Rosetta `database/` directory
- Rosetta source access for
  `source/scripts/python/public/molfile_to_params.py`

Recommended operational entry points:

- `scripts/bootstrap_native_research.sh`
- `scripts/create_runtime_env.sh`
- `scripts/install_offline_runtime.sh`
- `scripts/install_rosetta_from_bundle.sh`
- `scripts/qualify_env_native.sh`

For real local runs and integration tests, set:

- `REAL_ROSETTA_BIN`
- `REAL_ROSETTA_DB`
- `REAL_ROSETTA_SOURCE_ROOT`

## Output Artifacts

A successful case or pair run writes a structured work directory. Important
paths include:

- `ingest/run_manifest.json`
- `ingest/case_manifest.json`
- `ingest/copied_inputs/`
- `prepared/authoritative_ligand.sdf`
- `prepared/LIG.params`
- `prepared/complex_rosetta.pdb`
- `prepared/prep_manifest.json`
- `prepared/pose_*/complex_rosetta.pdb`
- `rosetta/score_only/pose_*/score.sc`
- `rosetta/local_refine/pose_*_rep_*/score.sc`
- `rosetta/local_refine/pose_*_rep_*/run_manifest.json`
- `rosetta/rendered/*.xml`
- `reports/score_report.json`
- `reports/score_report.md`
- `reports/audit_log.jsonl`

Pair runs additionally write:

- `branches/reference/...`
- `branches/mutant/...`
- `ingest/run_manifest.json` for the pair
- `reports/score_report.json` with paired delta fields and mutation metadata

## Validation, Qualification, And Release

- `affinity_pipeline.cli qualify-env` writes an `EnvironmentManifest`
  describing the validated Rosetta environment and mode eligibility
- `configs/schemas/` contains the exported JSON Schemas for manifests and
  reports
- `scripts/build_pair_ddg_bundle.py` and `scripts/build_shadow_proxy_bundle.py`
  assemble calibration bundles used by regulated modes
- `docs/release_readiness_review.md` tracks the broader release posture beyond
  the implemented runtime code

## Testing

The checked-in test suite is `unittest`-based.

```bash
make test
```

Key coverage areas include:

- ingest and provenance handling
- atom mapping and chemistry preparation
- Rosetta execution guards and protocol tuning
- aggregation, QC, and paired feature construction
- calibration gating
- qualification and reporting

## Current Safety Model

The implementation is intentionally fail-closed:

- unsupported or ambiguous chemistry must stop the run
- regulated modes require stricter provenance and cross-format equivalence
- calibration is blocked unless an approved bundle is present
- clinical mode requires an environment manifest that matches the qualified
  Rosetta toolchain
- result statuses degrade to `PASS_WITH_WARNINGS` or `FAIL_NO_RESULT` instead
  of returning a best-effort numeric claim
