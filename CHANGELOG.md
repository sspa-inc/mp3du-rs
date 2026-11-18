# Changelog

## 0.1.3 (2026-11-18)

Changes since 0.1.2. Results are unchanged for valid input.

### Fixed
- **A cell that cannot be fitted is reported, not a crash.** `fit_waterloo()` (in Rust, `WaterlooField::new`) now raises a `ValueError` naming the cell when a cell's fit input has fewer than 3 faces, face arrays of different lengths, or a no-flow face index out of range. Before, such a cell panicked when a particle first entered it, and because the release build aborts on a panic, the Python process ended without an error message. Valid input gives the same results as before.

### Documentation
- **`q_top` and `q_bot` are positive upward,** as in mod-PATH3DU: `q_bot` positive is flow into the cell from below, `q_top` positive is flow out through the top, so recharge entering the top is negative. The documentation said `q_top` positive was flow entering through the top; following that made particles in recharge cells rise instead of sink. The code did not change.
- **`q_well` is MODFLOW's well rate, unchanged** (negative = pumping). The documentation's IFACE tables and its helper `route_iface_bc_flows()` negated IFACE 0 flows; with the negated rate a pumping well's cell repels particles near the well. The helper is corrected in the documentation repository. Direct `q_well` arrays were always to be passed unchanged.
- **The Quickstart and the minimal script run.** Both passed cells without faces (the Quickstart crashed Python, the minimal script raised a `ValueError`), listed vertices counter-clockwise, and gave `z` as an elevation instead of a height within the cell. Both now set up uniform flow on a real grid and print positions that can be checked by hand.

## 0.1.2 (2026-11-18)

Changes since 0.1.1.

### Fixed
- **The Waterloo method uses aquifer storage.** `fit_waterloo()` subtracts `cell_flows.q_storage` from the vertical term `q_vert`, in both the fit of each cell and the velocity evaluation, as mod-PATH3DU 2.1.9 does (`qvert = (qtop - qbot) + (qtopface - qbotface) - qsto`). Storage acts as a source or sink spread evenly over the cell. In 0.1.0 and 0.1.1, `hydrate_cell_flows()` accepted `q_storage` and kept it, but nothing used it: passing storage gave the same result as passing zeros. In transient models the horizontal velocity was therefore wrong wherever storage is a sizable part of a cell's flow. In a transient MODFLOW 6 test model with 15-day steps, using storage moved particle end positions by 0.68 m on average and by up to 39 m. In a closed-form case with uniform storage, the path error after 300 days fell from 15.8 m to 0.44 m.

### Changed
- **Pass storage in `q_storage`, as MODFLOW gives it.** `q_storage` is the storage flow in MODFLOW's sign and in volume per time: positive is water released from storage into the cell. For MODFLOW 6 pass `STO-SS + STO-SY` per cell (through the API, `GWF/STO/STRGSS + GWF/STO/STRGSY`); for MODFLOW-2005 and MODFLOW-USG pass `STORAGE`; for steady state pass zeros. Do not negate it.
- **Check scripts written for 0.1.1.** A non-zero `q_storage` passed to 0.1.1 now takes effect. If you added storage into `q_vert` yourself (for example `q_vert = -storage`, or `q_vert` set to the cell residual `sum(face_flow) + q_well + q_other`, which already contains storage), either keep that and pass `q_storage` as zeros, or take storage out of `q_vert` and pass it in `q_storage`. Doing both counts storage twice.
- Steady-state models, and scripts that pass `q_storage` as zeros, give results identical to 0.1.1, bit for bit.
- **Documentation: boundary flows through the top and bottom faces belong in `q_vert`.** The documentation said that flows routed to the top or bottom face (IFACE 5, 6 and 7, such as recharge) must be left out of `q_vert`. mod-PATH3DU includes them, and the Waterloo fit needs them: leaving recharge out of `q_vert` leaves it out of the horizontal velocity, as leaving storage out did. For a cell whose budget closes, `q_vert = sum(face_flow) + q_well + q_other + q_storage` (`face_flow` positive into the cell, the rest in raw MODFLOW sign). The code did not change on this point; scripts that followed the old advice should be checked.

### Added
- Rust: `mp3du_grid::flow::qvert_with_storage(q_vert, q_storage)` gives the vertical term that `CellFitInput::q_vert` and `CellEvalContext::q_vert` expect, for Rust users who build these directly.
- The docstrings of `hydrate_cell_flows()`, `hydrate_waterloo_inputs()` and `fit_waterloo()`, and the type stub, describe `q_vert` and `q_storage`. The stub lists the keys of `TrajectoryResult.to_records()`: `step`, `time`, `x`, `y`, `z`, `vx`, `vy`, `vz`, `cell_id` and `dt`.
- Tests: storage passed in `q_storage` gives the same trajectories as the same storage folded into `q_vert`; and a closed-form case with uniform storage.

## 0.1.1 (2026-11-17)

Changes since 0.1.0.

### Fixed
- **Adaptive solvers no longer stay on the Euler fallback.** In 0.1.0, once the step size dropped to `adaptive.euler_dt` or below, every later step was a first-order Euler step of `euler_dt`, and `tolerance` had no further effect. With `euler_dt` at 1.0 or 0.1, as the 0.1.0 README and examples had it, `DormandPrince`, `CashKarp`, the Verner solvers and `Rk4StepDoubling` all ran as fixed-step Euler. After the one Euler step, the next step now grows by `safety / 0.5^alpha`, as in the C++, and adaptive stepping resumes.
- **A pathline ends on `capture.max_time`.** Every step is limited to the tracking time that is left, as in the C++. In 0.1.0 the last step could overshoot `max_time` by up to `max_dt`.
- **A particle that reaches the edge of the domain ends as `ExitedDomain`.** In 0.1.0 it could end as `SolverFailed` (with `min_dt` at 1e-10 and a smaller `euler_dt`), and `Rk4StepDoubling` could stall there until `max_steps`.
- **`Rk4StepDoubling` reports a dry or outside start cell at once,** like the other solvers. In 0.1.0 it retried up to `max_rejects` times first.
- **`adaptive.min_dt` above `max_dt` is rejected** with a `ValueError` when the configuration is built. In 0.1.0 it was accepted: every particle then ended as `SolverFailed` at its first step, or, with the `Euler` solver, tracking panicked.

### Changed
- **`tolerance` is relative to the model extent.** The step error in each direction is divided by the model extent in that direction (`xmax - xmin`, `ymax - ymin`, `zmax - zmin`), as in the C++. In 0.1.0 it was divided by the particle's own coordinates, so the accuracy depended on where the grid's origin was. Step counts and results change slightly. Models in large coordinates, such as UTM, now take smaller and more accurate steps.
- **The `dt` of a trajectory record is the step taken to reach it** (0 at the release point), as the documentation says. In 0.1.0 it held the suggested next step.
- **Steps rejected because a stage point left the domain no longer count toward `max_rejects`.**

### Added
- `SimulationConfig.from_json()` and `.validate()` issue a Python `UserWarning` when `adaptive.euler_dt` is used like a step size: above `adaptive.min_dt` and not small next to `initial_dt`.

### Guidance
- Set `adaptive.euler_dt` equal to `adaptive.min_dt`, for example both `1e-10`. `euler_dt` is the threshold of the adaptive solvers' emergency Euler step, which makes it a floor on the step size. The `Euler` solver does not use it; it steps at `initial_dt`. This applies to 0.1.0 as well.
