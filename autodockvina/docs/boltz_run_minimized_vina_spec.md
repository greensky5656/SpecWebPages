## Goal

Given a **single Boltz run folder** (containing ~10 `*.cif` models), produce:

- **Per-CIF docking affinity** (AutoDock Vina mode‑1 score; Vina reports units as kcal/mol)
- **Per-run aggregated affinity summary** (min/mean/median/std across the CIF models)

This pipeline preserves intermediate artifacts needed for debugging and reproducibility (protein prep, minimization outputs, docking inputs/outputs).

## Scope

- **Input**: Boltz run folder (e.g. `p10721kd_b49_wt_3g0e-c971d8454c63/`) containing ~10 `*.cif` files (possibly nested under `run-*-cifs/`).
- **Ligand**: **single ligand** per CIF, identified by CCD `comp_id` present in the CIF (e.g. `B49`, `IRE`).
- **Docking box**: **ligand envelope + padding** (configurable).
- **Protein protonation**: **locked at preparation stage** (default **pH 7.4**).
- **Ligand charges for docking**: **Gasteiger** (via Meeko) — sufficient for docking / docking score.
- **No MD production**: this pipeline performs **pre-docking minimization only** (no heating/NPT/trajectory output).

## Environment requirements

This pipeline is designed to run via a pinned micromamba environment (no shell activation required).

- **Required micromamba env**: `amber311`
  - Path: `/Users/hansparsons/Github/autodockvina/.micromamba/envs/amber311`
  - Key contents:
    - Python 3.11
    - AmberTools (`pdb4amber`, `cpptraj`, `tleap`, etc.)
    - OpenMM
    - RDKit
    - Meeko 0.7.1
    - gemmi

Run commands with:

```bash
micromamba run -p /Users/hansparsons/Github/autodockvina/.micromamba/envs/amber311 <command>
```

## Primary CLI: score a Boltz run folder

### Command

```bash
micromamba run -p /Users/hansparsons/Github/autodockvina/.micromamba/envs/amber311 \
  python /Users/hansparsons/Github/autodockvina/scripts/score_boltz_run_minimized.py \
  /Users/hansparsons/Github/autodockvina/p10721kd_b49_wt_3g0e-c971d8454c63 \
  --overwrite
```

### Tool components (what was built)

This tool has two layers:

- **Run-folder driver**: `/Users/hansparsons/Github/autodockvina/scripts/score_boltz_run_minimized.py`
  - Finds all `*.cif` files under a single Boltz run directory.
  - Runs the per-CIF pipeline for each CIF.
  - Writes **run-level** outputs (`per_cif_affinities.tsv`, `run_affinity_summary.tsv`).
- **Per-CIF pipeline**: `/Users/hansparsons/Github/autodockvina/meeko_cif_to_vina.py`
  - Converts one Boltz mmCIF model into prepared receptor/ligand + docking config.
  - Optionally runs **protein preparation + restrained minimization** (Option A).
  - Runs AutoDock Vina and parses the **mode‑1 affinity**.

### CLI reference (score_boltz_run_minimized.py)

Required:

- `boltz_run_dir`: path to a Boltz run folder (or its `run-*-cifs/` subfolder).

Key options and defaults:

- `--out-root vina_boltz_minimized`
  - Output root under repo root (if relative).
- `--overwrite`
  - If the output folder `vina_boltz_minimized/<boltz_run_name>/` already exists, the script **refuses to run** unless `--overwrite` is provided.
  - With `--overwrite`, the existing output directory is **deleted** and re-created.
- `--vina-bin /Users/hansparsons/Github/autodockvina/vina_1.2.7_mac_aarch64`
  - Absolute path to the Vina binary (can be overridden).
- Docking box/search:
  - `--padding 6.0` (Å)
  - `--exhaustiveness 8`
  - `--num-modes 9`
- Ligand:
  - `--charge-model gasteiger` (Meeko ligand PDBQT charge model)
  - `--ligand-comp-id <CCD>` (optional: force ligand CCD comp_id rather than auto-detect)
- Minimization (Option A):
  - `--minimize-ph 7.4`
  - `--minimize-backbone-k 50.0` (kcal/mol/Å²)
  - `--minimize-max-iters 200`
- Debug:
  - `--max-cifs N` (if `N > 0`, only process first `N` CIFs)

### Run name / folder mapping

The output run name is derived as:

- If `boltz_run_dir` is the run folder: `boltz_run_name = boltz_run_dir.name`
- If `boltz_run_dir` is a `run-*-cifs/` subfolder: `boltz_run_name = boltz_run_dir.parent.name`

### Outputs

Outputs are written under:

- `vina_boltz_minimized/<boltz_run_name>/`

Files:

- `run_affinity_summary.tsv`
  - One row summarizing min/mean/median/std across CIF models.
- `per_cif_affinities.tsv`
  - One row per CIF model with `affinity_kcal_per_mol`.
- Per-CIF folders:
  - `vina_boltz_minimized/<boltz_run>/<cif_stem>/...`

Each per-CIF folder contains:

- **Input-derived**
  - `ligand_comp_id.txt`
  - `receptor.pdb` (polymer-only receptor extracted from CIF)
- **Protein preparation artifacts (repro/debug)**
  - `receptor_pdb4amber.pdb` (+ related `pdb4amber` artifacts)
  - `cpptraj_prepareforleap.in`
  - `receptor_prepareforleap.pdb`
  - `receptor_tleap.pdb`
  - `receptor.prmtop`
  - `receptor.inpcrd`
  - `receptor_tleap_minimized.pdb` (post‑minimization, written **without hydrogens**)
- **Ligand chemistry artifacts**
  - `ligand.sdf` (CCD-derived bond orders, coordinates assigned from CIF)
- **Docking inputs**
  - `receptor.pdbqt`
  - `ligand.pdbqt`
  - `vina_config.txt`
- **Docking outputs**
  - `docked.pdbqt`
  - `docked.log`
  - `affinity_kcal_per_mol.txt`

### Output file schemas

`per_cif_affinities.tsv` columns:

- `boltz_run`: boltz run folder name
- `cif`: absolute path to CIF input
- `model_index`: parsed from CIF stem `*_model_<N>.cif`
- `ligand_comp_id`: detected/forced CCD comp_id (e.g. `IRE`, `B49`)
- `affinity_kcal_per_mol`: Vina mode‑1 affinity (float; Vina-reported units are kcal/mol)
- `out_dir`: per-CIF output directory path

`run_affinity_summary.tsv` columns:

- `boltz_run`
- `ligand_comp_id`
- `n_models`: number of CIFs processed
- `vina_affinity_min_kcal_per_mol`
- `vina_affinity_mean_kcal_per_mol`
- `vina_affinity_median_kcal_per_mol`
- `vina_affinity_std_kcal_per_mol` (population std across CIF models)

## Pipeline flow summary (per CIF, human-readable)

For each `*_model_<N>.cif` in a Boltz run folder, we do:

### 1) Parse Boltz mmCIF

- **Input**: `BoltzModel.cif`
- **Extract**:
  - **Protein (polymer) atoms** → receptor coordinates
  - **Ligand `comp_id`** (e.g., `B49`, `IRE`)
  - **Ligand 3D coordinates** (used to define the docking box)

### 2) Write the initial receptor PDB (protein-only)

- **Output**: `receptor.pdb`

### 3) Protein preparation + restrained minimization (Option A, protein-only)

- **Goal**: normalize receptor records for LEaP and relieve clashes without running full MD
- **Steps / outputs**:
  - `pdb4amber` normalization → `receptor_pdb4amber.pdb`
  - `cpptraj prepareforleap` cleanup → `cpptraj_prepareforleap.in`, `receptor_prepareforleap.pdb`
  - `tleap` build from prepared receptor → `receptor_tleap.pdb`, `receptor.prmtop`, `receptor.inpcrd`
  - OpenMM restrained minimization from `prmtop/inpcrd` → `receptor_tleap_minimized.pdb` (written **without hydrogens** for Meeko compatibility)

### 4) Ligand chemistry reconstruction (CCD → RDKit) and coordinate transfer

- **Goal**: correct bond orders/aromaticity (do not rely on PDB `CONECT`)
- **Steps / outputs**:
  - Fetch ligand definition from RCSB CCD (`<comp_id>.cif`)
  - Build RDKit ligand with CCD bond orders, then assign CIF coordinates
  - Write `ligand.sdf`

### 5) Prepare docking inputs (Meeko)

- **Outputs**:
  - `ligand.pdbqt` (Gasteiger charges)
  - `receptor.pdbqt` (Meeko polymer templating)

### 6) Define docking box from ligand location

- **Method**: ligand heavy-atom coordinate bounds + padding (default 6 Å)
- **Output**: `vina_config.txt`

### 7) Dock (AutoDock Vina) and record the score

- **Outputs**:
  - `docked.pdbqt`, `docked.log`
  - `affinity_kcal_per_mol.txt` (Vina mode‑1 “affinity”, Vina-reported units: kcal/mol)

After all CIF models in the run finish:

- Write `per_cif_affinities.tsv` (one row per CIF model)
- Write `run_affinity_summary.tsv` (min/mean/median/std across CIF models)

## Key implementation details

### Ligand bonding + charges (feedback-aligned)

- Bond orders/aromaticity are taken from **CCD** (RCSB CCD CIF) and used to build an RDKit molecule.
- Coordinates are assigned from Boltz CIF ligand coordinates.
- Docking charges are assigned by Meeko using **Gasteiger** (`mk_prepare_ligand.py --charge_model gasteiger`).
- We do **not** use PDB `CONECT` for ligand chemistry.

### Protein preparation + minimization (feedback-aligned)

- `pdb4amber` is used to normalize PDB/termini handling.
- `cpptraj prepareforleap` is used to standardize coordinates/layout before LEaP.
- `tleap` generates Amber topology/coordinates from `receptor_prepareforleap.pdb`.
- OpenMM performs restrained minimization (implicit solvent) from `receptor.prmtop` + `receptor.inpcrd`.
- The minimized receptor PDB is written **without hydrogens** to avoid RDKit valence issues during Meeko polymer building.

### Receptor PDBQT via Meeko

- Receptor PDBQT is produced using Meeko’s Polymer-based path (in-process) with strict inter-residue bond filtering (peptide bonds + disulfides only).

## Scientific rationale (practical)

This section explains the scientific intent of each step (what it is doing, and what it is not doing).

### Protein preparation and minimization (Step 3 path)

- **Why**: Boltz-generated structures can have local geometry/pathology that breaks receptor prep; Step 3 applies deterministic cleanup before docking prep.
- **How**: `pdb4amber` normalizes receptor records, `cpptraj prepareforleap` standardizes LEaP-facing coordinates/layout, `tleap` builds Amber topology, and OpenMM runs restrained minimization from Amber topology/coordinates.
- **Important caveat**: this path does not write `set default PBradii mbondi2`; despite that, OpenMM may still emit a non-fatal OBC2 parameter warning depending on topology/radii metadata.

### Restrained OpenMM minimization (Option A, protein-only)

- **Why**: Predicted structures can have local strain/clashes that destabilize docking preparation; minimization provides fast clash relief and improves geometry consistency across models.
- **How**:
  - Normalize with `pdb4amber`, then run `cpptraj prepareforleap`, then build Amber topology with `tleap`.
  - Create an OpenMM system from `receptor.prmtop` + `receptor.inpcrd` with **implicit solvent OBC2**.
  - Apply harmonic positional restraints to heavy atoms with `k` default **50 kcal/mol/Å²** to preserve fold while allowing local relaxation.
  - Run energy minimization (default **200 iterations**).
- **What this is not**: This is not MD equilibration (no heating/NPT/production), and it does not include explicit solvent, ions, or waters. It is a rapid, deterministic geometry cleanup step.

### Ligand chemistry from CCD (and why not CONECT)

- **Why**: Bond orders/aromaticity matter for docking atom typing and partial charges. PDB `CONECT` is frequently incomplete or incorrect for ligands and can break aromaticity perception.
- **How**: The ligand is rebuilt from the RCSB **Chemical Component Dictionary (CCD)** entry for `comp_id`:
  - Use CCD bond table to define bond orders/aromaticity in RDKit.
  - Transfer **coordinates from the CIF** onto that chemically-correct graph.
  - Write `ligand.sdf` and then generate ligand `pdbqt` with Meeko using **Gasteiger** charges.
- **Limitations**: Gasteiger charges are adequate for docking scores but are not a substitute for higher-quality charge models (e.g., AM1-BCC) used in physics-based free energy workflows.

### Docking box definition (ligand envelope + padding)

- The docking box is computed from the ligand’s heavy-atom coordinate bounds in the CIF.
- Padding (default **6 Å**) expands the box to allow pose exploration near the reference ligand location.

### Interpreting Vina scores vs experimental ΔG

- Vina’s “affinity” is an **empirical scoring function** (reported as kcal/mol-like units) designed primarily for **pose ranking and relative scoring**.
- The score is **not** guaranteed to match experimental binding free energies from Platinum. Differences in receptor state (protonation, conformational ensemble, waters/ions) and the scoring function’s approximations can shift absolute values.

## Operational notes: observed failure modes and fixes

This section documents real issues observed during batch runs and how to address them.

### Interactive zsh: `zsh: command not found: #`

Cause: In interactive `zsh`, `#` comments may be treated as commands unless `interactivecomments` is enabled.\n\nFix:\n\n```bash\nsetopt interactivecomments\n```\n\nOr remove comment lines from pasted batch loops.

### CCD download failures (RCSB): `failed to fetch ... <lig>.cif`

Cause: Network/DNS resolution failure when downloading CCD definitions (e.g., `https://files.rcsb.org/ligands/download/IRE.cif`).\n\nFix:\n- Ensure network connectivity/DNS is functional during runs.\n- Recommended improvement (not implemented): local CCD caching/prefetch to remove online dependency.

### OpenMM minimization NaNs: `openmm.OpenMMException: Particle coordinate is NaN`

Cause (scientific): Severe steric clashes or invalid local geometry can cause extremely large forces, leading to numerical instability during minimization.\n\nFix strategy (recommended improvement; not implemented):\n- Add a deterministic pre-min “clash check” (e.g., detect extremely small inter-atomic distances).\n- Use a staged minimization protocol (stronger restraints first, then relax) to reduce NaN risk without skipping models.

## Parameter knobs

- `--padding` (Å): docking box padding.
- `--exhaustiveness`, `--num-modes`: Vina search settings.
- `--minimize-ph` (default 7.4): protein protonation pH.
- `--minimize-backbone-k` (kcal/mol/Å^2): backbone restraint during minimization.
- `--minimize-max-iters`: minimization iteration cap.

## Known limitations

- These are **Vina docking scores**; they are not guaranteed to numerically equal experimental ΔG without calibration or physics-based free energy calculations.
- This spec does not include MD heating/NPT/production, solvent padding, or trajectory unwrapping (MD extension layer).

## Relevant files

- Main per‑CIF pipeline: `/Users/hansparsons/Github/autodockvina/meeko_cif_to_vina.py`
- Run‑folder driver: `/Users/hansparsons/Github/autodockvina/scripts/score_boltz_run_minimized.py`
- Micromamba env: `/Users/hansparsons/Github/autodockvina/.micromamba/envs/amber311`

