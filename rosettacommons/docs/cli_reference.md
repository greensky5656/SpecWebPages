# CLI Reference

The Affinity Pipeline CLI is currently exposed as a Python module:

```bash
PYTHONPATH=src python3 -m affinity_pipeline.cli --help
```

## Common Notes

- real Rosetta execution requires a real Rosetta binary and database
- `prepare-case`, `run-rosetta`, `run-case`, and `run-pair` accept `--force`,
  but the underlying implementation only permits forced re-runs in
  `research` mode
- case reports are written to `reports/score_report.json`
- pair reports are also written to `reports/score_report.json`, but with the
  pair-report schema

## `export-schemas`

Export the current Pydantic schemas into `configs/schemas/`.

```bash
PYTHONPATH=src python3 -m affinity_pipeline.cli export-schemas
```

Outputs:

- writes the current manifest and report schemas under `configs/schemas/`

## `qualify-env`

Qualify a Rosetta environment for a runtime mode.

```bash
PYTHONPATH=src python3 -m affinity_pipeline.cli qualify-env \
  --rosetta-bin /path/to/rosetta_scripts.default.linuxgccrelease \
  --rosetta-db /path/to/database \
  --mode research
```

Arguments:

- `--rosetta-bin` required
- `--rosetta-db` required
- `--mode` optional, defaults to `research`
- `--require-gpu` optional
- `--write-report` optional Markdown output path

Outputs:

- writes `work/environment_manifest.json`
- optionally writes a Markdown qualification report
- returns `0` on pass and `8` on qualification failure

## `ingest-case`

Ingest one Boltz case into a local work directory.

```bash
PYTHONPATH=src python3 -m affinity_pipeline.cli ingest-case \
  --run-manifest /path/to/run_manifest.json \
  --out /path/to/work_root
```

Arguments:

- `--run-manifest` required
- `--preferred-structure-format` optional override
- `--out` required

Outputs:

- writes `ingest/run_manifest.json`
- writes `ingest/case_manifest.json`
- copies selected Boltz artifacts into `ingest/copied_inputs/`
- prints the created case directory path

## `prepare-case`

Prepare Rosetta-ready artifacts for an ingested case.

```bash
PYTHONPATH=src python3 -m affinity_pipeline.cli prepare-case \
  --case-manifest /path/to/case/ingest/case_manifest.json \
  --rosetta-source-root /path/to/rosetta/source \
  --rosetta-bin /path/to/rosetta_scripts.default.linuxgccrelease \
  --rosetta-db /path/to/database
```

Arguments:

- `--case-manifest` required
- `--rosetta-source-root` optional if set in config
- `--rosetta-bin` optional if set in config
- `--rosetta-db` optional if set in config
- `--force` optional, research-only in practice

Outputs:

- writes `prepared/authoritative_ligand.sdf`
- writes `prepared/LIG.params`
- writes `prepared/complex_rosetta.pdb`
- writes per-pose prepared complexes under `prepared/pose_*/`
- writes `prepared/prep_manifest.json`
- prints the prepared complex path

## `run-rosetta`

Run Rosetta score-only, local-refine, or both for a prepared case.

```bash
PYTHONPATH=src python3 -m affinity_pipeline.cli run-rosetta \
  --case-dir /path/to/case \
  --protocol all \
  --rosetta-bin /path/to/rosetta_scripts.default.linuxgccrelease \
  --rosetta-db /path/to/database
```

Arguments:

- `--case-dir` required
- `--protocol` optional, one of `score_only`, `local_refine`, `all`
- `--rosetta-bin` optional if set in config
- `--rosetta-db` optional if set in config
- `--force` optional, research-only in practice

Outputs:

- writes `rosetta/score_only/pose_*/score.sc`
- writes `rosetta/local_refine/pose_*_rep_*/score.sc`
- writes `rosetta/*/run_manifest.json`
- writes rendered XML under `rosetta/rendered/`

## `aggregate`

Summarize one prepared case into a report.

```bash
PYTHONPATH=src python3 -m affinity_pipeline.cli aggregate \
  --case-dir /path/to/case \
  --calibration-bundle /path/to/bundle
```

Arguments:

- `--case-dir` required
- `--calibration-bundle` optional

Outputs:

- writes `reports/score_report.json`
- prints the JSON report path

## `aggregate-pair`

Summarize an already prepared reference/mutant pair into a pair report.

```bash
PYTHONPATH=src python3 -m affinity_pipeline.cli aggregate-pair \
  --pair-dir /path/to/pair \
  --calibration-bundle /path/to/bundle
```

Arguments:

- `--pair-dir` required
- `--calibration-bundle` optional

Outputs:

- writes `reports/score_report.json` using the pair-report schema
- prints the JSON report path

## `render-report`

Render Markdown from an existing JSON report.

```bash
PYTHONPATH=src python3 -m affinity_pipeline.cli render-report \
  --score-report /path/to/reports/score_report.json
```

Arguments:

- `--score-report` required

Outputs:

- writes `reports/score_report.md`

## `run-pair`

Execute the full paired reference/mutant workflow.

```bash
PYTHONPATH=src python3 -m affinity_pipeline.cli run-pair \
  --run-manifest /path/to/pair_run_manifest.json \
  --out /path/to/work_root \
  --rosetta-bin /path/to/rosetta_scripts.default.linuxgccrelease \
  --rosetta-db /path/to/database \
  --rosetta-source-root /path/to/rosetta/source
```

Arguments:

- `--run-manifest` required
- `--out` required
- `--rosetta-bin` optional if set in config
- `--rosetta-db` optional if set in config
- `--rosetta-source-root` optional if set in config
- `--force` optional, research-only in practice

Behavior:

1. validates the pair run manifest
2. enforces environment qualification for `clinical` and, when present,
   `shadow`
3. ingests both branches
4. prepares both branches
5. runs Rosetta for both branches
6. aggregates paired features
7. renders the Markdown report

Outputs:

- creates `branches/reference/`
- creates `branches/mutant/`
- writes `reports/score_report.json`
- writes `reports/score_report.md`
- appends events to `reports/audit_log.jsonl`

## `run-case`

Execute the full single-case workflow.

```bash
PYTHONPATH=src python3 -m affinity_pipeline.cli run-case \
  --run-manifest /path/to/run_manifest.json \
  --out /path/to/work_root \
  --rosetta-bin /path/to/rosetta_scripts.default.linuxgccrelease \
  --rosetta-db /path/to/database \
  --rosetta-source-root /path/to/rosetta/source
```

Arguments:

- `--run-manifest` required
- `--out` required
- `--rosetta-bin` optional if set in config
- `--rosetta-db` optional if set in config
- `--rosetta-source-root` optional if set in config
- `--force` optional, research-only in practice

Behavior:

1. validates the run manifest
2. enforces environment qualification for `clinical` and, when present,
   `shadow`
3. ingests the case
4. prepares Rosetta-ready inputs
5. runs Rosetta
6. summarizes the case report
7. renders the Markdown report

Outputs:

- writes the full case work tree under `--out/<run_id>/`
- prints the JSON report path

## Related Files

- `configs/example_run_manifest.json`
- `configs/schemas/*.json`
- `docs/runtime_artifacts_reference.md`
- `docs/setup_install_guide.md`
