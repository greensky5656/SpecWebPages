# Runtime And Artifacts Reference

This document describes the main on-disk outputs produced by the implemented
Affinity Pipeline.

## Case Work Directory

A full single-case run writes a directory rooted at:

```text
<work_root>/<run_id>/
```

Typical structure:

```text
<case_dir>/
  ingest/
    run_manifest.json
    case_manifest.json
    copied_inputs/
  prepared/
    authoritative_ligand.sdf
    LIG.params
    LIG_0001.pdb
    complex_rosetta.pdb
    load_test.sc
    prep_manifest.json
    pose_001/
      complex_rosetta.pdb
    ...
  rosetta/
    rendered/
      boltz_score_only.xml
      boltz_local_refine.xml
    score_only/
      pose_000/
        score.sc
        stdout.log
        stderr.log
        run_manifest.json
    local_refine/
      pose_000_rep_000/
        score.sc
        stdout.log
        stderr.log
        run_manifest.json
      ...
  reports/
    score_report.json
    score_report.md
    audit_log.jsonl
```

## Pair Work Directory

A paired reference/mutant run writes:

```text
<work_root>/<pair_run_id>/
```

Typical structure:

```text
<pair_dir>/
  ingest/
    run_manifest.json
  branches/
    reference/
      ingest/
      prepared/
      rosetta/
      reports/
    mutant/
      ingest/
      prepared/
      rosetta/
      reports/
  reports/
    score_report.json
    score_report.md
    audit_log.jsonl
```

The top-level pair report is a paired schema report containing:

- mutation metadata
- branch summaries for the reference and mutant cases
- paired delta features
- optional calibrated output
- pair-level QC and provenance fields

## Core Manifest Files

### `ingest/run_manifest.json`

The execution request used for the current case or pair run. This records:

- mode
- input case locations
- ligand source
- sample count
- replicate count
- structure-format preference
- config profile
- calibration bundle path
- environment manifest path

### `ingest/case_manifest.json`

Written by the ingest stage. This records:

- Boltz provenance metadata
- selected samples
- selected coordinate source for each sample
- optional alternate coordinate source
- confidence JSON hashes
- cross-format comparison results
- copied affinity JSON metadata
- authoritative ligand source hash

### `prepared/prep_manifest.json`

Written by the chemistry preparation stage. This records:

- the selected coordinate source
- ligand residue and chain identity used for Rosetta
- authoritative ligand summary
- generated Rosetta params artifacts
- authoritative-to-Rosetta atom-name mapping
- authoritative-to-Boltz atom-serial mapping
- prepared complex hash

### `rosetta/*/run_manifest.json`

Written once per Rosetta execution directory. This records:

- protocol name
- input PDB hash
- params path
- XML hash
- seed
- Rosetta executable and database hashes
- command line
- environment hash
- start and finish times
- return code
- stdout and stderr log hashes
- scorefile hash

## Report Files

### `reports/score_report.json`

Single-case reports contain:

- `case_id`
- `mode`
- `result_status`
- operating-domain summary
- Boltz summary fields
- Rosetta baseline summary
- Rosetta refined summary
- per-pose Rosetta summaries
- cross-pose Rosetta metrics
- optional calibrated estimate
- QC flags, warnings, and errors
- provenance hashes

Pair reports contain:

- `pair_id`
- `reference_case_id`
- `mutant_case_id`
- mutation descriptor
- branch-level report sections
- paired delta features
- optional calibrated estimate
- QC and provenance sections

### `reports/score_report.md`

Rendered from the JSON report. The Markdown renderer exposes a readable summary
of:

- case or pair metadata
- operating-domain status
- Boltz summary values
- Rosetta summaries
- paired features when applicable
- calibration section
- QC flags
- provenance summary
- limitations

### `reports/audit_log.jsonl`

The reporting layer appends a hash-linked event log. Each entry includes a
`prev_hash` so the sequence forms an append-only audit chain.

## Prepared Files

### `prepared/authoritative_ligand.sdf`

Canonicalized authoritative ligand exported for Rosetta params generation.

### `prepared/LIG.params`

Rosetta params file generated from the authoritative ligand.

### `prepared/complex_rosetta.pdb`

Prepared Rosetta-ready complex built from the selected Boltz structure, with:

- ligand residue normalized to `LIG`
- ligand chain normalized to `X`
- atom names reconciled to the generated params file
- polymer and retained cofactor chains rewritten deterministically

Per-pose prepared complexes are stored under `prepared/pose_*/`.

## Rosetta Outputs

### `rosetta/rendered/*.xml`

Rendered RosettaScripts XML generated from the checked-in templates in
`configs/rosetta/templates/`.

### `rosetta/score_only/`

Contains one baseline scoring run per selected Boltz pose.

### `rosetta/local_refine/`

Contains one local-refine directory per selected pose and replicate.

Each execution directory normally contains:

- `score.sc`
- `stdout.log`
- `stderr.log`
- `run_manifest.json`

## Environment Qualification Artifact

Environment qualification writes:

```text
work/environment_manifest.json
```

This manifest records:

- operating system
- Python runtime
- locale
- GPU summary
- Rosetta binary and database hashes
- Python package versions
- qualified modes
- qualification status and failure reasons

## Notes

- `work/benchmarks/` contains generated local benchmark runs and should be
  treated as output, not as the canonical documentation source.
- The pipeline always works from copied or generated artifacts inside the case
  work tree so reports can reference hashed local files rather than ephemeral
  source paths alone.
