# Sweep pipelining for compiled-system solves

Status: planned. This branch is where it gets attempted, as its own PR after
the Metal backend PRs (JWSound/BEAT_Engine#1, #2), to keep those reviewable.

## The gap

The Metal sweep pipeline (size-aware overlap `cee2af5`, deep lookahead
`d795501`) lives only in the source-request driver (`solver.jl` ->
`BeatEngineDriver.jl`). Boundary Lab's plain exterior radiator solves use that
driver and get it. System solves go through `coupled_solver.jl` and do not:

| Solve | Loop | Overlap today |
|---|---|---|
| Exterior-only system | `solve_exterior_request` | none across frequencies |
| Coupled FEM-BEM-LEM | `solve_request_impl` coupled loop | within one frequency only: BEM assembly (GPU) alongside FEM condensation (host), `BLAB_COUPLED_STAGE_OVERLAP` |

In both loops the GPU idles through each frequency's host solve, field
evaluation and emission.

## Plan

1. Exterior-only system solves. The loop has the driver's shape (Metal
   assembles, host solves), so `start_sweep_assembly_pipeline`, the
   memory-derived depth and `validate_metal_sweep_pipeline.jl` should carry over
   almost directly.
2. Coupled solves. BEM assembly and FEM condensation are combined in
   `build_condensed_coupled_system`, so the BEM operator assembly has to be split
   out as the producer step. Whether this pays depends on the per-frequency
   split: if host condensation and solve dominate, the within-frequency overlap
   already hides the GPU and the gain is small.

Decide on step 2 from measured stage timings (BEM operators, condensation,
solve) on SAWMOD before writing it. Either change only reorders work, so results
must stay bit-identical to the sequential path; the pipeline gate checks that.
