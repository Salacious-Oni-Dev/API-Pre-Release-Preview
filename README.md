# ONI Custom Simulation API — Pre-Release SDK Preview

> **Private technical preview and pre-release SDK reference** for the custom Oxygen Not Included simulation stack, `oni-framework`, flagship simulation systems, and SimViz.

This repository intentionally contains **documentation and demonstrations, not implementation source code**. It is the developer-facing preview of the SDK before the implementation repositories are released.

## Release status

- **Current:** documentation + showcase preview; no implementation source is attached.
- **Expected source release:** the currently private related repositories are **targeted for Sunday, September 13, 2026**, subject to final release preparation and validation.
- **Pre-release caveat:** API details may still change before source publication. The goal of this preview is to make the modder-facing contract as explicit and stable as possible.

See [Release Status](docs/RELEASE-STATUS.md).

---

# SDK Manual

The README is the portal. The detailed API reference is split into smaller documents so individual APIs can be documented with their real signatures, behavior, units, performance characteristics, native dependencies, limitations, and examples without creating one enormous file.

## Core API reference

- [API Reference Index](docs/api/README.md)
- [Gas Mixture](docs/api/GAS-MIXTURE.md)
- [Atmosphere](docs/api/ATMOSPHERE.md)
- [Material Properties](docs/api/MATERIALS.md)
- [Pipe Matter](docs/api/MATTER.md)
- [Pipe Networks](docs/api/PIPES.md)
- [Thermal & Energy](docs/api/THERMAL-ENERGY.md)
- [Elements & Cells](docs/api/ELEMENTS.md)

More pages for extension properties, attributes, events, scheduling, persistence, rooms, deterministic state, diagnostics, and tooling are being added as their source audits are completed.

---

# Architecture

```text
                         Oxygen Not Included
                                  │
                         flagship / 3rd-party mods
                                  │
                         ┌────────▼────────┐
                         │   oni-framework │
                         │    managed SDK  │
                         └────────┬────────┘
                                  │
                         ┌────────▼────────┐
                         │ ONI-Sim-Custom  │
                         │ native physics  │
                         │ + extension ABI │
                         └────────┬────────┘
                                  │
                         extended simulation

                    ┌──────────────────────────┐
                    │          SimViz           │
                    │ offline / live inspection │
                    └──────────────────────────┘
```

The central rule is:

> **New simulation capabilities belong in the custom SimDLL and are exposed to mods through `oni-framework`. Flagship and third-party mods should not privately reach into the native DLL.**

`ONI-Sim` remains the frozen 1:1 vanilla replacement. `ONI-Sim-Custom` is the extension implementation. `oni-framework` is the managed modder contract.

---

# Showcase Rigs

These are real executable demonstrations using the same framework surfaces intended for third-party mods.

> **Videos:** The demonstrations live on the [`Video-Uploads` branch](https://github.com/Salacious-Oni-Dev/API-Pre-Release-Preview/tree/Video-Uploads). GitHub provides its native video viewer for the committed `.mp4` files; the showcase pages below provide the technical context and API cross-reference.

| Showcase | What it demonstrates | Video |
|---|---|---|
| **AirLoop** | Multi-species atmosphere, oxygen partial pressure, composition-aware breathing | [▶ Watch AirLoop](https://github.com/Salacious-Oni-Dev/API-Pre-Release-Preview/blob/Video-Uploads/airloop.mp4) |
| **PhaseLoop** | Pressure-driven phase change, latent heat, gas/liquid bridging, work and heat transfer | [▶ Watch PhaseLoop](https://github.com/Salacious-Oni-Dev/API-Pre-Release-Preview/blob/Video-Uploads/phaseloop.mp4) |
| **PipeStress** | Pressure, condensation, freezing, standing matter and rupture/stress evaluation | [▶ Watch PipeStress](https://github.com/Salacious-Oni-Dev/API-Pre-Release-Preview/blob/Video-Uploads/pipestress.mp4) |
| **SimViz** | Offline/live simulation inspection, history playback, extension data and diagnostics | [▶ Watch SimViz](https://github.com/Salacious-Oni-Dev/API-Pre-Release-Preview/blob/Video-Uploads/simviz-showcase.mp4) |

For the full technical treatment, use the dedicated showcase pages once published.

## AirLoop

**Demonstrates:** multi-species atmosphere, oxygen partial pressure, composition-aware breathing, contamination, and atmospheric assessment.

**Video:** [▶ Watch AirLoop](https://github.com/Salacious-Oni-Dev/API-Pre-Release-Preview/blob/Video-Uploads/airloop.mp4)

AirLoop seals an atmosphere, mixes species to a target oxygen partial pressure, and checks what a duplicant actually breathes. It demonstrates why a real atmosphere needs composition rather than a single-element cell model.

**Primary references:** [Gas Mixture](docs/api/GAS-MIXTURE.md) · [Atmosphere](docs/api/ATMOSPHERE.md)

## PhaseLoop

**Demonstrates:** pressure-driven phase change, latent heat, gas/liquid bridging, work, and heat transfer rather than heat deletion.

**Video:** [▶ Watch PhaseLoop](https://github.com/Salacious-Oni-Dev/API-Pre-Release-Preview/blob/Video-Uploads/phaseloop.mp4)

The loop moves matter between gas and liquid networks while measuring the cold-side/hot-side energy result. It demonstrates thermodynamic machinery rather than scripted temperature changes.

**Primary references:** [Pipe Matter](docs/api/MATTER.md) · [Pipe Networks](docs/api/PIPES.md) · [Thermal & Energy](docs/api/THERMAL-ENERGY.md)

## PipeStress

**Demonstrates:** pipe pressure, condensation, freezing, standing matter, and rupture/stress evaluation.

**Video:** [▶ Watch PipeStress](https://github.com/Salacious-Oni-Dev/API-Pre-Release-Preview/blob/Video-Uploads/pipestress.mp4)

The 12-tile test run moves through baseline, mass overpressure, thermal overpressure, condensation, freezing, and rupture/stress states. A key result is that stress evaluation must account for **standing liquid and standing solid matter**, not only flowing contents.

**Primary reference:** [Pipe Networks](docs/api/PIPES.md)

---

# SimViz

**Demonstrates:** offline simulation inspection, captured-history playback, live attachment, cell inspection, extension properties, element attributes, events, state dumps, statistics, snapshots, and headless testing/rendering.

**Video:** [▶ Watch SimViz](https://github.com/Salacious-Oni-Dev/API-Pre-Release-Preview/blob/Video-Uploads/simviz-showcase.mp4)

SimViz is a **separate native application**, not an in-game graphics injector. It can inspect the same simulation information from captured/offline data or a running ONI process through framework bulk/debug routes.

---

# Documentation standard

This reference is being produced by reading the **actual source declarations and implementations**, then tracing important APIs through their managed and native layers. A filename is not treated as proof that a type is public or supported.

For significant surfaces, the manual aims to document:

1. exact public declarations;
2. parameter and return semantics;
3. units;
4. timing/threading considerations;
5. allocation/performance behavior;
6. stock-SimDLL behavior;
7. native extension dependency;
8. actual flagship consumers;
9. limitations and deliberate stubs;
10. small practical usage examples.

APIs are also classified as **modder-facing**, **developer/diagnostic**, **internal plumbing**, or **experimental/stubbed**.

---

# Integration rule

Consuming mods should reference `OniFramework.dll` as a shared framework dependency rather than bundling their own copy. `FrameworkVersion.Require(major, minor, consumer)` declares an API floor, while duplicate-instance detection protects against accidentally loading multiple framework copies.

Normal gameplay mods should use the public framework facade appropriate to the capability they need. `OniExtMessages` and direct native P/Invoke are implementation plumbing, not the intended modder contract.

---

# Preview philosophy

This project is intended to answer the questions an SDK user actually has:

> **What can I ask the simulation? What can I change? What does the value mean? What does it cost? What happens with a stock DLL? What native feature is underneath it? What are the limitations? And how would I actually use it in a mod?**

The detailed documents exist to answer those questions without hiding the important implementation boundaries behind a shallow list of class names.
