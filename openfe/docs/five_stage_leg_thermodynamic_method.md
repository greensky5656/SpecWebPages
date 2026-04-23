# Five-Stage Leg Thermodynamic Method

This document describes the literal five-stage thermodynamic method used for a
single real-compute mutation leg in the current codebase.

The implementation source for this method is:

- `scripts/pmx_real_compute_leg.py`

This is a **per-leg** method. It applies separately to an apo leg or a complex
leg.

## Scope

This document is specifically about how one mutation leg becomes a leg free
energy estimate.

It does not describe:

- how complex and apo legs are combined into `ddG`
- how WT anchor thermodynamics are harmonized
- how the final mutant affinity result is produced

Those higher-level steps belong to the pipeline-level thermodynamic chain.

## Stage 1: Build The Hybrid Mutation Topology

The code prepares a hybrid WT->MUT system using PMX mutation machinery.

What happens:

- prepare the WT protein structure for simulation
- apply the mutation with `pmx mutate`
- rebuild the hybrid topology with `pmx gentop`
- for complex legs, assemble the ligand-bearing system from the manifest-selected
  receptor and ligand

Thermodynamic meaning:

- this defines the alchemical endpoint system that will be sampled between
  lambda `0` and lambda `1`

## Stage 2: Equilibrate The Endpoint States

The code runs endpoint equilibration at both lambda endpoints.

What happens:

- prepare lambda `0`
- prepare lambda `1`
- run endpoint relaxation and equilibration before switching

Thermodynamic meaning:

- this produces physically prepared starting ensembles at the WT-like and
  MUT-like endpoints before non-equilibrium switching begins

## Stage 3: Run Forward And Reverse Non-Equilibrium Switching

The code launches fast-switching trajectories in both directions.

What happens:

- forward switches from `0 -> 1`
- reverse switches from `1 -> 0`
- multiple work samples are collected in both directions

Thermodynamic meaning:

- this generates the forward and reverse work distributions needed to estimate the
  leg free energy difference

## Stage 4: Integrate dH/dlambda Into Work Values

The switching trajectories produce `dH/dlambda` data that is integrated into
work.

What happens:

- read per-switch `dH/dlambda` output
- integrate across lambda steps
- collect forward and reverse work values for the leg

Thermodynamic meaning:

- this converts the switching trajectories into thermodynamic work observables

## Stage 5: Estimate The Leg Free Energy With BAR

The code applies BAR to the forward and reverse work distributions.

What happens:

- use the collected forward and reverse work values
- solve the BAR equation
- return the leg `delta_g_kcal_per_mol`

Thermodynamic meaning:

- this is the final free-energy estimate for one mutation leg, either the apo leg
  or the complex leg

## Important Mode Note

This five-stage method is the real-compute thermodynamic path. In
`research_mode`, the code does not execute these physical stages; it substitutes
deterministic synthetic leg values instead.

## Where This Lives In Code

- `scripts/pmx_real_compute_leg.py`

## Practical Summary

If someone asks, "what are the literal five thermodynamic stages?", the answer in
the current implementation is:

1. build the hybrid mutation topology
2. equilibrate the endpoint states
3. run forward and reverse non-equilibrium switching
4. integrate `dH/dlambda` into work values
5. estimate the leg free energy with BAR

## Related Documents

- `docs/pipeline_thermodynamic_chain.md`
- `docs/design_specification.md`
- `docs/pipeline_flowchart.md`
