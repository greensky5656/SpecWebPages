# CLI Reference

The maintained commands are direct Python scripts run from the repository root.
In practice they are usually invoked through the micromamba environment, for
example:

```bash
micromamba run -p .micromamba/envs/amber311 python scripts/run_cif_batch.py ...
```

## Commands

### `scripts/score_boltz_run_minimized.py`

Score one Boltz run with Option A minimization, score-only evaluation, docking,
preflight manifests, and failure summaries.

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/score_boltz_run_minimized.py \
  <boltz_run_dir_or_results_root> \
  --vina-bin ./vina_1.2.7_linux_x86_64
```

Arguments:

- `boltz_run_dir`: run folder, `run-*-cifs/` subfolder, or Boltz results root
- `--out-root`: optional output root override
- `--overwrite`: replace existing outputs for the selected run
- `--vina-bin`: Vina binary path
- `--padding`, `--charge-model`, `--exhaustiveness`, `--num-modes`: docking
  controls
- `--ligand-comp-id`: force the target ligand CCD identifier
- `--boltz-yaml`: Boltz YAML file for ligand or cofactor routing
- `--cofactor-aware` / `--no-cofactor-aware`: retain or ignore non-target
  ligand roster information from Boltz YAML
- `--auto-ligand-routing` / `--no-auto-ligand-routing`: enable or disable
  automatic route selection
- `--aux-comp-ids-file`: optional auxiliary comp-id allowlist
- `--minimize-ph`, `--minimize-backbone-k`, `--minimize-max-iters`,
  `--strict-receptor`: Option A preparation controls
- `--max-cifs`: debug limit on processed CIF count
- logging controls: `--log-level`, `--log-human`, `--log-jsonl`,
  `--log-stdout`, `--log-file`
- `--continue-on-error` / `--no-continue-on-error`: continue processing after a
  model-level failure or stop immediately

Behavior:

- normalizes legacy run folders, `run-*-cifs/` inputs, and Boltz results roots
- classifies each CIF into preflight `run`, `skip`, or runtime `error`
- writes `per_cif_affinities.tsv`, `run_affinity_summary.tsv`,
  `run_affinity_summary.json`, `batch_manifest.tsv`, `batch_results.tsv`,
  `batch_failures.tsv`, and `batch_summary.json`
- writes combined log output, default `pipeline.log`

## `scripts/score_boltz_run.py`

Score one Boltz run without the Option A minimization path.

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/score_boltz_run.py \
  <boltz_run_dir> \
  --vina-bin ./vina_1.2.7_linux_x86_64
```

Arguments:

- `boltz_run_dir`: run folder or `run-*-cifs/` subfolder
- `--out-root`: optional output root, default `vina_boltz_run_scores`
- `--overwrite`: replace existing outputs for the selected run
- `--vina-bin`: Vina binary path
- `--padding`, `--charge-model`, `--exhaustiveness`, `--num-modes`: docking
  controls
- `--ligand-comp-id`: force ligand selection by CCD identifier
- `--ligand-smiles`: provide ligand chemistry directly for non-CCD cases
- `--max-cifs`: debug limit on processed CIF count

Behavior:

- writes `per_cif_affinities.tsv`, `run_affinity_summary.tsv`, and
  `run_affinity_summary.json`
- exits non-zero on contract violations or subprocess failures

## `scripts/run_cif_batch.py`

Run docking across all top-level `p0*` and `p1*` Boltz folders.

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/run_cif_batch.py \
  --out-root vina_batch_test \
  --out-tsv vina_batch_test/vina_batch_summary.tsv \
  --vina-bin ./vina_1.2.7_linux_x86_64
```

Arguments:

- `--out-root`: output root, default `vina_batch_test`
- `--out-tsv`: summary TSV path, default `vina_batch_test/vina_batch_summary.tsv`
- `--overwrite`: replace existing per-run outputs
- `--vina-bin`: Vina binary path
- `--padding`, `--charge-model`, `--exhaustiveness`, `--num-modes`: docking
  controls
- `--max-cifs`: debug limit on processed CIF count per run

Behavior:

- discovers top-level `p0*` and `p1*` folders
- delegates low-level batch work to `run_all_p_runs.py`
- writes `vina_batch_summary.tsv`

## `run_all_p_runs.py`

Low-level batch driver used by `scripts/run_cif_batch.py`.

The primary output row shape is documented by
`specs/schemas/vina_batch_summary_row.schema.json`.

`vina_batch_summary.tsv` contains:

- `boltz_run`
- `cif`
- `ligand_comp_id`
- `affinity_kcal_per_mol`
- `out_dir`
- `ligand_pdbqt`
- `receptor_pdbqt`
- `config`
- `docked_pdbqt`
- `log`

## `scripts/build_docking_features.py`

Parse `vina_batch_summary.tsv`, compute RDKit ligand descriptors, and emit both
per-CIF and aggregate docking feature tables.

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/build_docking_features.py \
  --docking-tsv vina_batch_test/vina_batch_summary.tsv \
  --out-per-cif vina_batch_test/docking_per_cif_features.tsv \
  --out-agg vina_batch_test/docking_agg_features.tsv
```

Arguments:

- `--docking-tsv`: required input docking TSV
- `--out-per-cif`: required output TSV for one row per CIF
- `--out-agg`: required output TSV for one row per `(target_key, ligand_key)`

Behavior:

- normalizes `target_key`
- extracts model index from CIF naming
- computes ligand descriptors from written `ligand.sdf`

## `scripts/parse_platinum.py`

Parse `platinumresults.md` into a structured TSV.

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/parse_platinum.py \
  --in-md platinumresults.md \
  --out-tsv vina_batch_test/platinum_parsed.tsv
```

Arguments:

- `--in-md`: required source markdown
- `--out-tsv`: required parsed TSV path

Behavior:

- normalizes experimental Platinum rows into a tabular modeling dataset
- writes `platinum_parsed.tsv`

## `scripts/build_dataset.py`

Join aggregated docking rows with Platinum rows on `(target_key, ligand_key)`.

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/build_dataset.py \
  --docking-agg vina_batch_test/docking_agg_features.tsv \
  --platinum-tsv vina_batch_test/platinum_parsed.tsv \
  --out-dataset vina_batch_test/training_dataset.tsv
```

Arguments:

- `--docking-agg`: required aggregated docking features TSV
- `--platinum-tsv`: required parsed Platinum TSV
- `--out-dataset`: required joined dataset TSV

Behavior:

- computes duplicate-row experimental summaries when the same key appears more
  than once
- writes `training_dataset.tsv`

## `scripts/train_models.py`

Train ridge calibration models and write `manifest.json` plus one or more model
subdirectories.

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/train_models.py \
  --dataset vina_batch_test/training_dataset.tsv \
  --out-dir vina_batch_test/models \
  --mode global
```

Arguments:

- `--dataset`: required joined training dataset TSV
- `--out-dir`: required model output directory
- `--mode {per-target,global}`: training scope
- `--use-abfe-feature`: include merged ABFE values as an extra feature when
  present

Behavior:

- writes `manifest.json` plus one or more model subdirectories
- each trained model directory contains `model.joblib` and `meta.json`

Training constraints:

- `global` mode requires at least two rows total
- `per-target` mode requires at least two rows per `target_key`

## `scripts/predict_dg.py`

Apply saved model(s) to aggregated docking features.

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/predict_dg.py \
  --docking-agg vina_batch_test/docking_agg_features.tsv \
  --model-dir vina_batch_test/models \
  --out-tsv vina_batch_test/dg_predictions.tsv
```

Arguments:

- `--docking-agg`: required aggregated docking features TSV
- `--model-dir`: required model directory containing `manifest.json`
- `--out-tsv`: required prediction TSV path

Behavior:

- prepends `dg_pred_kcal_per_mol`
- preserves the full aggregated docking row for provenance

## `scripts/run_pipeline_from_cif.py`

Run the six-step maintained workflow:

1. batch docking
2. feature generation
3. Platinum parsing
4. dataset join
5. model training
6. prediction

```bash
micromamba run -p .micromamba/envs/amber311 \
  python scripts/run_pipeline_from_cif.py \
  <cif> \
  --vina-bin ./vina_1.2.7_linux_x86_64 \
  --train-mode global
```

Arguments:

- `cif`: path to a Boltz `*.cif` under a supported run folder
- `--out-root`: output root for generated artifacts
- `--platinum-md`: Platinum markdown source, default `platinumresults.md`
- `--vina-bin`: Vina binary path
- `--overwrite`: replace existing docking outputs
- `--padding`, `--charge-model`, `--exhaustiveness`, `--num-modes`: docking
  controls
- `--train-mode {global,per-target}`: model training scope

Behavior:

- regenerates docking, features, dataset, models, and predictions for the
  supplied CIF context
- prints the matched `target_key`, `ligand_key`, and `dg_pred_kcal_per_mol`

## `scripts/run_abfe.py`

Run the OpenMM ABFE pilot for prepared AMBER inputs.

Arguments:

- `--complex-prmtop`
- `--complex-inpcrd`
- `--solvent-prmtop`
- `--solvent-inpcrd`
- `--ligand-resname`
- restraint geometry arguments: `--r0-A`, `--thetaA0-rad`, `--thetaB0-rad`,
  `--phiA0-rad`, `--phiB0-rad`, `--phiC0-rad`
- restraint force constants: `--kr`, `--kthetaA`, `--kthetaB`, `--kphiA`,
  `--kphiB`, `--kphiC`
- `--out-json`

Common optional controls:

- `--temperature-K`
- `--n-lambdas`
- `--equil-steps`
- `--prod-steps`
- `--sample-interval`
- `--platform`

## `scripts/merge_abfe.py`

Merge an ABFE TSV into a training dataset TSV.

Arguments:

- `--dataset`
- `--abfe`
- `--out`

## `scripts/merge_run_affinity_summaries.py`

Merge multiple per-run `run_affinity_summary.tsv` files into one TSV for
downstream analysis.

Arguments:

- `--root`
- `--out`

## Error Handling

The maintained commands exit non-zero on contract violations, unavailable
tooling, malformed inputs, or subprocess failures. The minimized runner also
persists structured failure rows in `batch_failures.tsv` and summarizes them in
`batch_summary.json`.

## Related Files

- Per-CIF docking executor: `meeko_cif_to_vina.py`
- Batch driver: `run_all_p_runs.py`
- Minimized runner: `scripts/score_boltz_run_minimized.py`
- Feature parser: `affinity/parse_docking.py`
- Platinum parser: `affinity/parse_platinum.py`
- Model trainer: `affinity/train_calibrator.py`
