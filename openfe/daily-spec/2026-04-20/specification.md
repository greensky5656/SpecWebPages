# pmx/GROMACS Mutation RBFE Pipeline Specification

Updated: 2026-04-20 21:13:35 PST

Codebase-grounded scientific review of the currently shipped full pmx/GROMACS mutation-RBFE pipeline surface: CLI, orchestration, fail-closed gating, runtime/reporting layers, the true pmx/GROMACS compute lane, and major scientific limitations.

Repository: `/home/joethomas/Documents/Github/OpenFE`

Authority policy: this page is grounded primarily in current source code, tests, and shipped output-contract logic. README/docs were treated as secondary context only.

## Executive view for signoff

<p>OpenFE is currently code-shipped as a fail-closed mutation-RBFE orchestrator around WT/MUT paired inputs. Its real current surface is not just “a future clinical RBFE idea”: it has a real CLI, a deterministic pipeline ledger, QC gates, report/signature emission, and a release-bundle layer. The main signoff question is not whether it has structure, but whether the current shipped scientific and runtime surfaces are sufficient for the level of mutation-affinity authority you want.</p>

## Actual shipped operator surface

- `mutation-rbfe run`: WT/MUT paired pipeline run; returns 0 for `SUCCESS` or `NO_CALL`
- `validate-platinum`: holdout validation metrics and gating
- `monitor`: report-tree monitoring
- `verify-report`: HMAC signature verification
- `build-release-bundle`: release artifact assembly

## Scientific workflow actually implemented

1. Paired WT/MUT manifest build and schema validation
2. Model-selection metadata emission
3. Mutation mapping validation
4. Two mandatory mutation legs: complex and apo
5. Convergence/statistical aggregation into ddG
6. WT-anchor harmonization and mutant-affinity conversion
7. QC evaluation with fail-closed statusing
8. Report writing, signing, SBOM/provenance emission

## What scientific reviewers should separate

- Research mode: synthetic CI/dry-run behavior, not physical pmx/GROMACS evidence
- Production mode: depends on external workflow scripts and runtime environment
- ddG inference: mean(complex) − mean(apo) with sigma from leg stddevs
- Report layer: signed report/provenance is distinct from runtime validity

## Current scientific and operational limits

- Mutation scope is narrow: substitutions only, maximum two sites
- Applicability-domain logic is simple rule filtering
- Convergence logic is limited to stddev + bootstrap CI width
- Production realism is environment-dependent
- Release/test drift exists

## Recommended signoff questions

### For genetics leadership
- Is the current fail-closed WT/MUT pairing and mutation-scope restriction acceptable?
- Do you require stronger evidence than current research-mode and report surfaces before signoff?

### For thermodynamics review
- Is the current ddG inference and convergence logic scientifically sufficient?
- Should runtime-governance and post-run evidence be strengthened before treating outputs as rigorous authority?

## Evidence appendix

- `/home/joethomas/Documents/Github/OpenFE/src/cli.py`
- `/home/joethomas/Documents/Github/OpenFE/src/pipeline.py`
- `/home/joethomas/Documents/Github/OpenFE/src/intake/pair_builder.py`
- `/home/joethomas/Documents/Github/OpenFE/src/mutation/mapping_validator.py`
- `/home/joethomas/Documents/Github/OpenFE/src/fep/pmx_runner.py`
- `/home/joethomas/Documents/Github/OpenFE/src/fep/convergence.py`
- `/home/joethomas/Documents/Github/OpenFE/src/inference/ddg_estimator.py`
- `/home/joethomas/Documents/Github/OpenFE/src/calibration/anchor_harmonizer.py`
- `/home/joethomas/Documents/Github/OpenFE/src/calibration/affinity_converter.py`
- `/home/joethomas/Documents/Github/OpenFE/src/qc/gate_engine.py`
- `/home/joethomas/Documents/Github/OpenFE/src/reporting/report_writer.py`
- `/home/joethomas/Documents/Github/OpenFE/src/reporting/signing.py`
- `/home/joethomas/Documents/Github/OpenFE/src/release/bundle.py`
- `/home/joethomas/Documents/Github/OpenFE/scripts/run_production_batch.py`
- `/home/joethomas/Documents/Github/OpenFE/tests/test_pipeline.py`
- `/home/joethomas/Documents/Github/OpenFE/tests/test_validation_and_release.py`

## Top current clarifications

- OpenFE is code-shipped as a fail-closed mutation-RBFE orchestrator with five real CLI commands, not just a single run surface.
- Research-mode SUCCESS is synthetic and should not be interpreted as physical pmx/GROMACS evidence.
- There is meaningful code/test drift around release-bundle expectations, so release-surface interpretation should be cautious.