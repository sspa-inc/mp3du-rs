# mod-PATH3DU

**3D particle tracking engine for groundwater flow models — Rust core, Python API, agentic-first.**

mod-PATH3DU computes particle trajectories through groundwater flow grids and is compatible with all major versions of MODFLOW. It supports both structured grids (MODFLOW classic, MODFLOW 2005) and horizontally-unstructured, vertically-layered grids (MODFLOW-USG, MODFLOW 6 DIS/DISV).

> **Built with AI.** The API was designed and implemented in collaboration with AI. A custom MCP server exposed the original C++ source code directly to the implementing agent, grounding every algorithmic decision in the original mod-PATH3DU C++ codebase, which has years of real-world use behind it. The result is an API built from the ground up for agentic workflows — with machine-readable documentation, a JSON-schema-validated configuration contract, and structured outputs at every layer.

> **Grid limitation:** mod-PATH3DU requires *layer-conforming* discretization — cells must be organized into discrete horizontal layers with defined top and bottom elevations per cell. Fully 3D-unstructured connectivity (MODFLOW 6 DISU, where cells may connect arbitrarily in all three dimensions) is not supported.

---

## What it Does

Given a groundwater flow model, mod-PATH3DU will:

- **Track particles** forward or backward through 3D unstructured grids
- **Interpolate velocity** using the Waterloo method — a smooth, analytically differentiable stream-function fit
- **Integrate trajectories** with high-order adaptive Runge-Kutta solvers (Dormand-Prince, Cash-Karp, Verner variants)
- **Model stochastic dispersion** using GSDE or Itô formulations, or run purely advective simulations
- **Apply retardation** for chemical transport scenarios
- **Capture particles** at pumping wells through an analytically correct velocity field — no weak/strong sink designation required (which is a pragmatic approximation, not a physical one)
- **Run in parallel** — all particles are tracked concurrently

---

## Documentation

Full documentation, including a quickstart guide, Python API reference, schema reference, and worked examples:

**https://sspa-inc.github.io/mp3du-rs-docs/**

| Resource | Link |
|---|---|
| Quickstart | [Getting started](https://sspa-inc.github.io/mp3du-rs-docs/getting-started/quickstart/) |
| Python API Reference | [API docs](https://sspa-inc.github.io/mp3du-rs-docs/reference/python-api/) |
| Configuration Schema | [Schema reference](https://sspa-inc.github.io/mp3du-rs-docs/reference/schema-reference/) |
| Examples | [Worked examples](https://sspa-inc.github.io/mp3du-rs-docs/examples/) |
| For AI agents | [`llms.txt`](https://sspa-inc.github.io/mp3du-rs-docs/llms.txt) · [`llms-full.txt`](https://sspa-inc.github.io/mp3du-rs-docs/llms-full.txt) |

---

## Installation

mod-PATH3DU is proprietary freeware and is not published to PyPI. Install the compiled wheel directly from the [Releases](https://github.com/sspa-inc/mp3du-rs/releases) page.

**Requirements:** Python ≥ 3.8 · NumPy · Windows 64-bit

```bash
pip install https://github.com/sspa-inc/mp3du-rs/releases/download/v0.1.0/mp3du_py-0.1.0-cp38-abi3-win_amd64.whl
```

> **Note:** Replace the URL above with the link to the latest `.whl` asset on the [Releases](https://github.com/sspa-inc/mp3du-rs/releases) page.

Verify the installation:

```python
import mp3du
print(mp3du.version())
```

---

## Quick Look

```python
import json, numpy as np
import mp3du

# Build the grid
grid = mp3du.build_grid(vertices, centers)

# Hydrate cell data from NumPy arrays
props = mp3du.hydrate_cell_properties(top=..., bot=..., porosity=..., hhk=..., vhk=..., ...)
flows = mp3du.hydrate_cell_flows(head=..., water_table=..., face_flow=..., ...)

# Fit the Waterloo velocity field
waterloo_inputs = mp3du.hydrate_waterloo_inputs(...)
field = mp3du.fit_waterloo(mp3du.WaterlooConfig(), grid, waterloo_inputs, props, flows)

# Run particles
config = mp3du.SimulationConfig.from_json(json.dumps({
    "velocity_method": "Waterloo",
    "solver": "DormandPrince",
    "adaptive": {"tolerance": 1e-6, "safety": 0.9, "alpha": 0.2,
                 "min_scale": 0.2, "max_scale": 5.0, "max_rejects": 10,
                 "min_dt": 1e-10, "euler_dt": 1.0},
    "dispersion": {"method": "None"},
    "retardation_enabled": False,
    "capture": {"max_time": 365250.0, "max_steps": 1000000,
                "stagnation_velocity": 1e-12, "stagnation_limit": 100},
    "initial_dt": 1.0,
    "max_dt": 100.0,
    "direction": 1.0
}))

particles = [mp3du.ParticleStart(id=i, x=x, y=y, z=z, cell_id=cid, initial_dt=1.0)
             for i, (x, y, z, cid) in enumerate(starting_points)]

results = mp3du.run_simulation(config, field, particles, parallel=True)

for r in results:
    print(r.particle_id, r.final_status, len(r), "steps")
```

See the [Quickstart](https://sspa-inc.github.io/mp3du-rs-docs/getting-started/quickstart/) for a complete, runnable example.

---

## Capabilities at a Glance

| Feature | Details |
|---|---|
| Grid type | Structured (DIS) and horizontally-unstructured layered (DISV/USG); layer-conforming required — fully 3D-unstructured (DISU) not supported |
| Velocity interpolation | Waterloo stream-function method |
| ODE solvers | Euler, RK4 step-doubling, Dormand-Prince, Cash-Karp, Verner Robust, Verner Efficient |
| Dispersion | None (advection only), GSDE, Itô stochastic |
| Retardation | Per-cell retardation factor |
| Well capture | Physically correct by default — no weak/strong sink designation required; an optional capture radius is available for edge cases |
| Boundary capture | IFACE-based face routing |
| Parallelism | All particles tracked concurrently |
| Stagnation detection | Velocity threshold + step limit |

---

## License

See [license.txt](license.txt) for the end-user license agreement.
