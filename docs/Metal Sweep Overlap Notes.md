# Metal sweep overlap: deciding from measured timings

Parked: notes for a later change, no code yet.

## Where it stands

`BeatEngineSweepOverlap.jl` overlaps the next frequency's GPU assembly with
this frequency's host solve when the modelled saving per frequency,
`min(S - c, A - kappa S)`, is positive. Its constants were calibrated on one
M1 Pro. `scripts/calibrate_metal_sweep_overlap.jl` refits them, but GUI users
never run it, so every other machine decides with M1 Pro numbers. Near the
break-even point a wrong choice costs a few ms per frequency (`sample_half`,
`sample_quarter`). On a machine far from an M1 Pro, such as an Ultra's GPU or a
base M1's cores, it could cost more.

## Proposal: measure during the solve, keep the result until the program closes

1. Take A and S from the sweep itself: every frequency already reports
   `assembly_s` and `solve_s`. Run the first two frequencies sequentially,
   discard the first (it carries one-off warm-up, about 0.3 s on an M1 Pro) and
   decide from the second, with c and kappa still from the defaults (they only
   show up when overlapping). Switching mid-sweep keeps results bit-identical,
   because a Metal exterior sweep uses the same BLAS thread count either way.
2. Optionally check the choice: run the third frequency overlapped, compare it
   with the second, keep the faster.
3. Cache the measured A and S in the worker's memory, keyed by chip, Julia and
   BLAS threads, dof count, symmetry copies and solve method. Boundary Lab keeps
   one Julia worker per configuration alive until it closes
   (`get_beat_engine_worker`), so the cache lasts as long as the program and
   cannot go stale across hardware or software updates. A new mesh can scale
   the cached A by dofs squared rather than measure again.

## Risk: a busy machine while measuring

If another program is using the CPU or GPU during the deciding frequencies, A
or S come out wrong and the cached choice is wrong for the session.

- Keep measuring: every sweep reports A and S anyway, so decide from a running
  median rather than the first measurement.
- Only cache a measurement within a factor of two of the model.
- A wrong choice costs the gap between the two options, which is small near
  break-even, where a wrong choice is most likely.

## Core count

BLAS gets one thread fewer than its default so the GPU producer has a core.
Measured on the M1 Pro, overlapped, first sweep / repeat:

| BLAS threads | `sample.msh` (1,390 dofs) | `sample_detailed.msh` (3,502 dofs) |
|---|---|---|
| 8 | 1.56 / 1.38 s | 5.69 / 5.82 s |
| 7 (shipped) | 1.47 / 1.21 s | 5.73 / 5.50 s |
| 4 | 1.41 / 1.10 s | 6.30 / 6.39 s |

The best count depends on the mesh size, because small solves do not use many
threads well. The calibration script could search the count and print a
per-machine setting, possibly per mesh-size band. It has to stay fixed for a
given machine and mesh: LU results depend on the BLAS thread count, so a count
chosen from live load would make the answer depend on what else was running.

## Ruled out

- Changing thread counts during a solve from live load: breaks reproducibility,
  as above.
- Balancing from GPU utilisation and memory bandwidth: macOS only exposes them
  through `powermetrics`, which needs root.
