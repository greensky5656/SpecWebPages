# Boltzbase Affinity Platform Specification

















This review document is grounded in current code, tests, inspected artifacts, and maintained repository documentation where that documentation matches the shipped platform surface.

## Table of contents

1. [Executive summary](#1-executive-summary)
2. [The stages of the pipeline from a thermodynamic view](#2-the-stages-of-the-pipeline-from-a-thermodynamic-view)
3. [Supporting documentation](#3-supporting-documentation)
4. [Scientific workflow actually implemented](#4-scientific-workflow-actually-implemented)
5. [What scientific reviewers should separate](#5-what-scientific-reviewers-should-separate)
6. [Current scientific and operational limits](#6-current-scientific-and-operational-limits)
7. [Recommended signoff questions](#7-recommended-signoff-questions)
8. [Top current clarifications](#8-top-current-clarifications)
9. [Actual shipped operator surface](#9-actual-shipped-operator-surface)
10. [Evidence appendix](#10-evidence-appendix)

## 1. Executive summary








### What this pipeline is
BoltzBase is a combined native inference platform with two main surfaces: the `boltz` CLI and the Affinitas web application for queued prediction, monitoring, and download.

### Strongest justified claim
Its strongest justified claim is native structure-plus-affinity model-based screening/triage through the shipped `boltz predict` and UI-backed run surfaces.

### Main limiting factor
The main limiting factor is that affinity outputs remain learned model signals plus writer-level conversions rather than explicit physical thermodynamic simulation.

### Appropriate present use
The appropriate present use is governed screening and model-based affinity signal generation, while detached evaluation utilities remain outside the main approved product evidence surface.








## 2. The stages of the pipeline from a thermodynamic view







- **Stage 1 — Learned structural state generation:** the model first produces a structure/path prediction for the biomolecular system rather than simulating a thermodynamic cycle.
- **Stage 2 — Affinity-head invocation:** when the input explicitly requests affinity properties, the Boltz-2 affinity writer produces model-based affinity outputs alongside the structure outputs.
- **Stage 3 — Writer-level semantic conversion:** native affinity quantities are exposed in model-output units and convenience conversions such as IC50 and approximate pKd / ΔG representations.
- **Stage 4 — Platform surfacing and downstream views:** CLI and Affinitas UI layers package those learned outputs into run directories, queue records, analytics, and optional downstream analyses such as RMSD/TM-score/Vina views.
- **Thermodynamic boundary:** the pipeline is a learned structure-plus-affinity inference system, not RBFE, not MM/GBSA, and not an explicit molecular thermodynamics engine.







## 3. Supporting documentation

















### Core documentation
- [Boltzbase platform implementation specification](../docs/boltzbase_platform_implementation_spec.md) — current implemented platform scope, runtime boundaries, and major subsystems.
- [Requirements](../docs/requirements.md): current intended use, system, input, output, functional, and fail-closed requirements.
- [Design specification](../docs/design_specification.md) — architecture, queue model, execution flow, storage, and operational invariants.
- [CLI reference](../docs/cli_reference.md) — `boltz predict` usage, important options, and examples.
- [Web API reference](../docs/web_api_reference.md) — main Next.js API routes for run submission, queue inspection, analytics, and downloads.
- [Runtime and artifacts reference](../docs/runtime_artifacts_reference.md) — run directory structure, generated files, logs, database state, and exported CSV outputs.

### Model and evaluation documentation
- [Prediction guide](../docs/prediction.md) — detailed Boltz input schema, constraints, affinity properties, and model outputs.
- [Training guide](../docs/training.md) — current training and preprocessing notes.
- [Evaluation guide](../docs/evaluation.md) — benchmark and evaluation notes.

### Setup and operations
- [Setup and install guide](../docs/setup_install_guide.md) — Python environment, editable install, UI bootstrap, and local development setup.

### Flowchart
- [Rendered pipeline flowchart](../docs/pipeline_flowchart.svg) — quick visual workflow view.
- [Editable flowchart source](../docs/pipeline_flowchart.md) — Mermaid source for the end-to-end platform map.

















## 4. Scientific workflow actually implemented

















### A. Preprocess and validate target definitions

The prediction path:
1. expands cache and output dirs (`main.py:1119-1144`),
2. downloads required Boltz assets (`main.py:1146-1153`),
3. validates/collects inputs (`main.py:1155-1156`),
4. parses FASTA/YAML into processed records and structures (`main.py:1168-1187`),
5. loads `processed/manifest.json` (`main.py:1188-1189`).

Schema-level affinity rules (`schema.py:1078-1257`):
- affinity only for Boltz-2,
- exactly one affinity ligand chain total,
- binder must refer to an existing ligand chain,
- affinity on ligands with multiple copies is rejected,
- affinity on multi-residue CCD ligands is rejected,
- ligand CCD integer-like YAML values are normalized to strings before use (`schema.py:200-230`), with a regression test in `tests/test_parse_ccd_type_coercion.py:4-33`.

### B. Run structure prediction

The structure stage filters out already-computed prediction folders unless `--override` is used (`main.py:319-362`, `1191-1196`).

For Boltz-2 structure inference:
- data loads through `Boltz2InferenceDataModule` (`main.py:1279-1291`, `inferencev2.py:317-433`),
- each item tokenizes, loads molecules, computes features, and can include pocket/contact conditioning (`inferencev2.py:206-303`),
- trainer inference writes ranked structures via `BoltzWriter` (`main.py:1255-1269`, `1337-1342`).

Ranking for output files is by descending `confidence_score` when present (`writer.py:79-85`).

### C. Affinity is an explicit second pass, not a side channel of structure prediction

If any manifest record has `record.affinity`, `main.py:1344-1420` launches a second prediction pass.

Mechanically:
- the structure writer stores only the rank-0 Boltz-2 affinity target structure as `pre_affinity_<id>.npz` (`writer.py:184-188`),
- the affinity datamodule then reloads **that saved structure** from `predictions/<id>/pre_affinity_<id>.npz` instead of the original processed target (`inferencev2.py:61-68`, `1368-1379`),
- affinity preprocessing crops with `AffinityCropper(max_tokens=512, max_atoms=4096)` (`inferencev2.py:238-247`),
- affinity prediction runs with a separate checkpoint, defaulting to `boltz2_aff.ckpt` (`main.py:1391-1412`).

### D. How the affinity pose is chosen

Inside Boltz-2 forward pass, affinity uses the structure samples already produced and then selects the best sample by a ligand-oriented ranking metric (`boltz2.py:609-636`):
- prefer `ligand_iptm`,
- fallback to `iptm`,
- fallback to `confidence_score`.

This rule is implemented in `affinity_ranking.py:5-19` and regression-tested in `tests/test_affinity_ranking.py:4-19`.

Important nuance: this pose-selection metric is used internally for affinity computation even if all those confidence fields are not later persisted to user JSON.

### E. What the affinity module computes

`AffinityModule` constructs pairwise affinity features from:
- trunk pair representation `z`,
- token embeddings `s_inputs`,
- pairwise distance bins from predicted representative-atom distances,
- a receptor mask defined as proteins plus non-binder ligands,
- a ligand mask defined by `affinity_token_mask` (`affinity.py:77-145`).

The actual pooled affinity head then outputs:
- `affinity_pred_value`
- `affinity_logits_binary`

from global averaging over ligand/receptor and ligand/ligand cross-pair regions (`affinity.py:183-235`).

### F. What the shipped affinity number means

The writer interprets the final affinity scalar as `log10(IC50_uM)` (`writer.py:329-345`). It then publishes:
- `affinity_pred_value`
- `affinity_pred_value_unit = "log10(IC50_uM)"`
- `affinity_ic50_um`
- `affinity_ic50_nm`
- `affinity_pkd_approx`
- `affinity_dg_kcal_mol_approx`
- `affinity_probability_binary`

The underlying conversions are implemented in `affinity_units.py:5-44` and smoke-tested in `tests/test_affinity_units.py:8-12`.

The code explicitly labels pKd and ΔG as approximations under the assumption `Kd ≈ IC50`; this is convenience post-processing, not an experimentally grounded mechanistic guarantee.

### G. Ensemble and molecular-weight correction behavior

When the loaded Boltz-2 checkpoint has `affinity_ensemble=True`, the model averages two affinity heads (`boltz2.py:639-729`).

If `affinity_mw_correction` is active in that ensemble branch, the final affinity value is adjusted using:
- `model_coef = 1.03525938`
- `mw_coef = -0.59992683`
- `bias = 2.83288489`
- `mw = feats["affinity_mw"][0] ** 0.3`

(`boltz2.py:708-725`)

That correction is a learned calibration-like post-adjustment inside code, not a first-principles thermodynamic model.

















## 5. What scientific reviewers should separate


### Native evidence
The native evidence is the installed `boltz predict` inference path and its standard output contract, including structure outputs, confidence/ranking outputs, and Boltz-2 affinity-head outputs when affinity is requested. This native prediction surface is the authoritative product layer.

### Transformed / calibrated evidence
Writer-level convenience conversions such as IC50 in alternate units, approximate pKd, and approximate ΔG are transformed presentation layers built on top of the native affinity writer output. They should be reviewed as downstream semantic transforms, not as separate learned thermodynamic heads.

### Detached analyst artifacts
`scripts/eval/` workflows, internal aggregation helpers, and analyst-side physical-simulation comparison scripts are detached artifacts rather than the native runtime contract. They are useful for benchmarking and internal analysis, but they are not the installed product surface.

### Non-authoritative surfaces
The broader repository contains multiple layers (CLI, UI, eval scripts, post-processing helpers), but they should not be collapsed into one authority class. Native prediction outputs, writer transforms, and detached evaluation scripts should not be spoken about as though they all carry the same scientific weight.


## 6. Current scientific and operational limits


### Scientific limits
Affinity prediction is bounded by the current Boltz-2 affinity-head contract, including one supported affinity ligand, single-ligand-chain assumptions, multi-residue rejection, and atom-count limits. The affinity path uses the top-ranked first-stage structure rather than an ensemble over all exported poses. Native output semantics and writer-level convenience conversions should not be mistaken for explicit molecular thermodynamics.

### Operational / runtime limits
Evaluation scripts depend on external tools and environment assumptions absent from the installed CLI. The broader platform mixes CLI, UI, evaluation, and post-processing layers, which increases operational surface area and the chance of conflating product and analyst workflows. Affinity dataloader token/atom caps remain hard coded in the current implementation path.

### Evidence / provenance limits
Test coverage is targeted and narrow rather than broad end-to-end runtime validation. Some raw diagnostics and MW-correction metadata appear implemented in intent but are not fully shipped through the current prediction/writer path. Reviewers should therefore rely on the actual emitted artifacts and current writer behavior rather than inferred intent from partial intermediate code.

### Signoff consequence
BoltzBase is strong as a native structure-plus-affinity inference platform for governed screening/triage, but it should not be signed off as a general thermodynamic authority or as if its detached evaluation scripts were part of the same validated product surface.


## 7. Recommended signoff questions


### Scientific validity question
Is it acceptable that the native affinity contract remains a model-output `log10(IC50_uM)` surface plus writer-level convenience conversions, rather than a separately established physical free-energy contract?

### Operational readiness question
Is the current platform scope operationally acceptable, including single- ligand affinity limits, ligand-size guards, top-ranked-pose-only affinity evaluation, and the mixed CLI/UI/post-processing repository surface?

### Evidence / provenance question
Should additional raw affinity diagnostics or MW-correction metadata be promoted into shipped outputs for auditability, or is the current emitted artifact set sufficient for review?

### Deployment-decision question
Should BoltzBase be approved as a governed native inference platform for screening/triage only, while detached evaluation scripts remain outside the approved product evidence surface?


## 8. Top current clarifications

















1. No material change identified: the native structure-first affinity-inference surface remains the same.
2. The authoritative shipped surface is still `boltz predict`; evaluation utilities are not first-class installed CLI modes.
3. Affinity is a two-stage, file-backed workflow through `pre_affinity_<id>.npz`, not a single monolithic prediction pass.
4. The public affinity JSON expresses a model value labeled as `log10(IC50_uM)` plus convenience conversions; approximate pKd and ΔG are assumptions-based derivations.
5. Current affinity scope is intentionally narrow: Boltz-2 only, one ligand binder, no multi-copy binder, no multi-residue affinity ligand, with explicit ligand-size constraints.
6. Raw affinity diagnostics and MW correction metadata appear intended in code comments but are not actually preserved in the shipped affinity JSON output.




## 9. Actual shipped operator surface

















### Installed CLI

The authoritative shipped command group is declared in `boltz-main/src/boltz/main.py:811-814`, with the active command at `main.py:817-1424`.

### User-facing command mode

- `boltz predict <data>` is the only installed CLI mode visible in the inspected code.
- `--model` accepts `boltz1` or `boltz2` (`main.py:981-986`).
- Boltz-2 affinity-related CLI switches are exposed directly on `predict`:
  - `--affinity_mw_correction/--no_affinity_mw_correction` (`main.py:999-1004`)
  - `--sampling_steps_affinity` (`main.py:1005-1010`)
  - `--diffusion_samples_affinity` (`main.py:1011-1016`)
  - `--affinity_checkpoint` (`main.py:1017-1022`)
  - `--use_potentials_affinity` (`main.py:973-980`)
- Generic inference controls include output directory, cache, checkpoint, devices, accelerator, recycling/sampling controls, output format, override, MSA server options, and embedding export (`main.py:817-1049`).

### Input file surface actually accepted

`check_inputs` only accepts directory members or single files ending in `.fa`, `.fas`, `.fasta`, `.yml`, `.yaml` (`main.py:281-316`).

Within YAML/JSON schema parsing:
- entity types accepted: protein, DNA, RNA, ligand (`schema.py:1051-1063`)
- affinity properties are only accepted when parsing as Boltz-2 (`schema.py:1078-1083`)
- affinity binder must be a single ligand chain (`schema.py:1085-1109`)

### Output contract actually written

Structure writer (`writer.py:23-274`) creates per-target directories under `predictions/<id>/` and may emit:
- ranked model coordinate files: `<id>_model_<rank>.cif` or `.pdb`
- `confidence_<id>_model_<rank>.json`
- `plddt_<id>_model_<rank>.npz`
- optionally `pae_<id>_model_<rank>.npz`
- optionally `pde_<id>_model_<rank>.npz`
- optionally `embeddings_<id>.npz`
- for Boltz-2 affinity targets only, top-ranked structure snapshot: `pre_affinity_<id>.npz`

Affinity writer (`writer.py:276-397`) writes:
- `predictions/<id>/affinity_<id>.json`

### What is not a shipped product surface

The repo-level evaluation scripts in `scripts/eval/` are not wired into the installed click CLI and should be treated as analyst-side tooling, not a supported runtime API.



## 10. Evidence appendix

















### Core code consulted

- `boltz-main/src/boltz/main.py:281-316` — input file acceptance.
- `boltz-main/src/boltz/main.py:365-411` — affinity skip/override filtering.
- `boltz-main/src/boltz/main.py:811-1424` — installed CLI and complete `predict` flow.
- `boltz-main/src/boltz/data/write/writer.py:23-274` — structure writer and `pre_affinity_<id>.npz` contract.
- `boltz-main/src/boltz/data/write/writer.py:276-397` — affinity JSON writer semantics.
- `boltz-main/src/boltz/model/models/boltz2.py:609-750` — internal affinity pose selection, ensemble handling, MW correction branch.
- `boltz-main/src/boltz/model/models/boltz2.py:1085-1149` — prediction outputs actually surfaced to writers.
- `boltz-main/src/boltz/model/modules/affinity.py:77-145` — affinity feature construction.
- `boltz-main/src/boltz/model/modules/affinity.py:183-235` — affinity heads and pooling.
- `boltz-main/src/boltz/model/utils/affinity_ranking.py:5-19` — ranking-key policy.
- `boltz-main/src/boltz/utils/affinity_units.py:5-44` — unit conversions and approximation semantics.
- `boltz-main/src/boltz/data/module/inferencev2.py:27-109` — affinity-stage loading of `pre_affinity_<id>.npz`.
- `boltz-main/src/boltz/data/module/inferencev2.py:238-244` — affinity cropping limits.
- `boltz-main/src/boltz/data/parse/schema.py:200-230` — ligand CCD coercion helper.
- `boltz-main/src/boltz/data/parse/schema.py:1078-1257` — affinity schema restrictions and ligand size checks.

### Tests consulted

- `boltz-main/tests/test_affinity_ranking.py:4-19`
- `boltz-main/tests/test_affinity_units.py:8-12`
- `boltz-main/tests/test_parse_ccd_type_coercion.py:4-33`

### Evaluation scripts consulted as non-product context

- `scripts/eval/run_evals.py:8-167`
- `scripts/eval/aggregate_evals.py:12-753`
- `scripts/eval/physcialsim_metrics.py:15-304`

### Test execution notes for this page

- `PYTHONPATH=boltz-main/src pytest -q boltz-main/tests/test_affinity_ranking.py boltz-main/tests/test_affinity_units.py boltz-main/tests/test_parse_ccd_type_coercion.py`
- Result: 4 tests passed; 1 test failed to import because the local environment lacks Biopython (`ModuleNotFoundError: No module named 'Bio'`) required by `schema.py`.
- Interpretation: the targeted ranking/unit tests pass in this environment; the schema regression test is present and relevant but could not fully execute here due missing dependency, not because the inspected coercion code is absent.
