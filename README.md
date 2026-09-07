# ONI Custom Simulation API
## Pre-Release SDK Preview

> **A source-audited preview of the custom simulation API for Oxygen Not Included.**
>
> Physics extensions • Thermodynamics • Fluid dynamics • Matter • Atmospheres • Pipe systems • Automation • SimViz

This repository is the **pre-release SDK reference and showcase** for the custom ONI simulation stack.

It intentionally contains **documentation and demonstrations, not implementation source code**. The implementation repositories are currently private and are targeted for release on **Sunday, September 13, 2026**, subject to final release preparation and validation.

---

## 🎬 See the simulation in action

The four demonstrations below are the best way to understand what this API is actually intended to make possible.

### AirLoop — Composition-aware atmospheres

<a href="https://youtu.be/3JrISB8L77M"><img src="https://img.youtube.com/vi/3JrISB8L77M/maxresdefault.jpg" alt="Watch the AirLoop demonstration" width="800"></a>

**Multi-species atmosphere · oxygen partial pressure · composition-aware breathing · contamination**

AirLoop seals an atmosphere, mixes species to a target oxygen partial pressure, and checks what a duplicant actually breathes. It demonstrates why a real atmosphere needs composition rather than a single-element cell model.

**[▶ Watch AirLoop on YouTube](https://youtu.be/3JrISB8L77M)** · **[Read the Gas Mixture API](docs/api/GAS-MIXTURE.md)** · **[Read the Atmosphere API](docs/api/ATMOSPHERE.md)**

---

### PhaseLoop — Thermodynamic phase change

<a href="https://youtu.be/MsLZPUQXO1E"><img src="https://img.youtube.com/vi/MsLZPUQXO1E/maxresdefault.jpg" alt="Watch the PhaseLoop demonstration" width="800"></a>

**Pressure-driven phase change · latent heat · gas/liquid bridging · work · energy transfer**

PhaseLoop demonstrates a real gas/liquid loop in which pressure-driven phase change and work move energy between the cold and hot sides rather than simply changing a temperature value.

**[▶ Watch PhaseLoop on YouTube](https://youtu.be/MsLZPUQXO1E)** · **[Read the Pipe Matter API](docs/api/MATTER.md)** · **[Read the Pipe Network API](docs/api/PIPES.md)** · **[Read the Thermal & Energy API](docs/api/THERMAL-ENERGY.md)**

---

### PipeStress — Matter-aware pipe failure

<a href="https://youtu.be/5R5uP2IiXP4"><img src="https://img.youtube.com/vi/5R5uP2IiXP4/maxresdefault.jpg" alt="Watch the PipeStress demonstration" width="800"></a>

**Pressure · thermal stress · condensation · freezing · standing matter · rupture**

PipeStress drives a real 12-tile gas run through baseline, mass overpressure, thermal overpressure, condensation, freezing, and rupture/stress states. A major result of the test was that pipe stress must account for **standing liquid and standing solid matter**, not merely flowing conduit contents.

**[▶ Watch PipeStress on YouTube](https://youtu.be/5R5uP2IiXP4)** · **[Read the Pipe Network API](docs/api/PIPES.md)**

---

### SimViz — Inspect the simulation itself

<a href="https://youtu.be/1bOM0Sx26bE"><img src="https://img.youtube.com/vi/1bOM0Sx26bE/maxresdefault.jpg" alt="Watch the SimViz demonstration" width="800"></a>

**Offline simulation · live attachment · history playback · cell inspection · extension data · diagnostics**

SimViz is a separate native application for inspecting the same simulation data through captured/offline state or a running ONI process. It supports simulation history, individual cell inspection, extension properties, element attributes, events, state dumps, statistics, snapshots, and headless rendering/testing.

**[▶ Watch SimViz on YouTube](https://youtu.be/1bOM0Sx26bE)** · **[Explore the SDK documentation](#sdk-documentation)**

---

> ### About the video presentation
> GitHub does **not** permit arbitrary YouTube `<iframe>` players inside repository Markdown. The large preview images above are therefore implemented as **clickable video thumbnails**: they look and behave like embedded showcase panels while remaining compatible with GitHub's README renderer.
>
> The original `.mp4` recordings are also retained on the [`Video-Uploads` branch](https://github.com/Salacious-Oni-Dev/API-Pre-Release-Preview/tree/Video-Uploads) as archival copies.

---

# SDK documentation

The preview is organized as a **reference manual**, not a giant class-name dump. Each API page is being built from the actual source declarations and implementations, with the native path traced where relevant.

## Core APIs

| Area | Reference | What it exposes |
|---|---|---|
| 🧪 Gas | [Gas Mixture](docs/api/GAS-MIXTURE.md) | Multi-species composition, mass, pressure and conversion |
| 🌬️ Atmosphere | [Atmosphere](docs/api/ATMOSPHERE.md) | Oxygen, contaminants, temperature and combustion assessment |
| 🧱 Materials | [Material Properties](docs/api/MATERIALS.md) | Material-property extension and lookup surfaces |
| 💧 Matter | [Pipe Matter](docs/api/MATTER.md) | Condensed/boiled standing matter inside conduits |
| 🧯 Pipes | [Pipe Networks](docs/api/PIPES.md) | Pipe networks, pressure, stress and standing-matter state |
| 🔥 Energy | [Thermal & Energy](docs/api/THERMAL-ENERGY.md) | Thermal mass, work, heat and enthalpy extensions |
| 🧬 Elements | [Elements & Cells](docs/api/ELEMENTS.md) | Element/cell-facing simulation surfaces |
| 📚 All APIs | [API Reference Index](docs/api/README.md) | Complete verified API documentation as the audit progresses |

Additional reference pages cover extension properties, element attributes, events, scheduling, persistence, rooms, deterministic state, diagnostics, and tooling as their source audits are completed.

---

# Architecture

```text
                         OXYGEN NOT INCLUDED
                                  │
                    ┌─────────────▼─────────────┐
                    │       Mod / Gameplay      │
                    └─────────────┬─────────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │       oni-framework       │
                    │       managed SDK         │
                    └─────────────┬─────────────┘
                                  │
                    ┌─────────────▼─────────────┐
                    │      ONI-Sim-Custom       │
                    │  native simulation + ABI  │
                    └─────────────┬─────────────┘
                                  │
                         extended simulation
                                  │
                    ┌─────────────▼─────────────┐
                    │          SimViz            │
                    │ offline + live inspection │
                    └───────────────────────────┘
```

### The integration rule

> **New simulation capabilities belong in the custom SimDLL and are exposed to mods through `oni-framework`. Flagship and third-party mods should not privately reach into the native DLL.**

- `ONI-Sim` — frozen 1:1 vanilla replacement.
- `ONI-Sim-Custom` — native extension implementation.
- `oni-framework` — managed modder-facing contract.
- `oni-flagship` — flagship systems consuming the framework.
- `oni-simviz` — independent simulation visualization and inspection.

---

# What makes this an SDK preview?

The goal is not merely to show that the custom simulation works. The goal is to document a **usable contract for mod authors**.

For significant public surfaces, the reference documents aim to answer:

1. **What can I read?**
2. **What can I change?**
3. **What does the value mean?**
4. **What are the units?**
5. **When is it safe or valid to call?**
6. **Does it allocate?**
7. **Is it simulation-thread constrained?**
8. **Does it require the custom SimDLL?**
9. **What happens with the stock SimDLL?**
10. **Which real flagship systems consume it?**
11. **What are the limitations or deliberate stubs?**
12. **What does a practical mod usage pattern look like?**

The documentation is explicitly being produced by tracing **actual source declarations and implementations**. A filename is never treated as proof that a type or method is public, supported, or modder-facing.

### API classifications

- **Modder-facing** — intended gameplay/mod integration surface.
- **Developer / diagnostic** — useful for tools, validation and development.
- **Internal plumbing** — implementation machinery that normal mods should not call directly.
- **Experimental / stubbed** — exposed for development or future work but not yet a complete gameplay contract.

---

# Integration requirements

Consuming mods should reference `OniFramework.dll` as a shared framework dependency rather than bundling their own copy.

`FrameworkVersion.Require(major, minor, consumer)` declares the minimum framework API floor, while duplicate-instance detection protects against accidentally loading multiple framework copies.

Normal gameplay mods should use the public framework facades appropriate to their needs. `OniExtMessages` and direct native P/Invoke are implementation plumbing, not the intended modder contract.

---

# Release status

**Current:** private pre-release documentation and showcase.

**Target source release:** **Sunday, September 13, 2026**, subject to final release preparation and validation.

The implementation repositories are currently private. This preview intentionally provides the SDK documentation, architecture, examples, demonstrations, compatibility information, ABI documentation, limitations, and source-audit notes without attaching the implementation source itself.

See **[Release Status](docs/RELEASE-STATUS.md)** for the detailed release model.

---

## Repository structure

```text
API-Pre-Release-Preview/
├── README.md                         ← you are here
├── docs/
│   ├── GETTING-STARTED.md
│   ├── ARCHITECTURE.md
│   ├── RELEASE-STATUS.md
│   ├── api/                          ← SDK reference
│   ├── native/                       ← ABI / native boundary
│   ├── examples/                     ← practical usage
│   └── showcases/                    ← rig-by-rig technical docs
└── Video-Uploads branch              ← archival MP4 recordings
```

---

<p align="center">
  <strong>Custom simulation, exposed as a modder-facing SDK.</strong><br>
  <sub>Source-audited • Native-backed • Demonstrated • Pre-release</sub>
</p>
