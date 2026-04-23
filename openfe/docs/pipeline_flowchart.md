# Pipeline Flowchart

This diagram reflects the current pipeline implementation in `src/pipeline.py`,
`src/fep/pmx_runner.py`, and the current CLI/config behavior.

```mermaid
flowchart TD
    A[WT run root<br/>boltz_results_*] --> B[mutation-rbfe run]
    A2[MUT run root<br/>boltz_results_*] --> B
    C[Protocol YAML<br/>path resolution plus qc_gates merge] --> B

    B --> D[Build paired manifest]
    D --> E[Intake schema validation]
    E --> F[Write provenance input hashes]
    F --> G[Write model selection metadata]
    G --> H[Validate WT to MUT mapping]

    H --> I{Execution mode}
    I -->|research_mode| J[Generate deterministic synthetic<br/>complex and apo leg replicates]
    I -->|production_mode| K[Run external workflow script<br/>for complex and apo legs]

    J --> L[Persist FEP replicate payloads<br/>plus observability]
    K --> L

    L --> M[Summarize complex and apo legs]
    M --> N[Estimate ddG]
    N --> O[Harmonize WT anchor]
    O --> P[Convert to mutant affinity]
    P --> Q[Evaluate QC gates]
    Q --> R[Write signed mutation affinity report]
    R --> S[Validate report schema]
    S --> T[Write software SBOM<br/>state ledger<br/>stage timings]
    T --> U[SUCCESS]

    D -->|intake or selection failure| W
    E -->|schema failure| W
    H --> V[Mapping or applicability failure]
    V --> W[Fail-closed report]
    W --> X[NO_CALL]

    K --> Y[Workflow or infrastructure failure]
    Y --> Z[Fail-closed report]
    Z --> AA[FAILED_INFRA]

    C --> AB[Production requirements]
    AB -->|missing signing key or openfe version pin| W
    S -->|schema failure| W
```

## Notes

- Supported mutation scope is one or more substitutions on the same chain.
- `research_mode` exercises the full pipeline with synthetic leg values.
- `production_mode` delegates leg execution through the configured workflow
  script.
- All terminal outcomes write a report artifact; the pipeline is intentionally
  fail-closed.
