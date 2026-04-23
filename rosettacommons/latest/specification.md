# RosettaCommons Affinity Pipeline Specification


















Updated: 2026-04-22 17:56 PST

Repository: `/media/joethomas/Backup1/Github/RosettaCommons`

This review document is grounded in current code, tests, inspected artifacts, and maintained repository documentation where that documentation matches the shipped implementation.

## Table of contents

1. [Executive summary](#1-executive-summary)
2. [The stages of the pipeline from a thermodynamic view](#2-the-stages-of-the-pipeline-from-a-thermodynamic-view)
3. [Supporting documentation](#3-supporting-documentation)
4. [Scientific workflow actually implemented](#4-scientific-workflow-actually-implemented)
5. [What scientific reviewers should separate](#5-what-scientific-reviewers-should-separate)
6. [Current scientific and operational limits](#6-current-scientific-and-operational-limits)
7. [Recommended signoff questions](#7-recommended-signoff-questions)
8. [Top current clarifications](#8-top-current-clarifications)
9. [Actual shipped operator surface](#9-actual-shipped-operator-surface)
10. [Evidence appendix](#10-evidence-appendix)

## 1. Executive summary








### What this pipeline is
RosettaCommons is a deterministic, fail-closed paired mutation-effect rescoring and calibration workflow built from qualified Boltz outputs, authoritative ligand chemistry, Rosetta rescoring branches, paired delta features, and gated reporting modes.

### Strongest justified claim
Its strongest justified claim is governed mutation-effect prioritization and, when all gates pass, calibrated reporting inside a declared operating domain.

### Main limiting factor
The main limiting factor is that the workflow is a rescoring-and-calibration architecture rather than a direct thermodynamic free-energy perturbation method.

### Appropriate present use
The appropriate present use is controlled mutation-effect decision support with fail-closed regulated-mode behavior, not direct free-energy authority from raw Rosetta terms alone.








## 2. The stages of the pipeline from a thermodynamic view







- **Stage 1 — Qualified paired input definition:** reference/wild-type and mutant cases are normalized into a comparable paired problem with authoritative ligand chemistry and controlled structure ingest.
- **Stage 2 — Rosetta-ready state construction:** ligand params, atom mappings, and normalized complex models define the concrete structural states that Rosetta will score and refine.
- **Stage 3 — Rosetta branch evaluation:** score-only and local-refine branches generate deterministic energetic and structural summaries for each side of the comparison.
- **Stage 4 — Paired delta construction:** branch summaries are aggregated into paired delta features that represent mutation-effect evidence rather than direct experimental affinity values.
- **Stage 5 — Calibrated reporting gate:** only after domain checks, bundle approval, and QC pass does the pipeline translate those paired features into the calibrated mutation-effect reporting surface.
- **Thermodynamic boundary:** raw Rosetta terms are not binding free energies; the shipped workflow is a paired mutation-effect rescoring and calibration architecture rather than endpoint MM/GBSA or alchemical FEP.







## 3. Supporting documentation


















### Core documentation
- [Affinity pipeline implementation specification](../docs/affinity_pipeline_implementation_spec.md) — implementation-scoped specification for the current Boltz-to-Rosetta pipeline.
- [Requirements](../docs/requirements.md): current intended use, system, input, output, functional, and fail-closed requirements.
- [Design specification](../docs/design_specification.md) — architecture, runtime modes, fail-closed boundaries, and component responsibilities.
- [CLI reference](../docs/cli_reference.md) — command-by-command usage, arguments, outputs, and guardrails.
- [Runtime and artifacts reference](../docs/runtime_artifacts_reference.md) — case and pair directory structure, Rosetta outputs, reports, and audit artifacts.

### Thermodynamics and scoring documentation
- [Paired mutation scoring method](../docs/paired_mutation_scoring_method.md) — branch scoring, paired delta construction, calibration gating, and QC logic.
- [Pipeline execution chain](../docs/pipeline_execution_chain.md) — end-to-end state progression from case intake to report emission.

### Setup and operations
- [Setup and install guide](../docs/setup_install_guide.md) — native research setup, Rosetta prerequisites, environment variables, schema export, and test entrypoints.
- [README](../docs/README.md) — repository front door and curated documentation map.

### Flowchart
- [Rendered pipeline flowchart](../docs/pipeline_flowchart.svg) — quick visual workflow view.
- [Editable flowchart source](../docs/pipeline_flowchart.md) — Mermaid source for the workflow map.


















## 4. Scientific workflow actually implemented


















### A. Ingest and provenance
- `src/affinity_pipeline/ingest/boltz_case.py` discovers Boltz provenance, confidence JSONs, structure files, and affinity JSONs.
- The pipeline ranks poses by available confidence metadata and selects the top `k` samples from the run manifest.
- For dual-format samples, `.mmcif` and `.pdb` can be compared before the chosen coordinate source is committed.

### B. Authoritative ligand and Rosetta preparation
- `src/affinity_pipeline/chemistry/ligand_source.py`, `canonicalize.py`, `atom_mapping.py`, and `rosetta_params.py` canonicalize the authoritative ligand, build Rosetta params, and align the authoritative ligand to Boltz coordinates and Rosetta atom naming.
- `complex_builder.py` writes Rosetta-ready complexes with residue `LIG` on chain `X`.

### C. Rosetta rescoring
- `src/affinity_pipeline/rosetta/runner.py` and related protocol/rendering modules run:
  - one score-only baseline protocol
  - a local-refine ensemble with `rosetta_replicates_per_pose` replicates
- Parsed scorefiles yield interface-focused fields such as `interface_delta_X`, `best_interface_delta_X`, `median_interface_delta_X`, and variability metrics.

### D. Pair feature construction and calibration
- `src/affinity_pipeline/aggregation/summarize.py` constructs branch summaries and paired delta features such as `delta_best_interface_delta_X`, `delta_median_interface_delta_X`, `delta_ligand_iptm`, and `delta_affinity_pred_value`.
- `src/affinity_pipeline/calibration/` applies domain checks and optional bundle-backed linear or isotonic calibration.

### E. QC and status assignment
- QC combines Boltz confidence checks, ligand IPTM checks, Rosetta variance, touching-fraction checks, calibration availability, and domain support.
- Result statuses map into `PASS`, `PASS_WITH_WARNINGS`, or `FAIL_NO_RESULT` rather than unsupported best-effort claims.


















## 5. What scientific reviewers should separate


### Native evidence
The native evidence is the deterministic paired workflow itself: qualified manifests, Rosetta score-only/local-refine outputs, branch summaries, paired delta features, and the directly emitted report/audit artifacts. This is the first layer scientific review should treat as authoritative.

### Transformed / calibrated evidence
Domain checks, calibration bundles, QC gates, and the final mutation-effect reporting surface are transformed layers built on top of the native Rosetta- derived paired evidence. These transformed outputs matter only when their approval/gating conditions are actually satisfied.

### Detached analyst artifacts
Planning documents, regulatory-adjacent documents, and analyst-side summaries outside the curated maintained-doc set are detached artifacts for review context, not the authoritative shipped runtime surface. They should be read as supporting material rather than as replacements for code/test/runtime evidence.

### Non-authoritative surfaces
Raw Rosetta terms are not by themselves reportable thermodynamic conclusions. Research-mode evidence and regulated-mode calibrated reporting are distinct layers and should not be collapsed. Governance/planning document breadth does not itself establish runtime scientific authority.


## 6. Current scientific and operational limits


### Scientific limits
Current v1 boundaries remain restricted to protein targets, one non-covalent small-molecule ligand, same ligand identity across a reference/mutant pair, and mutation-effect estimation rather than absolute single-structure affinity. The method is deterministic and conservative by design, but that does not by itself establish experimentally validated thermodynamic authority. Raw Rosetta-derived evidence remains mutation-effect rescoring input rather than a direct physical free-energy claim.

### Operational / runtime limits
Real execution still depends on an installed Rosetta executable, Rosetta database, Rosetta source access for `molfile_to_params.py`, plus `gemmi` and `rdkit`. Missing calibration in regulated modes, unsupported structure comparisons, ambiguous chemistry, and unqualified clinical environments remain fail-closed runtime barriers. The workflow is therefore operationally stricter than a typical best-effort research script.

### Evidence / provenance limits
The repository contains many planning and governance documents, but those do not carry the same authority as current code/tests/curated runtime outputs. Regulated-mode reporting only has meaning when its bundle, domain, and QC conditions actually pass. Scientific review must distinguish native runtime evidence from governance breadth or future-facing documentation.

### Signoff consequence
This pipeline is suitable for governed mutation-effect rescoring and gated calibrated reporting, but it should not be signed off as if Rosetta terms alone constituted validated thermodynamic ground truth.


## 7. Recommended signoff questions


### Scientific validity question
Is the intended scientific claim bounded to mutation-effect prioritization and gated calibrated reporting inside a declared operating domain, rather than a stronger thermodynamic-affinity claim?

### Operational readiness question
Are the current fail-closed chemistry, provenance, calibration, and qualified- environment rules acceptable for the intended `research`/`shadow`/`clinical` operating modes?

### Evidence / provenance question
What evidence threshold is required before calibrated pair outputs are treated as decision-grade in `shadow` or `clinical` mode, and should any Rosetta branch-level features ever become stakeholder-visible?

### Deployment-decision question
Is the current deterministic paired rescoring-and-calibration workflow ready for governed mutation-effect deployment now, or does signoff require stronger runtime evidence before regulated numeric output is trusted?


## 8. Top current clarifications


















- The current method is a paired Boltz-plus-Rosetta rescoring workflow, not a thermodynamic free-energy perturbation method.
- Calibrated output is only justified when an approved calibration bundle is present and the case remains inside the declared operating domain.
- Clinical-mode execution is gated by explicit environment qualification rather than permissive best-effort execution.
- The authoritative ligand is loaded separately from Boltz coordinates and normalized before Rosetta preparation.




## 9. Actual shipped operator surface


















### Top-level CLI
- `PYTHONPATH=src python3 -m affinity_pipeline.cli --help` exposes the current command surface.
- The current subcommands in `src/affinity_pipeline/cli.py` include:
  - `export-schemas`
  - `qualify-env`
  - `ingest-case`
  - `prepare-case`
  - `run-rosetta`
  - `aggregate`
  - `aggregate-pair`
  - `render-report`
  - `run-pair`
  - `run-case`

### Single-case and paired workflow commands
- `run-case` drives the single-case Boltz-to-Rosetta workflow from run manifest to report.
- `run-pair` drives paired reference/mutant workflows and writes a paired `reports/score_report.json` and `reports/score_report.md`.
- `aggregate` and `aggregate-pair` summarize already-run Rosetta outputs with optional calibration bundles.

### Qualification and regulated-mode guards
- `qualify-env` writes an `environment_manifest.json` and returns success only when qualification passes.
- In `clinical` mode, environment qualification is mandatory and enforced before execution.
- In `shadow` mode, qualification is enforced when an environment manifest is provided.

### Output/report contract
- Reports are written under `reports/` as JSON and Markdown.
- Pair runs write paired delta fields and mutation metadata into `reports/score_report.json`.
- Audit events are appended through `src/affinity_pipeline/reporting/audit_log.py`.



## 10. Evidence appendix


















Primary code/tests consulted for this page:
- `/media/joethomas/Backup1/Github/RosettaCommons/src/affinity_pipeline/cli.py`
- `/media/joethomas/Backup1/Github/RosettaCommons/src/affinity_pipeline/ingest/boltz_case.py`
- `/media/joethomas/Backup1/Github/RosettaCommons/src/affinity_pipeline/chemistry/complex_builder.py`
- `/media/joethomas/Backup1/Github/RosettaCommons/src/affinity_pipeline/rosetta/runner.py`
- `/media/joethomas/Backup1/Github/RosettaCommons/src/affinity_pipeline/aggregation/summarize.py`
- `/media/joethomas/Backup1/Github/RosettaCommons/src/affinity_pipeline/aggregation/qc.py`
- `/media/joethomas/Backup1/Github/RosettaCommons/src/affinity_pipeline/calibration/domain_check.py`
- `/media/joethomas/Backup1/Github/RosettaCommons/src/affinity_pipeline/calibration/feature_vector.py`
- `/media/joethomas/Backup1/Github/RosettaCommons/src/affinity_pipeline/models/reports.py`
- `/media/joethomas/Backup1/Github/RosettaCommons/tests/test_pipeline.py`
- `/media/joethomas/Backup1/Github/RosettaCommons/tests/test_execution_guards.py`
- `/media/joethomas/Backup1/Github/RosettaCommons/docs/affinity_pipeline_implementation_spec.md`
- `/media/joethomas/Backup1/Github/RosettaCommons/docs/paired_mutation_scoring_method.md`
