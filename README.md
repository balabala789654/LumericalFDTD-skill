# LumericalFDTD

An OpenCode skill that automates Ansys Lumerical FDTD simulation workflows. Describe your photonic device in natural language, and the skill generates Python scripts, runs simulations, debugs errors autonomously, and delivers FSP files and result charts.

## Features

- **Natural Language to Simulation** — Describe structures, materials, sources, and monitors; receive working `.fsp` files
- **Autonomous Debug Loop** — Scripts are executed with Lumerical's built-in Python interpreter, errors are caught and fixed automatically
- **Separation of Simulation & Analysis** — Long-running simulations and fast data analysis are kept in independent scripts so you can tweak plots without re-running
- **Built-in Best Practices** — Dedicated references for building blocks, API usage, common errors, and diffraction analysis
- **Cross-Platform** — Supports Windows and Linux, auto-detects Lumerical installation paths and Python interpreter

## Supported Use Cases

Photonics device design, diffraction analysis, metasurfaces, waveguides, gratings, TGV (Through Glass Via) structures, optical field propagation — any task requiring FDTD simulation with Ansys Lumerical.

## Requirements

- [Ansys Lumerical FDTD](https://www.ansys.com/products/optics/fdtd) (tested with 2025 R2 / v252; older versions may work)
- The Lumerical installation must include the Python API (`lumapi`)

## File Management

Each simulation project follows a consistent directory structure:

| Directory | Contents |
|-----------|----------|
| `fsp/` | `.fsp` project files and simulation logs |
| `data/` | `.npz` result data and `.json` metadata |
| `pic/` | `.png` / `.jpg` charts and figures |
| root | `.py` scripts and `.md` documentation |

## Usage

In an OpenCode / Cowork session with this skill installed, describe your simulation need directly:

- "Design a 30 μm diameter circular aperture in a 50 μm thick SiO2 substrate, illuminated by a 100 μm plane wave, and observe the transmitted diffraction pattern"
- "Simulate the reflection spectrum of a gold grating — period 10 μm, duty cycle 0.5, wavelength 1–2 μm"
- "Parameter sweep: circular aperture diameter from 20 μm to 60 μm, step 10 μm, compare transmittance"

The skill follows a five-step pipeline: understand requirements → generate script → run & debug → verify results → save deliverables.

> [!NOTE]
> On first use, the skill automatically probes your system for the Lumerical installation path. If it cannot be found, you will be asked to provide it.

## Script Template

A reusable template is included at `scripts/template.py`. Each generated script follows this structure:

```python
# 1. Imports (API path, lumapi, numpy, matplotlib)
# 2. Parameter definitions (wavelength, geometry, monitors)
# 3. Simulation session (with lumapi.FDTD block):
#    - Simulation region
#    - Materials / substrate
#    - Geometry (bottom to top)
#    - Source
#    - Monitors
#    - Save → Run → Extract results
# 4. Post-processing (outside the session block)

with lumapi.FDTD(hide=True) as fdtd:
    fdtd.addfdtd()
    # ... build structure ...
    fdtd.save("fsp/simulation.fsp")
    fdtd.run()
```

Simulation and analysis are kept in separate scripts — `project_sim.py` for the heavy FDTD run and `project_analysis.py` for data processing and plotting. This lets you adjust colors, labels, and figure layout without re-running the simulation.

## Known Constraints

The Lumerical Python API has several quirks documented in `references/common-errors.md`. Key ones include:

| Constraint | Detail |
|------------|--------|
| Material names must be full strings | `"PEC (Perfect Electrical Conductor)"`, not `"PEC"` |
| Polygon method is `addpoly` | Not `addpolygon` |
| `addcone` does not exist | Stack `addcircle` layers instead |
| Monitors must stay within the simulation domain | Out-of-bounds monitors yield silent result failures |
| Raw strings cannot end with `\` | Use double backslashes: `"C:\\path\\"` |

## References

| File | When to read |
|------|-------------|
| `references/building-blocks.md` | Building geometry, setting up sources, monitors, and mesh |
| `references/api-reference.md` | Session management, SimObject, data passing, lumopt |
| `references/common-errors.md` | Troubleshooting runtime errors |
| `references/diffraction.md` | Aperture diffraction, Airy rings, near-field / far-field analysis |
