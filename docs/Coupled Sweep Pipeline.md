# Sweep pipelining for compiled-system solves

Status: exterior-only planned; coupled not worth implementing for FEM-heavy
projects (measured below). This branch is where it gets attempted, as its own PR
after the Metal backend PRs (JWSound/BEAT_Engine#1, #2), to keep those
reviewable.

## The gap

The Metal sweep pipeline (size-aware overlap `cee2af5`, deep lookahead
`d795501`) lives only in the source-request driver (`solver.jl` ->
`BeatEngineDriver.jl`). Boundary Lab's plain exterior radiator solves use that
driver and get it. System solves go through `coupled_solver.jl` and do not:

| Solve | Loop | Overlap today |
|---|---|---|
| Exterior-only system | `solve_exterior_request` | none across frequencies |
| Coupled FEM-BEM-LEM | `solve_request_impl` coupled loop | within one frequency only: BEM assembly (GPU) alongside FEM condensation (host), `BLAB_COUPLED_STAGE_OVERLAP` |

## When can pipelining win?

Per frequency, let

- `S` = host work the within-frequency overlap already runs alongside the GPU
  (FEM condensation: factorization + Schur extraction),
- `G` = GPU work (BEM operator assembly),
- `L` = host work that needs the GPU result (coupled factorization, BEM matrix,
  solve, field).

Today a frequency costs `max(S, G) + L`. An ideal cross-frequency pipeline costs
`max(S + L, G)`. The largest possible gain is therefore

    0                 if G <= S
    min(G - S, L)     if G >  S

so pipelining a coupled sweep only pays when the GPU assembly outlasts the host
condensation. A faster GPU shrinks `G` and makes that less likely, not more.

## Measured: SAWMOD (Boundary Lab example), M1 Pro, PR #1 Metal

12 frequencies 20 Hz - 20 kHz, `xy` symmetry, 3 drivers, 3,110 BEM dofs, FEM
volumes of 11k / 10.5k / 94k tets, 4 Julia threads. Per frequency (steady):

| Stage | s |
|---|---|
| FEM Schur extraction (host UMFPACK, 4 threads) | 5.3 |
| FEM condensation factorization | 0.5 |
| BEM operators (GPU) | 1.5 |
| Coupled factorization (host dense LU) | 2.4 |
| BEM matrix + block assembly | 0.4 |
| Solve + field | 0.4 |

`S` = 5.8, `G` = 1.5, `L` = 3.2: the gain is zero, and stays zero unless the GPU
were ~4x slower relative to the CPU than on this machine, which no current
Apple Silicon configuration is. Larger GPUs move further from the crossover. For
FEM-heavy coupled projects the time is in host condensation and the coupled LU;
that is where speed has to come from.

The exception is project-dependent, not hardware-dependent: a coupled model
with a small FEM region and a large exterior can have `G > S`. If a FEM-light
benchmark shows that, decide per sweep rather than per machine: after the first
frequency compare the reported `bem_operator_s` with `fem_condensation_s` and
pipeline only when `G > S`. No calibration constants needed.

## Plan

1. Exterior-only system solves. The loop has the driver's shape (Metal
   assembles, host solves), so the gain is real on every machine above some
   size. Reuse `start_sweep_assembly_pipeline`, the memory-derived depth and
   `validate_metal_sweep_pipeline.jl`. Take the on/off decision from a
   calibration (GPU assembly vs host solve throughput, env-overridable, like
   the dense-solve cost model and `scripts/calibrate_dense_solve.jl`) rather
   than the fixed `METAL_PIPELINE_MIN_DOFS = 1900` measured on one machine.
2. Coupled solves. Not implemented unless a representative FEM-light project
   shows `G > S`. Then: the per-sweep decision above.

Either change only reorders work, so results must stay bit-identical to the
sequential path; the pipeline gate checks that.
