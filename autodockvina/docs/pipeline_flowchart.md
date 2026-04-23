# Pipeline Flowchart

This diagram reflects the current implemented flow centered on
`scripts/score_boltz_run_minimized.py`, `run_all_p_runs.py`,
`meeko_cif_to_vina.py`, and the `affinity/` calibration modules.

```mermaid
flowchart TD
    A[Top-level p0 or p1 folders<br/>single run folder<br/>or Boltz results root] --> B{Choose entrypoint}
    Y[Optional Boltz YAML<br/>ligand and cofactor metadata] --> B
    Z[platinumresults.md] --> P

    B -->|Batch docking| C[run_all_p_runs.py<br/>or scripts/run_cif_batch.py]
    B -->|Single-run direct| D[scripts/score_boltz_run.py]
    B -->|Single-run minimized| E[scripts/score_boltz_run_minimized.py]
    B -->|End-to-end wrapper| F[scripts/run_pipeline_from_cif.py]

    C --> G[meeko_cif_to_vina.py per CIF]
    D --> G
    F --> C

    E --> H[Preflight ligand routing]
    H -->|run| I[Option A receptor prep<br/>and restrained minimization]
    H -->|skip| J[Write batch_manifest.tsv<br/>and batch_summary.json]
    I --> G

    G --> K[Rebuild ligand chemistry<br/>apply RMSD guards]
    K --> L[Vina score-only or docking]
    L --> M[Write per-CIF artifacts]

    M -->|batch path| N[vina_batch_summary.tsv]
    M -->|single-run path| O[per_cif_affinities.tsv<br/>run_affinity_summary.tsv]
    O --> Q[Optional downstream analysis]

    N --> P[scripts/build_docking_features.py]
    P --> R[docking_per_cif_features.tsv]
    P --> S[docking_agg_features.tsv]
    Z --> T[scripts/parse_platinum.py]
    T --> U[platinum_parsed.tsv]
    S --> V[scripts/build_dataset.py]
    U --> V
    V --> W[training_dataset.tsv]
    W --> X[scripts/train_models.py]
    X --> AA[models/manifest.json<br/>model.joblib<br/>meta.json]
    S --> AB[scripts/predict_dg.py]
    AA --> AB
    AB --> AC[dg_predictions.tsv]
```

## Notes

- Top-level batch discovery is limited to repository-root folders named `p0*`
  and `p1*`.
- The minimized runner classifies each CIF as `run`, `skip`, or runtime
  `error`.
- Ligand chemistry and pose consistency are enforced before Vina execution.
- The calibration chain joins aggregate docking rows to Platinum data on
  `(target_key, ligand_key)`.
- The code is intentionally fail-fast and does not introduce fallback routes
  when required chemistry, routing, or tooling is missing.
