# Paired Mutation Scoring Method

This document explains the method implemented in the current Affinity Pipeline.
It is a paired Boltz-plus-Rosetta rescoring workflow, not a thermodynamic free
energy perturbation method.

## Method Summary

The pipeline computes mutation-effect evidence by combining:

1. qualified Boltz confidence and affinity metadata
2. Rosetta score-only interface terms for selected poses
3. Rosetta local-refine replicate ensembles for selected poses
4. branch-level summaries for the reference and mutant cases
5. paired delta features derived from those branch summaries

Optional calibrated output is then applied only when:

- the mode allows calibration
- an approved bundle is provided
- the case remains inside the declared operating domain

## Step 1: Select Boltz Poses

During ingest, the pipeline:

- discovers confidence JSON files and structure files
- ranks poses by the available confidence metadata
- selects the top `k` samples from the run manifest
- preserves both the selected coordinate source and any alternate source

For dual-format samples, `.mmcif` and `.pdb` can be compared before the chosen
coordinate source is committed into the case manifest.

## Step 2: Prepare Rosetta-Ready Inputs

The Rosetta stage does not consume the raw Boltz structure directly. The
pipeline first:

- canonicalizes the authoritative ligand chemistry
- generates Rosetta params artifacts
- maps authoritative ligand atoms to the Boltz ligand atoms
- maps authoritative ligand atoms to Rosetta atom names
- builds `complex_rosetta.pdb` with residue `LIG` on chain `X`

This preparation step is what makes the ligand coordinates and Rosetta params
consistent enough for deterministic scoring.

## Step 3: Score Each Branch

For each selected pose, the pipeline runs:

- one score-only baseline protocol
- a local-refine ensemble with `rosetta_replicates_per_pose` replicates

The score parser extracts interface-focused values from each `score.sc` file.
These are summarized into:

- baseline branch summary
- refined branch summary
- per-pose summaries
- cross-pose disagreement metrics

The key implemented Rosetta summary fields include:

- `interface_delta_X`
- `best_interface_delta_X`
- `median_interface_delta_X`
- `mad_interface_delta_X`
- `touching_fraction`
- `non_touching_replicate_count`

## Step 4: Build Pair Features

For paired reference/mutant workflows, the pipeline computes deltas between the
mutant and reference branches. The current pair report includes:

- `delta_best_interface_delta_X`
- `delta_median_interface_delta_X`
- `delta_ligand_iptm`
- `delta_affinity_pred_value`
- `combined_uncertainty`

These paired deltas are the mutation-effect features that drive downstream
calibration.

## Step 5: Domain Checks

Before calibrated output is considered, the pipeline checks whether the case is
inside the operating domain. The current checks include:

- ligand size
- rotatable bond count
- confidence score
- ligand IPTM

When a calibration bundle is present, the bundle domain definition becomes the
active domain gate. Without a bundle, research mode still produces evidence, but
regulated modes treat missing calibration as fail-closed.

## Step 6: Calibration

Calibration is optional and bundle-backed. The implemented bundle loaders
support:

- linear bundle prediction
- isotonic bundle prediction

Calibration is only attempted when:

- the current mode is not `research`
- a bundle path exists
- the bundle is approved for the current mode
- the case is still inside the bundle domain

If these conditions fail, the report preserves the reason codes instead of
emitting an unsupported numeric result.

## Step 7: QC And Result Status

The QC layer combines:

- Boltz confidence checks
- ligand IPTM checks
- complex IPDE checks
- ligand size and flexibility checks
- Rosetta variance checks
- touching-fraction checks
- calibration availability
- domain support

These are mapped into the implemented result statuses:

- `PASS`
- `PASS_WITH_WARNINGS`
- `FAIL_NO_RESULT`

## Important Scientific Boundary

The implemented method does not claim that raw Rosetta scores are experimental
affinity values. Rosetta terms remain intermediate rescoring features. The
reportable paired mutation-effect endpoint is only justified when a validated
calibration bundle is present and approved for the current operating domain.

## Related Files

- `src/affinity_pipeline/aggregation/summarize.py`
- `src/affinity_pipeline/aggregation/qc.py`
- `src/affinity_pipeline/calibration/domain_check.py`
- `src/affinity_pipeline/calibration/feature_vector.py`
- `src/affinity_pipeline/models/reports.py`
