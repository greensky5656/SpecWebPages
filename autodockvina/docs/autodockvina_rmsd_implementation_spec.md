# AutoDock Vina RMSD Implementation Specification

This file has been refreshed to reflect the current implemented code path.

The canonical maintained narrative specification is now:

- `docs/design_specification.md`

Use that file as the primary implementation-facing document. This file is kept
as an alternate specification entrypoint because older references may still
point here.

## Current Locked Decisions

- The repository is run from source rather than as a packaged Python CLI.
- The primary docking engine is `AutoDock Vina` through `meeko`-prepared inputs.
- Ligand identity is resolved strictly from direct CCD selection or Boltz-YAML
  CCD or SMILES definitions.
- Canonical modeling joins use `(target_key, ligand_key)`.
- The code is fail-fast and does not introduce fallback execution paths.

## Current Runtime Contract

Primary maintained entrypoints:

```bash
python scripts/score_boltz_run_minimized.py <run_or_results_root> --vina-bin <path>
python scripts/run_cif_batch.py --out-root <dir> --out-tsv <tsv> --vina-bin <path>
```

Primary outputs:

- `vina_batch_test/vina_batch_summary.tsv`
- `vina_batch_test/docking_per_cif_features.tsv`
- `vina_batch_test/docking_agg_features.tsv`
- `vina_batch_test/platinum_parsed.tsv`
- `vina_batch_test/training_dataset.tsv`
- `vina_batch_test/models/manifest.json`
- `vina_batch_test/dg_predictions.tsv`
- `vina_boltz_minimized/<run>/per_cif_affinities.tsv`
- `vina_boltz_minimized/<run>/run_affinity_summary.tsv`
- `vina_boltz_minimized/<run>/run_affinity_summary.json`
- `vina_boltz_minimized/<run>/batch_manifest.tsv`
- `vina_boltz_minimized/<run>/batch_results.tsv`
- `vina_boltz_minimized/<run>/batch_failures.tsv`
- `vina_boltz_minimized/<run>/batch_summary.json`

Expected selected model artifacts come from Boltz-style run folders and include:

- `run-*-cifs/*.cif`
- per-model receptor or ligand artifacts written under the selected output root
- optional Boltz YAML metadata for ligand routing and retained cofactor handling

## Current Execution Styles

### Standard Docking

- implemented by `meeko_cif_to_vina.py`, `run_all_p_runs.py`,
  `scripts/run_cif_batch.py`, and `scripts/score_boltz_run.py`
- performs direct CIF parsing, ligand reconstruction, receptor or ligand
  preparation, and Vina docking
- writes per-CIF artifacts and summary TSV outputs

### Minimized Option A

- implemented by `scripts/score_boltz_run_minimized.py` and
  `meeko_cif_to_vina.py`
- adds protein-only preparation and restrained OpenMM minimization
- runs Vina score-only on the retained Boltz pose
- persists deterministic preflight routing, batch manifests, and failure
  classification outputs

### Scoring And Calibration

- implemented by `scripts/build_docking_features.py`,
  `scripts/parse_platinum.py`, `scripts/build_dataset.py`,
  `scripts/train_models.py`, `scripts/predict_dg.py`, and
  `scripts/run_pipeline_from_cif.py`
- converts docking outputs into descriptor tables, training datasets, models,
  and predicted `ΔG`

## Current Validation Rules

- one selected ligand must be resolvable for each docking record
- CCD-native and SMILES-native routes must reconstruct chemically coherent
  ligand graphs
- pose-guard checks must preserve expected geometry between CIF, RDKit, and
  PDBQT representations
- top-level batch discovery is limited to repository-root folders named `p0*`
  or `p1*`
- `global` model training requires at least two rows total
- `per-target` model training requires at least two rows per `target_key`

## Current Provenance And Contract Model

- the maintained row or object schemas live under `specs/schemas/`
- per-model and per-run artifacts preserve the paths needed for reproducibility
- minimized runs emit manifest, result, failure, and summary artifacts even when
  some CIFs are skipped or fail
- prediction rows carry forward the original aggregate docking fields for
  provenance

Maintained schemas currently exist for:

- `specs/schemas/vina_batch_summary_row.schema.json`
- `specs/schemas/run_affinity_summary.schema.json`
- `specs/schemas/dg_predictions_row.schema.json`
- `specs/schemas/model_manifest.schema.json`

## Current Tooling And Environment Model

- Linux usage should override `--vina-bin` explicitly because several scripts
  still default to checked-in macOS Vina paths
- the minimized path requires AmberTools executables plus OpenMM
- the feature and calibration path requires RDKit, scikit-learn, and joblib
- unavailable tooling or malformed chemistry inputs stop the active command

## Related Current Documents

- `docs/design_specification.md`
- `docs/setup_install_guide.md`
- `docs/cli_reference.md`
- `docs/runtime_artifacts_reference.md`
- `docs/per_model_docking_and_minimization_method.md`
- `docs/pipeline_scoring_and_calibration_chain.md`
- `docs/pipeline_flowchart.md`
