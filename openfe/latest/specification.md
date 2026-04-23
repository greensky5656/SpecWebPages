# pmx/GROMACS Mutation RBFE Pipeline Specification


















Updated: 2026-04-21 09:03:19 PST

Repository: `/home/joethomas/Documents/Github/OpenFE`

This review document is grounded in current code, tests, and inspected artifacts, with repository documentation used only as supporting context where it matches the shipped implementation.

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
OpenFE is a fail-closed WT/MUT mutation-RBFE operator with paired intake, complex/apo mutation legs, aggregation, calibration, QC gating, signed reporting, and release-bundle surfaces.

### Strongest justified claim
Its strongest justified claim is that it is the most thermodynamically direct mutation-effect architecture in the set, with a real RBFE-style design and a materially shipped operator surface.

### Main limiting factor
The main limiting factor is that the expanded dual-site source/runtime surface is ahead of the strongest demonstrated end-to-end production physical evidence.

### Appropriate present use
The appropriate present use is governed mutation-effect evaluation, with signoff bounded to currently demonstrated mutation classes and runtime proof rather than the broadest apparent source capability.









## 2. The stages of the pipeline from a thermodynamic view








- **Stage 1 — Paired state definition:** the workflow defines a WT/reference and mutant system with matched ligand/cofactor context so the thermodynamic comparison is a mutation cycle rather than an isolated score.
- **Stage 2 — Complex mutation leg setup:** the bound complex is prepared for the pmx/GROMACS mutation protocol as the bound-state half of the thermodynamic cycle.
- **Stage 3 — Complex mutation leg execution:** the complex leg is executed to estimate the free-energy effect of the mutation in the bound state.
- **Stage 4 — Apo mutation leg setup:** the corresponding unbound/apo system is prepared as the second half of the thermodynamic comparison.
- **Stage 5 — Apo mutation leg execution:** the apo leg is executed to estimate the free-energy effect of the mutation outside the binding interaction.
- **Stage 6 — ΔΔG construction and reporting translation:** complex and apo leg results are combined into a mutation ΔΔG, then WT anchor harmonization and calibration translate that mutation-effect estimate into the mutant-affinity reporting surface when all gating conditions pass.
- **Thermodynamic boundary:** in `production_mode` this is a real RBFE-style thermodynamic architecture; in `research_mode` the same stages are exercised structurally but the leg values are synthetic and should not be read as physical evidence.








## 3. Supporting documentation


















### Core documentation
- [Mutation RBFE implementation specification](../docs/mutation_rbfe_implementation_spec.md) — current implementation specification.
- [Requirements](../docs/requirements.md): current intended use, system, input, output, functional, and fail-closed requirements.
- [Design specification](../docs/design_specification.md) — architecture, execution modes, and runtime contract.
- [CLI reference](../docs/cli_reference.md) — command-by-command CLI behavior.
- [Runtime artifacts reference](../docs/runtime_artifacts_reference.md) — output tree, provenance, and fail-closed artifacts.

### Thermodynamics documentation
- [Five-stage leg thermodynamic method](../docs/five_stage_leg_thermodynamic_method.md) — literal thermodynamic method for a real mutation leg.
- [Pipeline thermodynamic chain](../docs/pipeline_thermodynamic_chain.md) — whole-pipeline path from complex/apo legs to final mutant affinity.

### Setup and operations
- [Setup and install guide](../docs/setup_install_guide.md) — editable install and production prerequisites.

### Flowchart
- [Rendered pipeline flowchart](../docs/pipeline_flowchart.svg) — quick visual workflow view.
- [Editable flowchart source](../docs/pipeline_flowchart.md) — Mermaid source for the workflow map.


















## 4. Scientific workflow actually implemented

1. **Paired intake and model selection**
   - `src/intake/pair_builder.py` locates one WT run and one MUT run, parses run YAML, confirms binder identity, computes ligand chemistry hashes, selects paired structural models, and writes `manifests/paired_manifest.json`.
   - The pipeline records selection metadata and input hashes before simulation.
2. **Mutation mapping actually enforced**
   - `src/mutation/mapping_validator.py` compares WT and MUT protein sequences, rejects zero-difference cases, rejects WT/MUT ligand hash mismatches, rejects retained-cofactor mismatches when cofactors are in scope, and rejects substitutions spanning multiple chains.
   - Current executable truth is therefore: same-ligand WT/MUT protein mutation handling, with same-chain multi-site substitutions accepted by current mapping/precheck code rather than a strict hard cap of two substitutions at the mapper level.
3. **FEP leg execution**
   - `src/fep/pmx_runner.py` runs complex and apo legs.
   - In `research_mode`, it produces deterministic synthetic leg energies for CI/dry-run testing only.
   - In `production_mode`, it requires pmx, GROMACS, a persisted paired manifest, and an external workflow script, then launches external leg jobs and stores replicate payloads under `fep/complex_leg`, `fep/apo_leg`, and `fep/replicate_observability.json`.
4. **Inference, calibration, and QC**
   - `src/pipeline.py` summarizes leg convergence, estimates ΔΔG, harmonizes WT anchor records, converts to mutant affinity terms, and runs QC.
   - `src/qc/gate_engine.py` evaluates structural RMSD, replicate variability, ΔΔG CI width, convergence flags, ligand heavy-atom applicability bounds, mutation classes, and mutation multiplicity classes.
5. **Reporting and provenance**
   - `src/reporting/report_writer.py` emits a signed `mutation_affinity_report.json` plus signature, including mutation structure, uncertainty intervals, anchor information, software/hardware provenance, and `stage_timings_path`.
   - `src/pipeline.py` also persists `logs/state_ledger.json`, `logs/stage_timings.json`, provenance hashes, and signed fallback reports on fail-closed paths.
6. **Release companion surfaces**
   - `src/release/bundle.py` packages schemas plus the current implementation/design/setup/CLI/runtime-artifact/flowchart document set into the release bundle payload.

## 5. What scientific reviewers should separate



### Native evidence
The native evidence is the shipped WT/MUT mutation-RBFE workflow surface: paired intake artifacts, complex/apo leg outputs, `inference/ddg_result.json`, QC outcomes, signed reports, and provenance/monitoring artifacts. In `production_mode`, this is the physically meaningful runtime layer that scientific review should privilege.

### Transformed / calibrated evidence
WT anchor harmonization, mutant-affinity translation, QC gating, and final signed reporting are transformed layers built on top of the native mutation-leg evidence. These transformed outputs matter operationally, but they should be read as downstream interpretation layers on top of the leg-level thermodynamic core.

### Detached analyst artifacts
Ad hoc summaries, validation memos, and analyst-side readings that are not the current shipped CLI/report surface are detached artifacts. They may explain the system, but they do not replace direct inspection of the executable runtime artifacts.

### Non-authoritative surfaces
`research_mode` synthetic values are not authoritative physical evidence. Signed artifacts are not, by themselves, proof of scientific correctness. Config/docs statements that are not actually consumed by current source should not outrank executable truth.



## 6. Current scientific and operational limits



### Scientific limits
Supported mutation multiplicity is currently bounded to at most two substitutions, and dual-site support is restricted to same-chain mutations. The workflow is a same-ligand protein-mutation RBFE architecture rather than a ligand-transformation FEP platform. `research_mode` remains synthetic by design and is not scientific evidence for physical ΔΔG performance.

### Operational / runtime limits
Production execution still depends on pmx, GROMACS, external workflow scripts, a pinned `execution.openfe_version`, and a non-default signing key. Observed dual-site production evidence remains fail-closed rather than broadly proven successful, with inspected cases still failing inside pmx hybrid- topology generation. Release completeness now depends on a broader synchronized documentation and packaging surface in addition to code.

### Evidence / provenance limits
Config/docs language can still lag executable truth; some older wording still implies single-residue-only behavior even though source/tests/schemas now support bounded dual-site handling. Signed artifacts improve integrity and auditability but do not, by themselves, establish scientific correctness. Scientific review should prioritize inspected runtime artifacts over config or prose expectations when they differ.

### Signoff consequence
This pipeline is the strongest thermodynamic architecture in the set for mutation-effect estimation, but signoff should remain bounded to the currently demonstrated mutation classes and production evidence rather than the full apparent source surface.



## 7. Recommended signoff questions



### Scientific validity question
Do you approve bounded same-chain dual-site handling as part of the claimed scientific surface, or should scientific validity remain limited to the more strongly demonstrated mutation classes?

### Operational readiness question
Is the current fail-closed production stack operationally ready, given its dependence on pmx/GROMACS, workflow-script execution, signing keys, and the still-fragile observed dual-site production path?

### Evidence / provenance question
What evidence threshold is required before dual-site support is treated as scientifically established: source/tests/schemas only, one successful end-to-end production case, or a broader validation set?

### Deployment-decision question
Should dual-site support remain visible in the production-facing surface now, or be explicitly gated/qualified until stronger runtime evidence is in hand?



## 8. Top current clarifications


















- OpenFE is already a concrete fail-closed operator surface with five CLI commands; it is not just a future roadmap statement.
- The code/test surface now supports bounded same-chain dual-site mutation handling, but this should not be confused with demonstrated physical dual-site production success.
- Production-oriented intent is mutation-RBFE with pmx/GROMACS execution, WT anchor calibration, QC gating, signed reporting, and release packaging; shipped runtime surfaces still depend heavily on external environment readiness.
- Research-mode outputs are intentionally synthetic and should only be treated as pipeline-contract evidence.
- A key current drift point is that config/docs language still references single-residue-only behavior while source/tests now allow bounded dual-site cases.
- Stage timings and replicate observability are now part of the real shipped post-run surface and materially improve reviewability even when runs fail closed.





## 9. Actual shipped operator surface


















- `python -m src.cli run --wt-run --mut-run --config --out` is the main execution command. It returns exit code 0 for `SUCCESS` or `NO_CALL`, and nonzero on failing conditions such as `FAILED_INFRA`.
- `python -m src.cli validate-platinum --dataset --qc-gates --out` computes holdout metrics and release-gate status.
- `python -m src.cli monitor --report-dir --out` summarizes report outcomes for operational monitoring.
- `python -m src.cli verify-report --report --signature --key` verifies report signatures.
- `python -m src.cli build-release-bundle --repo-root --bundle-dir --validation-summary` assembles a required artifact bundle.
- Current manifest/report/schema surfaces now include structured mutation metadata:
  - `mutation.descriptor`
  - `mutation.class`
  - `mutation.multiplicity_class`
  - `mutation.site_count`
  - `mutation.uses_dual_site_hybrid`
  - per-site arrays with chain, residue index, WT residue, MUT residue, and site class
- `scripts/run_production_batch.py` is a real companion owner surface for production-style batching. It no longer only screens for single mutations; it now preclassifies supported one-site/two-site same-chain cases and emits skip statuses such as `SKIPPED_UNSUPPORTED_MUTATION_COUNT` and `SKIPPED_AMBIGUOUS_MULTI_SITE_MAPPING`.
- The fail-closed output contract remains central: controlled `PipelineError` paths emit signed fallback reports with `NO_CALL` or `FAILED_INFRA`, and unhandled exceptions are also converted into signed fallback artifacts.




## 10. Evidence appendix


















Primary code/tests consulted for this page:

- `/home/joethomas/Documents/Github/OpenFE/src/cli.py`
- `/home/joethomas/Documents/Github/OpenFE/src/pipeline.py`
- `/home/joethomas/Documents/Github/OpenFE/src/types.py`
- `/home/joethomas/Documents/Github/OpenFE/src/intake/pair_builder.py`
- `/home/joethomas/Documents/Github/OpenFE/src/mutation/mapping_validator.py`
- `/home/joethomas/Documents/Github/OpenFE/src/fep/pmx_runner.py`
- `/home/joethomas/Documents/Github/OpenFE/src/qc/gate_engine.py`
- `/home/joethomas/Documents/Github/OpenFE/src/reporting/report_writer.py`
- `/home/joethomas/Documents/Github/OpenFE/src/release/bundle.py`
- `/home/joethomas/Documents/Github/OpenFE/scripts/run_production_batch.py`
- `/home/joethomas/Documents/Github/OpenFE/configs/openfe/mutation_qc_gates.yaml`
- `/home/joethomas/Documents/Github/OpenFE/specs/schemas/boltz_mutation_pair_intake.schema.json`
- `/home/joethomas/Documents/Github/OpenFE/specs/schemas/mutation_affinity_report.schema.json`
- `/home/joethomas/Documents/Github/OpenFE/tests/test_pipeline.py`
- `/home/joethomas/Documents/Github/OpenFE/tests/test_validation_and_release.py`
- `/home/joethomas/Documents/Github/OpenFE/tests/test_run_production_batch.py`
- `/home/joethomas/Documents/Github/OpenFE/out_dual_site_zst_validation/reports/mutation_affinity_report.json`
- `/home/joethomas/Documents/Github/OpenFE/out_dual_site_zst_validation/logs/stage_timings.json`
- `/home/joethomas/Documents/Github/OpenFE/out_production_zst_dual_site/reports/mutation_affinity_report.json`
- `/home/joethomas/Documents/Github/OpenFE/out_production_zst_dual_site/run.log`

Git and validation evidence used for materiality decision:

- Previous recorded head: `67e2015a33a1607e7457a19aa419e525039c0a18`
- Current head reviewed: `b019bc13fc09f01d6f5b195db99505497fac2696`
- Latest commit line: `2026-04-21 05:06:56 -0700 b019bc13fc09f01d6f5b195db99505497fac2696 updated doc`
- Targeted regression suite run used for this page: `python -m pytest -q tests/test_pipeline.py tests/test_validation_and_release.py tests/test_run_production_batch.py`
- Result: `10 passed in 0.07s`

Material-change judgment:

- **Scientific conclusion:** the shipped scientific and operator surface changed in code, schemas, reports, QC applicability handling, batch prechecks, and persisted observability artifacts.
