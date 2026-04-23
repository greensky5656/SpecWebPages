# Design Specification

This document describes the current implemented architecture of the
`md_pipeline` repository.

## Design Goals

The present codebase is organized around one strict end-to-end objective:

- start from one `boltz_results_*` directory,
- select one best-scoring pose,
- verify the pose assets,
- build one explicit-solvent simulation system,
- run one defined OpenMM protocol,
- analyze that trajectory,
- write machine-readable outputs for downstream inspection.

The design favors explicit contracts, structured outputs, and fail-fast
behavior over permissive heuristics.

## Implemented Architecture

The main package is `md_pipeline/`.

Primary module responsibilities:

- `md_pipeline.cli`: command-line parser, orchestration, output-root selection,
  and final summary assembly
- `md_pipeline.selection`: model selection from the Boltz/Vina batch metadata
- `md_pipeline.pose`: structural consistency checks across CIF, receptor, and
  ligand assets
- `md_pipeline.prep`: receptor preparation, ligand parameterization, solvation,
  and topology/coordinate generation
- `md_pipeline.protonation`: optional protein and ligand protonation workflows
- `md_pipeline.protocol`: protocol and tool configuration dataclasses plus
  default schedules and validation rules
- `md_pipeline.openmm_runner`: OpenMM stage execution and MD summary generation
- `md_pipeline.analysis`: trajectory analysis metrics
- `md_pipeline.gmx_mmgbsa`: GROMACS conversion, trajectory conditioning, and
  gmx_MMPBSA execution
- `md_pipeline.utils`: path handling, logging, command helpers, and JSON I/O

## Command Model

The CLI is module-based:

```bash
python -m md_pipeline <command> ...
```

Supported commands:

- `check-openmm`
- `select`
- `prepare`
- `simulate`
- `analyze`
- `full`

`full` is the default command when none is provided.

The commands are layered rather than independent:

- `select` writes selection and protocol metadata only
- `prepare` runs selection, validation, and system preparation
- `simulate` runs preparation and MD
- `analyze` reuses existing `prep_summary.json` and `md_summary.json`
- `full` performs the entire workflow

## Output Root Design

When `--output-root` is not supplied, the CLI resolves the output directory to:

```text
<run_dir>/openmm_md/<boltz_run>/<model_name>/
```

This design keeps outputs colocated with the source run while separating stage
artifacts from the original Boltz/Vina tree.

The CLI writes:

- `selected_model.json`
- `tool_config.json`
- `protocol.json`

before later stages run, so the effective execution contract is captured even if
subsequent work fails.

## Data Flow

The current data flow is:

1. `select_best_model(run_dir)` identifies the chosen model and its paths.
2. `validate_pose_assets(...)` verifies the chosen pose assets.
3. `prepare_complex(...)` builds the receptor-ligand simulation system.
4. `run_md_protocol(...)` executes OpenMM minimization and MD stages.
5. `analyze_trajectory(...)` computes stability metrics from the production
   trajectory.
6. `run_mmgbsa(...)` prepares GROMACS-compatible files and runs MM/GBSA.
7. `run_summary.json` is assembled from all stage outputs.

The `analyze` command is a deliberate reuse mode that loads:

- `prep_summary.json`
- `md_summary.json`

and resumes from those artifacts instead of rebuilding the system or rerunning
MD.

## Protocol Configuration Design

`md_pipeline.protocol.ProtocolConfig` captures scientific and operational
defaults in a validated dataclass.

It defines:

- force fields
- solvent and ionic settings
- temperature and pressure
- random seeds
- MM/GBSA settings
- protonation settings
- stage schedule

The default stage schedule is encoded through `DEFAULT_RESTRAINT_SCHEDULE` and
materialized into `StageConfig` objects.

Current default stages:

- `nvt_heavy`
- `npt_heavy`
- `npt_backbone_strong`
- `npt_backbone_light`
- `production`

Validation is performed during dataclass initialization so unsupported values are
rejected before runtime-heavy stages begin.

## Tool Resolution Design

`md_pipeline.utils.infer_tool_config(...)` and `ToolConfig` define how external
tools are resolved.

The design supports:

- a default tool lookup based on the active Python environment,
- a shared override directory via `--tool-bin-dir`,
- explicit per-tool overrides through CLI flags.

This keeps the orchestration code independent from hardcoded local paths.

## Protonation Design

Protonation is opt-in and explicit.

Protein protonation:

- supported mode: `propka-pdb2pqr`
- default mode: `none`

Ligand protonation:

- supported mode: `epik`
- default mode: `none`

The code validates protonation-specific arguments eagerly. In particular, Epik
use is constrained to deterministic single-state selection in the current phase.

## Runtime Constraints

Current runtime constraints reflected in code include:

- `--run-dir` is required for every command except `check-openmm`
- at least one MM/GBSA solvent model must be provided
- only `gb` and `pb` solvent models are accepted
- only `none`, `c2`, and `ie` entropy modes are accepted
- random seeds must be positive integers when provided
- protonation pH values must be within valid bounds
- `ligand_protonation_max_states` must be `1` when Epik is used

These are validated before long-running work begins.

## Failure Model

The design is intentionally strict:

- missing required inputs raise errors
- inconsistent summaries loaded for reuse raise errors
- unsupported config values raise errors
- missing external tool dependencies for enabled stages raise errors
- tied best-model selection raises errors

The architecture does not introduce alternate execution branches to rescue bad
inputs or incompatible runtime states.

## Testability

The repository is structured around separable modules with focused tests in
`tests/`.

This design supports testing of:

- parser defaults and CLI argument handling,
- protocol validation,
- selection invariants,
- preparation/protonation contracts,
- MM/GBSA configuration behavior,
- utility-layer helpers.
