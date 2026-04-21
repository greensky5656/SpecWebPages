# MDPipeline Specification

Updated: 2026-04-20 21:13:35 PDT

Codebase-grounded scientific review of the actual current shipped MDPipeline surfaces: CLI, one-pose selection policy, preparation/protonation/tooling layers, OpenMM MD, MM/GBSA analysis, output contract, and major scientific limitations.

Repository: `/home/joethomas/Documents/Github/MDPipeline`

This page was refreshed from current code and tests rather than relying on README summaries, which may be stale.

## Actual shipped CLI surface

- `check-openmm`: OpenMM preflight only
- `select`: writes selection/tool/protocol JSONs
- `prepare`: selection + pose validation + prep
- `simulate`: prep + MD
- `analyze`: reuse + trajectory analysis + MM/GBSA
- `full`: end-to-end per-case pipeline

## Selection policy actually implemented

- strictly lowest `score_only_affinity_kcal_per_mol`
- rigid `n_cifs_total = 10` contract
- manifest decision must be `run`
- CIF/model paths reconstructed from `run_dir` + `model_index`

## Preparation, protonation, and parameterization surfaces

- GAFF2/BCC default ligand backend
- optional `openff-sage` backend
- protein protonation: `none` or `propka-pdb2pqr`
- ligand protonation: `none` or deterministic single-state `epik`

## MD and MM/GBSA workflow actually shipped

- OpenMM MD on Amber prmtop/inpcrd
- default production stage = `20.0 ns`
- narrow trajectory analytics
- MM/GBSA through ParmEd/GROMACS/gmx_MMPBSA
- default MM/GBSA skip window = `2.0 ns`

## Output contract and operator evidence

The per-case output contract is strongly structured with selection, protocol, pose validation, prep, MD, trajectory, MM/GBSA, and top-level `run_summary.json` artifacts.

## Current scientific/operator limitations

- single-pose selection
- strict ten-model assumption
- strong external tool dependence
- entropy-off default
- narrow analytics
- chemistry limits such as iodine/GB decomposition and single-state ligand protonation

## Evidence appendix

- `/home/joethomas/Documents/Github/MDPipeline/md_pipeline/cli.py`
- `/home/joethomas/Documents/Github/MDPipeline/md_pipeline/selection.py`
- `/home/joethomas/Documents/Github/MDPipeline/md_pipeline/prep.py`
- `/home/joethomas/Documents/Github/MDPipeline/md_pipeline/protocol.py`
- `/home/joethomas/Documents/Github/MDPipeline/md_pipeline/openmm_runner.py`
- `/home/joethomas/Documents/Github/MDPipeline/md_pipeline/analysis.py`
- `/home/joethomas/Documents/Github/MDPipeline/md_pipeline/gmx_mmgbsa.py`
- `/home/joethomas/Documents/Github/MDPipeline/md_pipeline/protonation.py`
- `/home/joethomas/Documents/Github/MDPipeline/tests/test_cli.py`
- `/home/joethomas/Documents/Github/MDPipeline/tests/test_protocol.py`
- `/home/joethomas/Documents/Github/MDPipeline/tests/test_gmx_mmgbsa.py`
- `/home/joethomas/Documents/Github/MDPipeline/tests/test_prep_protonation.py`
- `/home/joethomas/Documents/Github/MDPipeline/tests/test_selection.py`

## Top current clarifications

- The current shipped CLI is still a per-case six-command surface, not a benchmark or cohort-ranking command layer.
- Selection remains explicitly one-pose and upstream-score-driven via `score_only_affinity_kcal_per_mol`.
- The strongest source-backed caution is operational/tool dependence: the pipeline logic is rich, but scientific execution still depends on many external binaries and optional packages.