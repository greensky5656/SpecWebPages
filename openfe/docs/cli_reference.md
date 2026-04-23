# CLI Reference

The package installs the CLI entrypoint:

```bash
mutation-rbfe
```

## Commands

### `run`

Run the WT-to-MUT mutation RBFE pipeline.

```bash
mutation-rbfe run \
  --wt-run /path/to/boltz_results_wt \
  --mut-run /path/to/boltz_results_mut \
  --config configs/openfe/mutation_rbfe_protocol.yaml \
  --out /path/to/output_dir
```

Arguments:

- `--wt-run`: path to the WT `boltz_results_*` root
- `--mut-run`: path to the MUT `boltz_results_*` root
- `--config`: protocol YAML path
- `--out`: output directory

Behavior:

- prints the final report JSON to stdout
- writes the full artifact tree under `--out`
- returns exit code `0` for `SUCCESS` and `NO_CALL`
- returns exit code `1` for `FAILED_INFRA`

## `validate-platinum`

Run PLATINUM holdout validation metrics and release-gate evaluation.

```bash
mutation-rbfe validate-platinum \
  --dataset /path/to/dataset.json \
  --qc-gates configs/openfe/mutation_qc_gates.yaml \
  --out /path/to/validation_summary.json
```

Arguments:

- `--dataset`: validation dataset JSON containing `cases`
- `--qc-gates`: QC gate YAML
- `--out`: output JSON path
- `--bootstrap-resamples`: optional, default `10000`
- `--bootstrap-seed`: optional, default `12345`

Behavior:

- prints the validation payload to stdout
- writes the validation summary JSON to `--out`
- returns exit code `0` only when the payload status is `PASS`

## `monitor`

Summarize report populations and `NO_CALL` rates from generated reports.

```bash
mutation-rbfe monitor \
  --report-dir /path/to/report/root \
  --out /path/to/monitor_summary.json
```

Arguments:

- `--report-dir`: root directory to search for `mutation_affinity_report.json`
- `--out`: output summary JSON path

Behavior:

- prints the summary payload to stdout
- writes the monitor summary to `--out`
- reports totals, status counts, `NO_CALL` rate, and `NO_CALL` breakdowns

## `verify-report`

Verify a signed mutation affinity report.

```bash
mutation-rbfe verify-report \
  --report /path/to/mutation_affinity_report.json \
  --signature /path/to/mutation_affinity_report.signature.json \
  --key your-signing-key
```

Arguments:

- `--report`: report JSON path
- `--signature`: signature JSON path
- `--key`: signing key

Behavior:

- prints a verification payload to stdout
- returns exit code `0` when the signature is valid
- returns exit code `1` when the signature is invalid

## `build-release-bundle`

Build a release bundle containing required configs, schemas, docs, and validation
evidence.

```bash
mutation-rbfe build-release-bundle \
  --repo-root /path/to/OpenFE \
  --bundle-dir /path/to/release_bundle \
  --validation-summary /path/to/validation_summary.json
```

Arguments:

- `--repo-root`: repository root
- `--bundle-dir`: output bundle directory
- `--validation-summary`: validation summary JSON path

Behavior:

- prints the bundle manifest payload to stdout
- copies required files into `--bundle-dir`
- writes `bundle_manifest.json`

## Error Handling

The CLI catches `PipelineError` and emits a JSON payload with:

- `status: FAILED_INFRA`
- an `error` object containing the structured code and message

## Related Files

- CLI entrypoint: `src/cli.py`
- Pipeline orchestration: `src/pipeline.py`
- Research config: `configs/openfe/mutation_rbfe_protocol.yaml`
- Production config: `configs/openfe/mutation_rbfe_protocol.production.yaml`
