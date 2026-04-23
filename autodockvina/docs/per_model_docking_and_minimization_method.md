# Per-Model Docking And Minimization Method

This document describes the literal per-model method used by the current
codebase.

The implementation sources for this method are:

- `meeko_cif_to_vina.py`
- `scripts/score_boltz_run_minimized.py`

This is a **per-CIF** method. It applies separately to each Boltz model.

## Scope

This document is specifically about how one Boltz CIF becomes receptor and
ligand artifacts, optional minimized receptor state, Vina evaluations, and
per-model affinity outputs.

It does not describe:

- how per-CIF rows are aggregated into run-level summaries
- how docking features are aggregated by `(target_key, ligand_key)`
- how Platinum joins, model training, and prediction export are performed

Those higher-level steps belong to the pipeline-level scoring and calibration
chain.

## Stage 1: Parse The CIF And Resolve The Ligand Route

The code reads the Boltz CIF and determines how the target ligand should be
reconstructed.

What happens:

- parse protein atoms, ligand coordinates, and non-polymer component identifiers
- honor a forced `--ligand-comp-id` when provided
- otherwise choose a CCD-native or Boltz-YAML SMILES-native route
- optionally preserve retained cofactor identifiers when the minimized runner is
  in cofactor-aware mode

Method meaning:

- this fixes the exact ligand identity and chemistry definition that the rest of
  the docking path will use

## Stage 2: Rebuild Ligand Chemistry And Write Base Artifacts

The code reconstructs the ligand graph and writes the initial receptor and
ligand files.

What happens:

- write a protein-only receptor PDB
- write a ligand SDF with chemically coherent bonding
- derive ligand chemistry from CCD definitions or Boltz-YAML SMILES rather than
  trusting PDB-style `CONECT` records

Method meaning:

- this creates the chemically defined receptor or ligand inputs needed for
  subsequent RMSD checks and docking preparation

## Stage 3: Apply RMSD Guard Checks

The code verifies that ligand reconstruction preserves the intended pose.

What happens:

- compute CIF-to-RDKit heavy-atom RMSD
- compute RDKit-to-PDBQT heavy-atom RMSD
- write `boltz_pose_rmsd_report.txt`

Method meaning:

- this ensures the accepted ligand route preserves the expected geometry before
  Vina is run

## Stage 4: Optionally Run Protein-Only Minimization

The minimized path performs a deliberately narrow receptor cleanup stage.

What happens:

- normalize the receptor with `pdb4amber`
- run `cpptraj prepareforleap`
- build topology with `tleap`
- run restrained OpenMM minimization

Method meaning:

- this supplies a lightly relaxed protein-only receptor before docking; it is
  not a full MD equilibration workflow

## Stage 5: Prepare PDBQT Inputs And The Search Box

The code converts the accepted receptor and ligand artifacts into Vina-ready
inputs.

What happens:

- prepare receptor and ligand PDBQT files through Meeko
- build the Vina search box from ligand heavy-atom bounds plus configurable
  padding
- write `vina_config.txt` and related setup artifacts

Method meaning:

- this defines the concrete docking configuration for the current model

## Stage 6: Run Score-Only And Docking

The code evaluates the Boltz pose and then runs docking or redocking.

What happens:

- in the minimized path, run Vina score-only on the retained Boltz pose
- run Vina docking or redocking
- parse affinity values from Vina outputs

Method meaning:

- this produces the model-level score-only and docking observables used by the
  downstream summaries

## Stage 7: Write Per-Model Artifacts

The code persists the full model-local artifact set.

What happens:

- write `affinity_kcal_per_mol.txt`
- write receptor, ligand, PDBQT, and docking log artifacts
- write per-model output directories referenced by run-level or batch-level TSVs

Method meaning:

- this creates the reproducible per-model evidence consumed by later pipeline
  stages

## Important Mode Note

This method is shared across direct scoring and the minimized runner. The
minimized runner adds preflight classification, logging, score-only evaluation,
and batch summary artifacts around the same core per-model docking path.

## Where This Lives In Code

- `meeko_cif_to_vina.py`
- `scripts/score_boltz_run.py`
- `scripts/score_boltz_run_minimized.py`

## Practical Summary

If someone asks, "what are the literal per-model stages?", the answer in the
current implementation is:

1. parse the CIF and resolve the ligand route
2. rebuild ligand chemistry and write base artifacts
3. apply RMSD guard checks
4. optionally run protein-only minimization
5. prepare PDBQT inputs and the search box
6. run score-only and docking
7. write per-model artifacts

## Related Documents

- `docs/pipeline_scoring_and_calibration_chain.md`
- `docs/runtime_artifacts_reference.md`
- `docs/design_specification.md`
- `docs/pipeline_flowchart.md`
