# Pipeline Thermodynamic Chain

This document describes the thermodynamic chain used by the full mutation RBFE
pipeline after leg values exist.

This is a **pipeline-level** thermodynamic view. It explains how complex and apo
leg free energies, WT anchor thermodynamics, and calibration are combined into
the final mutant affinity result.

## Scope

This document covers the higher-level thermodynamic chain implemented across:

- `src/fep/convergence.py`
- `src/inference/ddg_estimator.py`
- `src/calibration/anchor_harmonizer.py`
- `src/calibration/affinity_converter.py`
- `src/pipeline.py`

It does not describe the literal five-stage non-equilibrium physical method used
to compute a single real mutation leg. That lower-level method is documented
separately.

## Stage 1: Estimate The Complex-Leg Free Energy

The complex leg is summarized from its replicate values.

What happens:

- collect complex-leg replicate outputs
- compute mean, standard deviation, and confidence width
- produce the summarized complex-leg free energy estimate

Thermodynamic meaning:

- this is the free-energy effect of the mutation in the bound state

## Stage 2: Estimate The Apo-Leg Free Energy

The apo leg is summarized in the same way.

What happens:

- collect apo-leg replicate outputs
- compute mean, standard deviation, and confidence width
- produce the summarized apo-leg free energy estimate

Thermodynamic meaning:

- this is the free-energy effect of the mutation in the unbound protein state

## Stage 3: Compute The Mutation Binding Delta-Delta-G

The pipeline combines the two legs into `ddG`.

What happens:

- compute `ddG = complex_leg - apo_leg`
- propagate uncertainty from the two legs

Thermodynamic meaning:

- this is the mutation-induced binding free-energy shift

## Stage 4: Harmonize The WT Anchor Thermodynamics

The pipeline builds a WT absolute binding free-energy anchor from assay records.

What happens:

- load WT anchor records
- convert `KD`, `Ki`, or `IC50`-derived values into a common thermodynamic form
- prioritize assays by the implemented hierarchy
- combine the best-priority records into a WT anchor `delta_g`

Thermodynamic meaning:

- this supplies the WT absolute thermodynamic reference state

## Stage 5: Convert WT Anchor Plus ddG Into Mutant Affinity

The final mutant affinity is computed from the WT anchor and the mutation `ddG`.

What happens:

- compute the raw mutant free energy from WT anchor plus `ddG`
- apply the configured calibration model
- propagate final uncertainty
- emit the mutant affinity outputs in the final report

Thermodynamic meaning:

- this produces the final mutant absolute affinity estimate reported by the
  pipeline

## Core Pipeline Equation

At the full-pipeline level, the implemented thermodynamic chain is:

```text
complex leg
minus
apo leg
= mutation ddG

WT anchor delta G
plus
mutation ddG
= mutant delta G

calibration(mutant delta G)
= final reported mutant affinity result
```

## Where This Lives In Code

- `src/fep/convergence.py`
- `src/inference/ddg_estimator.py`
- `src/calibration/anchor_harmonizer.py`
- `src/calibration/affinity_converter.py`
- `src/pipeline.py`

## Practical Summary

If someone asks, "what is the thermodynamic chain for the whole pipeline?", the
current answer is:

1. estimate the complex-leg free energy
2. estimate the apo-leg free energy
3. compute mutation `ddG`
4. harmonize the WT anchor thermodynamics
5. convert WT anchor plus `ddG` into the final mutant affinity result

## Related Documents

- `docs/five_stage_leg_thermodynamic_method.md`
- `docs/runtime_artifacts_reference.md`
- `docs/design_specification.md`
- `docs/pipeline_flowchart.md`
