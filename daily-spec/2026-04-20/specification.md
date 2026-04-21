# MDPipeline Specification for Scientific Review

Updated: 2026-04-20 21:13:35 PDT

This version is written primarily for a genetics PhD director and a chemical / molecular engineer focused on thermodynamic interpretation. It keeps code evidence, but foregrounds scientific purpose, assumptions, thermodynamic meaning, and signoff implications.

## Executive view for signoff

- Current role: per-case post-processing and ranking workflow.
- Starts from: upstream Boltz + Vina outputs.
- Current scientific commitment: one selected pose per case.
- Current thermodynamic endpoint: MM/GBSA-style relative ranking support.
- Not the same thing as: de novo docking, rigorous absolute free-energy calculation, or cohort-level benchmark-governed decision authority.

## What a genetics director should know

- The workflow starts after upstream structural candidate generation and docking-style scoring already happened.
- The pipeline's main purpose is to add a more physically informed follow-up layer than raw docking scores alone.
- Current shipped operation is still per-case, not a full program-level release/governance framework.
- The biggest strategic scientific simplification is single-pose commitment.

## What a thermodynamics / molecular engineer should know

- The workflow includes physically meaningful steps: explicit-solvent system build, restrained equilibration, production MD, trajectory analysis, and MM/GBSA.
- The endpoint remains approximate: current docs and code support MM/GBSA-style ranking, not rigorous free-energy truth.
- Default entropy mode is `none`.
- Current docs frame the outputs as best suited to relative ranking within comparable series.

## Scientific workflow

```text
Boltz structural candidates + Vina scoring
  → choose one candidate pose
  → verify CIF / receptor / ligand consistency
  → prepare receptor and ligand parameters
  → build solvated protein–ligand system
  → run restrained MD and production MD
  → measure structural stability / contacts
  → compute MM/GBSA-style energy summaries
  → write machine-readable outputs
```

## Pose-selection rule

Grounded in `md_pipeline/selection.py`:
- reads `batch_results.tsv`, `batch_manifest.tsv`, `batch_summary.json`
- chooses the model with minimum `score_only_affinity_kcal_per_mol`
- does not choose by `docking_affinity_kcal_per_mol`
- expects `n_cifs_total = 10`
- fails closed on ambiguity or missing assets

Scientific implication: all downstream thermodynamic interpretation inherits this one-pose structural commitment.

## Thermodynamic interpretation of the workflow

- Pose validation: integrity check, not thermodynamic validation.
- Preparation: defines protonation- and force-field-sensitive starting assumptions.
- Solvated build: moves into a physically richer simulation context.
- Restrained equilibration: relaxes around the selected state but does not prove global correctness.
- Production MD: samples local dynamics around the selected hypothesis.
- Trajectory metrics: supportive structural evidence.
- MM/GBSA: tractable energetic approximation, best treated comparatively.

Current default protocol facts from `ProtocolConfig`:
- production stage = `20.0 ns`
- solvent models allowed = `gb`, `pb`
- default solvent models = `("gb",)`
- entropy modes allowed = `none`, `c2`, `ie`
- default entropy mode = `none`
- default MM/GBSA skip window = `2.0 ns`

## Input and output evidence

### Inputs
- `predictions/<boltz_run>/<boltz_run>_model_*.cif`
- `vina_analysis/<boltz_run>/batch_results.tsv`
- `vina_analysis/<boltz_run>/batch_manifest.tsv`
- `vina_analysis/<boltz_run>/batch_summary.json`
- `vina_analysis/<boltz_run>/<boltz_run>_model_N/receptor.pdb`
- `vina_analysis/<boltz_run>/<boltz_run>_model_N/ligand.sdf`

### Outputs a reviewer should inspect
- `selected_model.json`
- `protocol.json`
- `pose_validation.json`
- `prep_summary.json`
- `md_summary.json`
- `analysis/trajectory/trajectory_summary.json`
- `analysis/mmgbsa/mmgbsa_summary.json`
- `run_summary.json`

## Current scientific limits

- Single-pose workflow
- Approximate MM/GBSA endpoint
- Per-case shipped CLI surface
- Strong external-tool dependence
- Host readiness incomplete on this machine at generation time

## Operational dependencies observed missing on this host

- `propka3`: MISSING
- `pdb2pqr`: MISSING
- `epik`: MISSING
- `structconvert`: MISSING
- `gmx_MMPBSA`: MISSING
- `mpirun`: MISSING
- `gmx`: MISSING
- `pdb4amber`: MISSING
- `cpptraj`: MISSING
- `tleap`: MISSING
- `antechamber`: MISSING
- `parmchk2`: MISSING

## Recommended signoff questions

### Genetics director
- Is one selected structural hypothesis acceptable before follow-up physics begins?
- Is MM/GBSA-based ranking support sufficient for the biological decisions in scope?
- Do you require benchmarked cohort-level confidence before deployment?

### Thermodynamics / molecular engineering review
- Is 20 ns production adequate for the systems of interest?
- Is entropy-off MM/GBSA acceptable as the default output layer?
- Should multi-pose or replicate averaging be required before signoff?
- Are protonation and parameterization assumptions explicit enough?

## Evidence appendix

- `/home/joethomas/Documents/Github/MDPipeline/MDPIPELINE_SCIENTIST_GUIDE.md`
- `/home/joethomas/Documents/Github/MDPipeline/MDPIPELINE_ONE_PAGE_WORKFLOW.md`
- `/home/joethomas/Documents/Github/MDPipeline/MDPIPELINE_FLOW_DIAGRAM.md`
- `/home/joethomas/Documents/Github/MDPipeline/md_pipeline/cli.py`
- `/home/joethomas/Documents/Github/MDPipeline/md_pipeline/protocol.py`
- `/home/joethomas/Documents/Github/MDPipeline/md_pipeline/selection.py`
- `/home/joethomas/Documents/Github/MDPipeline/md_pipeline/gmx_mmgbsa.py`
