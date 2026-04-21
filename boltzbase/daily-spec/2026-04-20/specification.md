# boltzbase Specification

Updated: 2026-04-20 21:13:35 PDT

Codebase-grounded scientific review of the current boltzbase / boltz-main shipped surfaces: predict CLI, structure outputs, Boltz-2 affinity pass, writer contract, and the distinction between native output capability and non-productized evaluation scripts.

Repository: `/home/joethomas/Documents/Github/b1-b4/boltzbase`

Authority policy: this page is grounded primarily in current source code, tests, and shipped output-contract logic. README/docs were treated as secondary context only.

## Executive view for signoff

<p>The real productized operator surface in boltzbase is still the <code>boltz predict</code> command. Structure prediction and optional affinity prediction are embedded inside that path. Evaluation, benchmark, and calibration narratives must be separated from this native CLI surface unless they are exposed by current code or output contracts.</p>

## Actual shipped operator surface

- `boltz predict`: primary shipped CLI surface
- writer/output contract: real current output semantics
- `scripts/eval/*`: standalone scripts, not first-class installed CLI commands

## Scientific workflow actually implemented

1. Run structure inference through `boltz predict`
2. Rank generated samples by confidence-related criteria
3. Save a pre-affinity top-ranked structure snapshot for affinity-enabled Boltz-2 records
4. Run a second-stage affinity pass on that selected pose
5. Write affinity JSON with native prediction values plus convenience conversions

## What scientific reviewers should separate

- Native predict CLI: real shipped structure + optional affinity surface
- Affinity JSONs: writer-defined current output contract
- Utility conversions: convenience approximations, not new physical simulations
- Evaluation scripts: research/standalone surfaces, not productized CLI mode

## Current scientific and operational limits

- Affinity is model-based prediction, not physics-based free-energy simulation
- Affinity eligibility is constrained in code
- Writer/output quirks exist
- Evaluation is not productized as installed CLI

## Recommended signoff questions

### For genetics leadership
- Is native Boltz affinity being used as screening/triage support or overinterpreted as final quantitative authority?
- Do you want convenience conversions and benchmark narratives clearly separated from native outputs?

### For thermodynamics review
- Should model-based affinity outputs be explicitly labeled as distinct from MM/GBSA or RBFE evidence?
- Should evaluation-script claims be quarantined from the productized CLI surface unless first-class operator support appears?

## Evidence appendix

- `/home/joethomas/Documents/Github/b1-b4/boltzbase/boltz-main/src/boltz/main.py`
- `/home/joethomas/Documents/Github/b1-b4/boltzbase/boltz-main/src/boltz/data/write/writer.py`
- `/home/joethomas/Documents/Github/b1-b4/boltzbase/boltz-main/src/boltz/model/models/boltz2.py`
- `/home/joethomas/Documents/Github/b1-b4/boltzbase/boltz-main/src/boltz/model/modules/affinity.py`
- `/home/joethomas/Documents/Github/b1-b4/boltzbase/boltz-main/src/boltz/model/utils/affinity_ranking.py`
- `/home/joethomas/Documents/Github/b1-b4/boltzbase/boltz-main/src/boltz/utils/affinity_units.py`
- `/home/joethomas/Documents/Github/b1-b4/boltzbase/boltz-main/src/boltz/data/module/inferencev2.py`
- `/home/joethomas/Documents/Github/b1-b4/boltzbase/boltz-main/src/boltz/data/parse/schema.py`
- `/home/joethomas/Documents/Github/b1-b4/boltzbase/scripts/eval/run_evals.py`
- `/home/joethomas/Documents/Github/b1-b4/boltzbase/scripts/eval/aggregate_evals.py`

## Top current clarifications

- The authoritative shipped operator surface is still `boltz predict`; evaluation is not exposed as a first-class installed CLI mode.
- Current affinity JSON semantics are defined by writer.py and represent model output plus convenience conversions, not a physics simulation contract.
- Repo-level benchmark/evaluation claims must be separated from the actual productized CLI and current writer/output surfaces.