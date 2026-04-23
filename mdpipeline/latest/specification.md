# MDPipeline Scientific Specification


















Updated: 2026-04-21 09:03:47 PST

Repository: `/home/joethomas/Documents/Github/MDPipeline`

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
MDPipeline is a single-case, single-pose downstream MD + endpoint MM/GBSA workflow built on one selected Boltz/Vina pose and one prepared Amber/OpenMM system.

### Strongest justified claim
Its strongest justified claim is governed comparative ranking support within a related series when a plausible bound pose has already been accepted.

### Main limiting factor
The main limiting factor is that the shipped workflow remains single-pose, single-trajectory, and endpoint-based rather than ensemble or alchemical in its thermodynamic evidence class.

### Appropriate present use
The appropriate present use is triage/ranking support, not stand-alone formal free-energy prediction or benchmark-grade campaign authority.








## 2. The stages of the pipeline from a thermodynamic view







- **Stage 1 — Pose commitment:** the pipeline chooses one Boltz/Vina pose and commits the thermodynamic interpretation to that single structure rather than to a pose ensemble.
- **Stage 2 — Prepared endpoint state:** receptor, ligand, and optional protonation/preparation steps define one simulation-ready bound endpoint state for downstream MD.
- **Stage 3 — Bound-state relaxation and endpoint readout:** restrained minimization, equilibration, and production OpenMM MD sample local fluctuations around that selected bound state, then MM/GBSA reads out an endpoint free-energy approximation over the sampled trajectory window.
- **Stage 4 — Ranking interpretation:** the resulting quantity is used as a comparative ranking proxy within related cases, not as a formal alchemical ΔΔG or a fully population-weighted thermodynamic estimate.
- **Thermodynamic boundary:** no apo leg, no explicit mutation cycle, no pose-population integration, and no replica-derived uncertainty are part of the present shipped contract.







## 3. Supporting documentation


















### Core documentation
- [Implementation specification](../docs/mdpipeline_implementation_spec.md) — current workflow contract and scientific assumptions.
- [Requirements](../docs/requirements.md): current intended use, system, input, output, functional, and fail-closed requirements.
- [Design specification](../docs/design_specification.md) — implemented architecture and runtime constraints.
- [CLI reference](../docs/cli_reference.md) — commands, defaults, and operator-facing arguments.
- [Runtime artifacts reference](../docs/runtime_artifacts_reference.md) — output tree and stage artifacts.

### Scientist-oriented documentation
- [Scientist guide](../docs/scientist_guide.md) — scientific interpretation, defaults, and caveats.
- [One-page workflow](../docs/one_page_workflow.md) — concise end-to-end workflow summary.

### Setup and operations
- [Setup and install guide](../docs/setup_install_guide.md) — external tool requirements and OpenMM preflight.

### Flowchart
- [Rendered pipeline flowchart](../docs/pipeline_flowchart.svg) — quick visual workflow view.
- [Editable flowchart source](../docs/pipeline_flowchart.md) — Mermaid source for the workflow map.


















## 4. Scientific workflow actually implemented


















1. **Select one pose.** The code reconstructs local model paths from `model_index` rather than trusting stale paths in TSV rows, validates manifest/summary consistency, and carries forward only one winning model.
2. **Validate bound assets.** Pose-validation logic checks that the selected CIF, receptor PDB, ligand SDF, and optional retained cofactors remain mutually consistent before prep starts.
3. **Prepare receptor and ligand.** Prep symlinks/copies source receptor/ligand files, measures ligand formal charge from RDKit, and prepares Amber-compatible receptor files with `pdb4amber` and `cpptraj`.
4. **Optional protein protonation.** If enabled, the pipeline:
   - runs `propka3` at operator-selected pH;
   - runs `pdb2pqr` with the validated chain flag;
   - parses pKa rows and writes a flagged-residue TSV for residues within the configured uncertainty window;
   - maps PDB2PQR output back to Amber-compatible residue names such as `HID/HIE/HIP`, `ASH`, `GLH`, `LYN`, `CYM`, `TYM`;
   - writes a mapping trace TSV and protonation summary JSON;
   - runs a `tleap` preflight on the renamed receptor and hard-fails if the renamed receptor is not acceptable to Amber.
5. **Optional ligand protonation.** If enabled, the pipeline:
   - converts SDF to Maestro with `structconvert`;
   - runs `epik` for a **single selected state** only;
   - converts back to SDF;
   - validates that exactly one molecule is returned, that 3D coordinates are present, and that heavy-atom identity/order and chiral tags are preserved;
   - records ligand protonation provenance and tool/version metadata.
6. **Parameterize and solvate.** The code supports default GAFF2/BCC preparation and optional `openff-sage`. Retained cofactors are templated separately and carried into the final build. Ion addition in the GAFF2 `tleap` path now uses `addions`, not `addionsrand`.
7. **Run staged OpenMM MD.** The code performs minimization, restrained relaxation, then production NPT, writing both `production.dcd` and `production.xtc`, plus per-stage CSV state reports and final state XML.
8. **Run narrow trajectory analysis.** Analysis computes protein-backbone RMSD, ligand-heavy RMSD, pocket-heavy RMSD, ligand-protein heavy-atom contact counts/minimum distances, and protein-ligand hydrogen-bond occupancy/event counts from the production XTC.
9. **Run endpoint MM/GBSA.** The code converts Amber outputs through ParmEd into GROMACS artifacts, cleans/fits/recenters the trajectory with GROMACS, writes `mmpbsa.in`, runs `gmx_MMPBSA`, optionally runs decomposition passes, and parses GB/PB totals plus primary terms into JSON.
10. **Enforce one specific chemistry guard.** If GB decomposition is requested on an iodine-containing system, the code now fails early with `mmgbsa-error:unsupported-gb-decomposition-iodine` rather than letting `gmx_MMPBSA/sander` fail later on `bad atom type: i`.


















## 5. What scientific reviewers should separate


### Native evidence
The native evidence is the shipped single-run workflow itself: pose selection, pose validation, prepared-system artifacts, OpenMM trajectory outputs, MM/GBSA summaries, and the final `run_summary.json`. This layer is the direct product of the current codepath and is the first thing scientific review should read.

### Transformed / calibrated evidence
MDPipeline does not currently ship a separate native calibration layer that upgrades its endpoint outputs into a formally calibrated affinity authority. Any interpretation of MM/GBSA values as ranking support remains a workflow- level scientific reading of the native outputs, not a separate validated calibration product.

### Detached analyst artifacts
One-off reports, repository-root notes, and exploratory markdown analyses are detached analyst artifacts unless they are explicitly promoted into the maintained docs set. They may be useful context, but they are not the authoritative description of the shipped workflow.

### Non-authoritative surfaces
Intended future benchmark engines, campaign-management ideas, protonation- ensemble aspirations, and publication-ready free-energy claims are not current authoritative surfaces. Upstream selection-score logic is also distinct from downstream MD/MM/GBSA evidence; downstream physics does not retroactively make pose selection exhaustive.


## 6. Current scientific and operational limits


### Scientific limits
The current workflow remains a single-pose, single-trajectory endpoint MM/GBSA pipeline rather than an RBFE/FEP/TI method. Protein and ligand protonation handling are still single-state operational choices rather than ensemble/population thermodynamics. Downstream MD/MM/GBSA evidence cannot recover alternative poses that were not selected upstream.

### Operational / runtime limits
Successful execution still depends on a large external toolchain including AmberTools, OpenMM, ParmEd, GROMACS, `gmx_MMPBSA`, RDKit, MDAnalysis, Gemmi, and optionally PROPKA3/PDB2PQR/Epik/Schrödinger conversion tooling. Ligand protonation remains operationally gated by Schrödinger tool availability and licensing. A known chemistry/tool incompatibility still blocks GB decomposition for some iodine-containing systems because downstream tooling rejects the GAFF2 lowercase atom type `i`.

### Evidence / provenance limits
Major repository docs still describe older workflow assumptions and do not fully match the current protonation/provenance surface. Some current tested behaviors, such as first-minimum-wins tie handling and partial-success batch acceptance, are stricter or different from older prose. Scientific signoff should therefore privilege current code/tests/runtime artifacts over legacy descriptive markdown.

### Signoff consequence
This pipeline is suitable for governed comparative ranking support when one plausible pose is already accepted, but it should not be signed off as a rigorous ensemble thermodynamics or benchmark-ready campaign engine.


## 7. Recommended signoff questions


### Scientific validity question
Is the intended scientific claim explicitly limited to single-pose comparative ranking support from endpoint MD + MM/GBSA, rather than rigorous ensemble or alchemical thermodynamics?

### Operational readiness question
Are the current runtime assumptions acceptable for routine use, including the external-tool stack, CUDA/precision expectations, and optional protonation / licensed-tool dependencies?

### Evidence / provenance question
Should the current code-and-test-backed 20 ns / 2–20 ns / protonation-enabled behavior now be treated as the committed surface even where older docs still lag behind it?

### Deployment-decision question
Is the present fail-fast, single-pose workflow ready for governed deployment as a ranking-support tool, or should multi-pose / multi-replica escalation be required before broader signoff?


## 8. Top current clarifications


















- Current code remains a **per-case six-command CLI**, not a campaign-management or benchmark-emission layer.
- Selection is still **one pose only** and is driven by `score_only_affinity_kcal_per_mol`, not by downstream MM/GBSA or consensus scoring.
- The most important code-vs-doc drift is still the move from **6 ns / 2–6 ns** narratives in docs to **20 ns / 2–20 ns** defaults in code.
- Protonation is now a real shipped surface in current code/tests, but it is still **deterministic single-state assignment**, not protonation ensemble science.
- The strongest operational caution remains dependence on external chemistry binaries and, for ligand protonation, Schrödinger tool availability/license state.




## 9. Actual shipped operator surface


















- CLI commands shipped in `md_pipeline/cli.py`: `check-openmm`, `select`, `prepare`, `simulate`, `analyze`, `full`.
- Selection remains one-pose only: `select_best_model()` reads `batch_results.tsv`, `batch_manifest.tsv`, and `batch_summary.json`, then chooses the **lowest** `score_only_affinity_kcal_per_mol`; ties fall to the first minimum encountered in the TSV.
- The selection contract is still strongly shaped by current Boltz/Vina artifacts:
  - expects the `vina_analysis/<boltz_run>/...` layout;
  - expects `batch_summary.json` to report `n_cifs_total == 10`;
  - allows partial success as long as `n_ok >= 1`;
  - can infer retained cofactors from `receptor_with_cofactors.pdb` when the TSV field is blank.
- Prep/operator flags now include explicit tool overrides for `propka3`, `pdb2pqr`, `epik`, and `structconvert`, in addition to AmberTools, GROMACS, `gmx_MMPBSA`, and MPI command paths.
- Protonation surface now exposed to operators:
  - `--protein-protonation-method {none,propka-pdb2pqr}`
  - `--protein-protonation-ph`
  - `--protein-protonation-flag-window`
  - `--protein-protonation-pdb2pqr-chain-flag {keep-chain,chain}`
  - `--ligand-protonation-method {none,epik}`
  - `--ligand-protonation-ph`
  - `--ligand-protonation-pht`
  - `--ligand-protonation-max-states` with a validated **phase-one hard requirement of exactly 1** when `epik` is used.
- Ligand parameterization surface remains:
  - default backend `gaff2`
  - optional backend `openff-sage`
  - default OpenFF file `openff_unconstrained-2.2.1.offxml` when that backend is chosen.
- MD protocol defaults currently exposed:
  - NVT heavy restraints: `1.0 ns`, `1000 kJ/mol/nm^2`
  - NPT heavy restraints: `0.5 ns`, `250 kJ/mol/nm^2`
  - NPT backbone strong: `0.75 ns`, `100 kJ/mol/nm^2`
  - NPT backbone light: `0.75 ns`, `50 kJ/mol/nm^2`
  - production NPT: **`20.0 ns`**, unrestrained
  - timestep: `2 fs`
  - reporting interval: `10 ps`
  - temperature: `300 K`
  - pressure: `1 bar`
- OpenMM platform/operator behavior currently shipped:
  - `auto` preference order is `CUDA`, then `OpenCL`, then `CPU`;
  - CUDA default precision is **double**;
  - CUDA runs set `DeterministicForces=true`;
  - if `md_random_seed` is provided, stage-specific integrator and barostat seeds are derived deterministically.
- MM/GBSA surface currently shipped:
  - solvent models `gb`, `pb`, or `gb,pb`
  - entropy modes `none`, `c2`, `ie`
  - time-based default window: skip first `2.0 ns`, so default analysis uses **2–20 ns** of the 20 ns production segment
  - optional frame-based window via `--no-mmgbsa-skip-initial-ns`
  - optional per-residue and pairwise decomposition
  - optional MPI execution via `--mmgbsa-mpi-procs`
- Top-level outputs written by `full` include `selected_model.json`, `tool_config.json`, `protocol.json`, `pose_validation.json`, `prep_summary.json`, `md_summary.json`, trajectory analysis outputs, MM/GBSA summaries, and `run_summary.json`. Current prep summaries and run summaries also serialize protonation artifacts and protonation status fields.



## 10. Evidence appendix


















**Priority code/test files reviewed**

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

**Focused verification performed**

- `python -m pytest -q tests/test_cli.py tests/test_protocol.py tests/test_gmx_mmgbsa.py tests/test_prep_protonation.py tests/test_selection.py`
- Result: **41 passed**.

**Working-tree observations used for materiality judgment**

- The currently observed scientific and operator surface matches the prior reviewed MDPipeline state in all material signoff points.
- Therefore the current page continues to describe the same material scientific and operator surface.
