# Changelog

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
