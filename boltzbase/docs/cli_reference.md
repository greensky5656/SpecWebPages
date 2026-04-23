# CLI Reference

## Entry Point

The root package exposes:

```bash
boltz
```

The primary implemented command in the inspected source is:

```bash
boltz predict <INPUT_PATH> [OPTIONS]
```

## Input Path

`<INPUT_PATH>` can currently be:

- a single `.yaml` or `.yml` file
- a single `.fa`, `.fas`, or `.fasta` file
- a directory containing supported files

If a directory is provided, unsupported file types or nested directories cause
validation failures.

## Output Directory Behavior

`--out_dir` is a parent directory, not the final run directory.

For an input named `example.yaml`, the runtime writes to:

```text
<out_dir>/boltz_results_example/
```

This applies even when the input file itself already lives in another
directory.

## Common Commands

### Minimal CPU Run

```bash
boltz predict examples/prot_no_msa.yaml \
  --model boltz2 \
  --accelerator cpu \
  --out_dir ./results
```

### Automatic MSA Generation

```bash
boltz predict examples/multimer.yaml \
  --model boltz2 \
  --use_msa_server \
  --msa_server_url https://api.colabfold.com \
  --out_dir ./results
```

### Override Existing Outputs

```bash
boltz predict your_input.yaml \
  --model boltz2 \
  --out_dir ./results \
  --override
```

### Potentials And Affinity Sampling

```bash
boltz predict your_input.yaml \
  --model boltz2 \
  --use_potentials \
  --use_potentials_affinity \
  --sampling_steps_affinity 200 \
  --diffusion_samples_affinity 5 \
  --out_dir ./results
```

## Important Options

### Model And Hardware

- `--model`: `boltz1` or `boltz2`
- `--accelerator`: `gpu`, `cpu`, or `tpu`
- `--devices`: number of devices
- `--num_workers`: dataloader worker count

### Prediction Sampling

- `--recycling_steps`
- `--sampling_steps`
- `--diffusion_samples`
- `--max_parallel_samples`
- `--step_scale`

### Affinity Options

- `--sampling_steps_affinity`
- `--diffusion_samples_affinity`
- `--affinity_checkpoint`
- `--affinity_mw_correction`
- `--use_potentials_affinity`

### MSA Options

- `--use_msa_server`
- `--msa_server_url`
- `--msa_pairing_strategy`
- `--msa_server_username`
- `--msa_server_password`
- `--api_key_header`
- `--api_key_value`
- `--max_msa_seqs`
- `--subsample_msa`
- `--num_subsampled_msa`

### Output And Reproducibility

- `--out_dir`
- `--output_format`
- `--write_full_pae`
- `--write_full_pde`
- `--write_embeddings`
- `--seed`
- `--override`

### Advanced Options

- `--checkpoint`
- `--method`
- `--preprocessing-threads`
- `--use_potentials`
- `--no_kernels`

## Environment Variables

The CLI reads these environment variables in the current implementation:

- `BOLTZ_CACHE`: absolute cache path override
- `BOLTZ_MSA_USERNAME`: MSA basic-auth username
- `BOLTZ_MSA_PASSWORD`: MSA basic-auth password
- `MSA_API_KEY_VALUE`: MSA API-key value

If both basic-auth credentials and an API key are provided, the runtime raises a
configuration error.

## Input Requirements

Important current behaviors:

- YAML is the preferred format.
- FASTA is still supported but documented as deprecated in `docs/prediction.md`.
- Missing protein MSAs require `--use_msa_server`.
- Affinity outputs are only generated when the input manifest requests
  `properties.affinity`.
- Method conditioning is only supported for Boltz-2.

## Generated Outputs

Key outputs are written under the run root:

- `processed/manifest.json`
- `processed/records/*.json`
- `processed/structures/*.npz`
- `processed/msa/*.npz`
- `predictions/<input_name>/*.cif`
- `predictions/<input_name>/confidence_*.json`
- `predictions/<input_name>/affinity_<input_name>.json`

See `docs/runtime_artifacts_reference.md` for the expanded filesystem layout.

## Cache Behavior

On first use, the CLI downloads:

- Boltz-1 or Boltz-2 structure weights
- Boltz-2 affinity weights when needed
- CCD or molecule assets required for preprocessing

By default the cache path resolves to `~/.boltz`, unless `BOLTZ_CACHE` is set.

## Failure Conditions

Typical hard-failure conditions in the current implementation include:

- unsupported input extensions
- directories found where files were expected
- missing MSA files without `--use_msa_server`
- unsupported `--method` values
- mixed MSA authentication modes
- missing checkpoints or missing cached assets after download failures

## Related Docs

- `docs/prediction.md`
- `docs/runtime_artifacts_reference.md`
- `docs/setup_install_guide.md`
