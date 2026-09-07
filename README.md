# ONI API Pre-Release Preview

> **Private technical preview** of the custom ONI simulation extension API, the managed `oni-framework` surface, and SimViz.
>
> This repository is a showcase and API reference for modders. It is intentionally separate from the implementation repositories.

---

## What this preview demonstrates

This preview sits on top of three layers:

```text
                         Oxygen Not Included
                                  │
                         flagship / 3rd-party mods
                                  │
                         ┌────────▼────────┐
                         │   oni-framework │
                         │ managed API     │
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

The design rule is simple: **new simulation capabilities belong in the custom SimDLL and are made available to mods through `oni-framework`; flagship mods should not privately reach into the native DLL.**

`ONI-Sim` remains the frozen 1:1 vanilla replacement. `ONI-Sim-Custom` is the branch where the native extension surface grows, while `oni-framework` provides the stable managed contract consumed by mods.

---

# Showcase Rigs

The three rigs below are executable demonstrations rather than synthetic diagrams. They use the same framework APIs intended for real mods and assert the resulting simulation state.

## 1. AirLoop

**Showcase:** breathable-air composition and partial-pressure simulation.

### Video

> **VIDEO UPLOAD STUB — AIRLOOP**
>
> Upload the AirLoop showcase video here.

### What it demonstrates

AirLoop builds a sealed atmosphere, mixes gas species to a target oxygen partial pressure, and verifies what a duplicant actually breathes. The important distinction is that air is treated as a **composition**, not simply as a single ONI element with an amount attached to it.

The rig demonstrates:

- per-cell gas species composition;
- volume fractions / mole fractions;
- oxygen partial pressure;
- CO₂ and toxic-gas fractions;
- temperature of the mixture;
- breathing-quality evaluation against the actual simulated atmosphere;
- agreement between the managed framework view and the simulation-side state.

### Framework APIs used

- `GasMixtureFacade`
- `AtmosphereFacade`
- `MaterialPropertyRegistry`
- `MaterialProperties`
- `ElementChunkFacade`
- `ElementRegistry`
- simulation-clock / frame infrastructure

### Why this matters to modders

A mod can use these APIs to build systems that care about **what gas is actually present**, rather than assuming one element represents the entire atmosphere. Examples include:

- breathable-air systems;
- gas sensors;
- toxic atmosphere detection;
- combustion and flammability systems;
- gas purification;
- atmospheric analyzers;
- environmental hazards;
- custom life-support logic;
- Stationeers-style gas processing.

---

## 2. PhaseLoop

**Showcase:** pressure-driven phase change, latent heat, and energy-conserving gas/liquid circulation.

### Video

> **VIDEO UPLOAD STUB — PHASELOOP**
>
> Upload the PhaseLoop showcase video here.

### What it demonstrates

PhaseLoop is a condensation / purge cooling loop connecting a cold room to a hot room through separate gas and liquid networks.

It demonstrates that phase change is not a magical disappearance of matter or energy:

```text
cold room → gas → pressure-driven condensation
                         │
                    latent heat
                         │
                         ▼
                    liquid loop
                         │
                     pump/work
                         │
                         ▼
                    hot room
```

The rig measures the simulated result and verifies that heat leaves the cold side and arrives on the hot side rather than being deleted.

### Framework APIs used

- `GasMixtureFacade`
- `PipeMatterFacade`
- `PipeNetworkFacade`
- `MaterialProperties`
- `MaterialPropertyRegistry`
- `ConversionEnthalpy`
- `ConversionEnthalpyRule`
- `EnergyLedgerFacade`
- `PowerHeat`
- `ConduitBackpressure`
- `SimExtPhases`
- `SimExtFrame`

### Why this matters to modders

These APIs make it possible to implement real thermodynamic machinery rather than scripted temperature changes. Potential uses include:

- condensers and evaporators;
- refrigeration;
- heat pumps;
- industrial gas processing;
- liquid/gas storage;
- pressure-controlled phase separators;
- realistic generators and turbines;
- machinery with explicit work and waste heat;
- Stationeers-inspired fluid systems.

---

## 3. PipeStress

**Showcase:** pipe pressure, standing matter, condensation, freezing, and rupture.

### Video

> **VIDEO UPLOAD STUB — PIPESTRESS**
>
> Upload the PipeStress showcase video here.

### What it demonstrates

PipeStress drives a real 12-tile gas run through multiple physical states:

1. baseline operation;
2. overpressure caused by mass;
3. overpressure caused by heat;
4. condensation;
5. freezing;
6. rupture / stress response.

The rig checks both the framework's physical answer and the resulting ONI selectable/status presentation.

A particularly important part of the implementation is **standing matter**. A pipe can finish a phase conversion with most or all of its contents as liquid or solid even when no active flow is occurring. Pipe stress therefore cannot be based solely on flowing contents.

`PipeNetworkFacade` exposes standing liquid and standing solid state so pressure/stress calculations and UI status can use the same physical quantity.

### Framework APIs used

- `PipeNetworkFacade`
- `PipeMatterFacade`
- `GasMixtureFacade`
- `MaterialProperties`
- `MaterialPropertyRegistry`
- `SimExtPhases`
- `SimExtFrame`
- `ConduitNetworks`
- `ConduitBackpressure`
- `ElementRegistry`
- `RigDiagnostics`

### Why this matters to modders

This enables systems such as:

- pipe burst mechanics;
- pressure ratings;
- freezing damage;
- condensation damage;
- fluid storage limits;
- industrial safety systems;
- pressure relief valves;
- gas/liquid network analyzers;
- realistic pipe temperature and stress displays.

---

# SimViz

**SimViz** is the standalone visualization and diagnostics application for the extended simulation surface.

### Video

> **VIDEO UPLOAD STUB — SIMVIZ SHOWCASE**
>
> Upload the SimViz showcase video here.

### What SimViz does

SimViz can work with captured simulation data, drive the simulation offline, and attach to a running ONI instance through the framework's bulk/debug routes.

It provides:

- offline simulation playback;
- history scrubbing;
- live attachment;
- individual-cell inspection;
- gas mixture inspection;
- extension-property inspection;
- element-attribute inspection;
- event-stream inspection;
- state dumps;
- statistics and simulation digests;
- simulation-test support;
- PNG snapshots;
- camera/view state;
- headless rendering for automated testing;
- multiple simulation visualization modes.

The key architectural benefit is that **the same simulation state can be inspected whether it came from an offline corpus or a live ONI process.**

SimViz is not an in-game graphics injector. It is a separate simulation visualization, inspection, and validation tool.

---

# `oni-framework` API Reference

The framework is the managed modder-facing contract around the custom simulation capabilities. The following is the framework's exposed source surface as currently organized in `OniFramework/`.

## Gas, atmosphere, and mixtures

### `GasMixtureFacade`

Provides access to the native per-cell gas mixture.

**Exposes:** gas species composition, pressure, temperature, species injection/withdrawal, conduit pressure/liquid-fill reads, phase-change calculations, and adiabatic-fill temperature calculations.

**Useful for:** gas processing, atmospheric simulation, breathing systems, sensors, combustion, life support, and industrial machinery.

### `AtmosphereFacade`

Provides a composition-aware atmosphere query intended for gameplay systems that need a single efficient read.

**Exposes:** oxygen partial pressure, temperature, CO₂ fraction, combined toxic fraction, and atmosphere grading. Includes a combustion-risk evaluation surface.

**Useful for:** breathing, environmental hazards, atmosphere quality, combustion-risk UI, and AI decisions.

### `MaterialProperties`

Represents physical material constants not supplied by vanilla `Element`.

**Exposes:** latent heat, heat-capacity ratio, vapor-pressure curve, minimum liquid pressure, critical point, and liquid density.

**Useful for:** phase change, fluid machinery, thermodynamics, and realistic material behavior.

### `MaterialPropertyRegistry`

Central registry for material properties.

**Important design point:** facades use this registry rather than duplicating physical constants in individual mods.

---

## Pipes and matter

### `PipeMatterFacade`

Access to conduit matter storage and manipulation.

**Exposes:** pipe matter state and operations including local latent-heat billing to the cell.

**Useful for:** custom pipes, phase-change machines, storage, fluid processing, and conservation-aware machinery.

### `PipeNetworkFacade`

Network-level pipe queries and state.

**Exposes:** network state, pressure-related information, standing matter, `StandingLiquid`, `StandingSolid`, and stress-relevant quantities.

**Useful for:** pipe analyzers, pressure limits, burst mechanics, freezing/condensation safety, and network diagnostics.

### `ConduitNetworks`

Shared conduit-network infrastructure used by the framework's pipe/conduit APIs.

### `ConduitBackpressure`

Framework surface for conduit backpressure behavior and diagnostics.

---

## Elements and cell data

### `ElementRegistry`

Registers custom ONI elements through the managed framework.

**Useful for:** adding real elements such as Nitrogen, Liquid Nitrogen, Pollutant variants, and other mod-defined materials.

### `ElementChunkFacade`

Public managed access to vanilla `ElementChunk` / `SimTemperatureTransfer` behavior.

**Useful for:** mods that need to reason about off-grid mass and its thermal interaction with a cell without reimplementing vanilla behavior.

### `ThermalMassBonus`

Adds additional cell heat capacity on top of vanilla `mass × specificHeatCapacity`.

**Useful for:** insulation-like systems, special environments, machinery thermal mass, custom materials, and large thermal reservoirs.

---

## Heat, energy, and thermodynamics

### `PowerHeat`

Maps building electrical consumption into real heat at the building.

**Useful for:** machinery that obeys energy conservation and systems inspired by real industrial devices.

### `ExhaustHeat`

Redirects vanilla exhaust-energy destruction paths into the building body.

**Useful for:** realistic exhaust and waste-heat systems.

### `Radiation`

A deliberately bounded extension point for radiated heat. The current surface acts as a reservoir/stub pending a larger environment model.

**Useful for:** future planetary/environmental radiation systems.

### `TurbineHeat`

Reworks Steam Turbine waste-heat handling so rejected energy is represented as heat rather than silently disappearing.

**Useful for:** heat engines, turbines, generators, and thermodynamic chains.

### `ConversionEnthalpy`

Provides enthalpy-aware conversion accounting for `ElementConverter`-style transformations.

**Useful for:** conversion machinery, geysers, generators, and any system where temperature-only accounting can violate energy conservation.

### `ConversionEnthalpyRule`

Rule/configuration surface for applying enthalpy-aware conversion behavior.

### `GeneratorEnthalpy`

Generator-oriented enthalpy handling.

### `StirlingCycleRule`

Thermodynamic rule surface for Stirling-cycle style machinery.

### `ColdBreatherHeat`

Thermal handling associated with cold/breathing interactions.

### `EnergyLedgerFacade`

Read access to the custom SimDLL energy-conservation ledger.

**Useful for:** debugging, validation, balancing, mod testing, and proving where heat/energy came from during a simulation run.

---

# Simulation control and state

### `SimRandom`

Framework access to simulation random state/checkpoint handling.

**Useful for:** deterministic tests, replay, checkpointing, and reproducible simulation experiments.

### `SimScheduling`

Access to simulation scheduling state.

**Useful for:** deterministic simulation tests and extension scheduling.

### `SimDiseaseGrowth`

Simulation disease-growth state extension.

**Useful for:** deterministic disease simulation extensions and state capture.

### `SimRegistryState`

Registry-state persistence/checkpoint surface.

### `SimExtCellState`

Extended per-cell state storage.

**Useful for:** custom simulation state that needs to live alongside cells.

### `SimStableTicks`

Stable-tick state/control used by deterministic simulation infrastructure.

### `SimGasSleeping`

Gas-sleeping state integration for the extended simulation surface.

---

# Native extension / registration API

These types form the managed layer over the native extension ABI. They are intentionally separated from gameplay facades so the framework can own P/Invoke, message layout, registration, persistence, and compatibility behavior.

### `OniExtMessages`

Internal managed bridge to the custom SimDLL extension message ABI.

**Modder guidance:** use a public facade rather than calling this directly.

### `SimExtRegistry`

Extension registration and registry management.

### `SimExtRegistrar`

Registration helper for custom simulation extensions.

### `SimExtCellProperties`

Registers and accesses custom per-cell properties.

### `SimExtElementAttributes`

Registers and accesses custom element attributes.

### `SimExtEventStreams`

Creates/subscribes to extension event streams.

### `SimExtFrame`

Frame-level extension coordination.

### `SimExtPhases`

Defines/coordinates extension phase and execution scope information.

### `SimExtMessages`

Managed message definitions and message-surface helpers.

---

# Rooms and environment

### `SimRooms`

Framework access to extended room promotion/state.

**Useful for:** custom room classification, environmental systems, and extensions that need a simulation-native room concept.

---

# Profiling, testing, and corpus infrastructure

These are primarily developer/diagnostic APIs rather than normal gameplay APIs.

### `SimProfile`

Simulation profiling/measurement support.

### `SimCorpus`

Captured simulation corpus support for repeatable offline runs and regression testing.

### `SimExtSelfTest`

Native/managed extension self-test surface.

### `RigHarness`

Base harness used by the showcase rigs.

### `RigRegistry`

Registers executable test/showcase rigs.

### `RigBlueprint`

Declarative blueprint representation used by rig construction.

### `BlueprintBuilder`

Builds test/showcase worlds from blueprints.

### `BlueprintWorld`

World construction/state support for blueprint rigs.

### `BlueprintUtilities`

Shared blueprint helpers.

### `BlueprintDump`

Blueprint inspection/dump support.

### `BlueprintRig`

Rig implementation support around blueprint worlds.

### `RigDiagnostics`

Assertions, diagnostics, and rig result reporting.

### `RigCrashScreen`

Failure presentation for showcase/test rigs.

### `DemoRig`

Common demonstration-rig infrastructure.

### `DemoNarrator`

Presentation/narration support used by showcase demonstrations.

### `FrameGeometry`

Simulation-frame geometry helpers.

### `StructureTemperatureGuard`

Safety/validation around structure temperature changes in constructed test environments.

### `NotificationSafety`

Safe notification behavior for managed simulation/demo surfaces.

### `DebugInspectorServer`

Live debug/inspection server used by external tooling.

### `DebugBulkRoutes`

High-throughput bulk routes for published grid arrays. These are designed to transfer whole simulation arrays efficiently rather than issuing one JSON request per cell.

### `RigSaves`

Rig save support.

### `SaveMigrationHandler`

Save migration support for framework/rig state.

### `FrameworkLog`

Framework logging surface.

### `FrameworkVersion`

Framework version/compatibility contract.

`FrameworkVersion.Require(major, minor, consumer)` lets a consuming mod declare the minimum framework API level it requires.

`CheckSingleInstance()` detects duplicate copies of `OniFramework.dll` instead of allowing assembly duplication to cause obscure runtime failures.

### `SimVersion`

Simulation-version discovery and compatibility support.

### `BuildStamp`

Build/version provenance information.

### `HoverCard`

Showcase/debug UI helper.

---

# Native custom SimDLL extension surface

`ONI-Sim-Custom` retains the vanilla SimDLL export surface while adding a dedicated extension ABI.

The custom ABI currently adds **48 native exports** and recognizes **23 additional extension message IDs** beyond the vanilla interface.

The extension header defines the ABI contract, payload layouts, descriptors, delivery semantics, phase scopes, persistence classes, registration rules, and published-property surface.

## Additional message IDs

### Physics and material operations

- `ONI_MSG_SET_CELL_THERMAL_MASS_BONUS`
- `ONI_MSG_INJECT_GAS_SPECIES`
- `ONI_MSG_REMOVE_VANILLA_MASS`
- `ONI_MSG_CONVERT_TO_VANILLA_MASS`
- `ONI_MSG_PROMOTE_ROOM`
- `ONI_MSG_SET_INVERTED_GRAVITY_ELEMENT`
- `ONI_MSG_SET_MOLECULAR_MASS`
- `ONI_MSG_SET_BUILDING_WASTE_HEAT_KW`
- `ONI_MSG_SET_BUILDING_EXHAUST`
- `ONI_MSG_SET_BUILDING_RADIATION`
- `ONI_MSG_SET_ENVIRONMENT_TEMPERATURE`

### Deterministic state/checkpoint operations

- `ONI_MSG_SET_RANDOM_STATE`
- `ONI_MSG_SET_SCHEDULING_STATE`
- `ONI_MSG_SET_STABLE_TICKS`
- `ONI_MSG_SET_DISEASE_GROWTH`
- `ONI_MSG_SET_REGISTRY_STATE`
- `ONI_MSG_SET_EXT_CELL_STATE`

### Extension registration/state

- `ONI_MSG_REGISTER_CELL_PROPERTY`
- `ONI_MSG_SET_CELL_PROPERTY`
- `ONI_MSG_PUBLISH_CELL_PROPERTY`
- `ONI_MSG_REGISTER_ELEMENT_ATTRIBUTE`
- `ONI_MSG_SET_ELEMENT_ATTRIBUTE`
- `ONI_MSG_SUBSCRIBE_EVENT_STREAM`

---

# Native message semantics

The extension ABI distinguishes several message classes:

- **parameter** — configure a simulation capability;
- **operation** — request an action;
- **store** — write persistent extension state;
- **checkpoint** — participate in deterministic saved/checkpoint state.

Messages can be delivered:

- **immediately** during the call; or
- **queued** for the next simulation frame.

Extension execution scopes include:

- frame;
- substep;
- region;
- grid-in-region.

Persistence classifications include:

- saved;
- rehydrated;
- checkpoint-only.

The ABI uses fixed, explicit scalar types including `F32`, `U8`, `U16`, and `I32` and defines a maximum message arity of 64.

---

# Published extension properties

The custom simulation publishes these built-in property names:

- `sim.thermal_mass_bonus`
- `sim.gas_occupied_mask`
- `sim.gas_species`
- `sim.gas_mass`
- `sim.room_promoted`
- `sim.molecular_mass`
- `sim.message_refused`

The extension system supports up to 32 event streams, 1 MiB of event-stream bytes per frame, and 64 element attributes.

Registration failures are explicitly classified, including bad name, duplicate, reserved, bad type, closed, full, unsupported, and bad arity.

---

# Major custom-simulation capabilities

The custom SimDLL is not merely exposing more getters. It adds simulation primitives that the managed framework can turn into modder-facing systems.

## Per-cell gas mixtures

Cells can carry species composition rather than treating the gas contents as a single homogeneous element.

This enables partial-pressure calculations, composition-aware breathing, gas processing, and real mixture manipulation.

## Real thermodynamic phase change

The extension surface supports phase-change calculations with latent heat rather than simply changing an element and assigning an arbitrary temperature.

## Adiabatic compression heating

Gas filling/compression can account for thermodynamic temperature changes rather than treating pressure changes as thermally free.

## Additional thermal mass

A cell can receive extra thermal capacity through the extension surface.

## Material properties

The framework provides a central material-property model for constants required by phase-change and thermodynamic systems.

## Building waste heat

Power, exhaust, turbine, and other machinery energy paths can be represented as physical heat instead of disappearing through magic constants or unaccounted sinks.

## Environment temperature control

The extension can provide simulation-native environment temperature behavior for custom environmental systems.

## Extended rooms

Cells can be promoted into the extension's room model for systems that need a simulation-native room concept.

## Custom cell properties

Mods can register and publish additional per-cell scalar properties without hard-coding every property into the core simulation.

## Custom element attributes

Elements can carry extension-defined attributes through the registry surface.

## Event streams

The native simulation can publish structured event data for external tools and managed consumers.

## Deterministic state and checkpointing

Random state, scheduling state, disease-growth state, registry state, stable ticks, and extension cell state have explicit checkpoint/state surfaces. This is foundational for reproducible testing, simulation replay, and offline analysis.

## Energy accounting

The energy ledger provides instrumentation for determining where simulation heat/energy originated and where it went.

---

# How modders consume the API

## 1. Reference the framework

A consuming mod should reference `OniFramework.dll` rather than bundling a private copy into its own assembly.

The framework is intended to load as its own mod assembly so every consuming mod sees the same framework instance.

## 2. Declare the required API level

Use the framework version contract to declare the minimum API level required by the mod.

Conceptually:

```csharp
FrameworkVersion.Require(major, minor, "MyMod");
```

The exact version should match the preview/release the mod was built against.

## 3. Use facades, not native P/Invoke

A normal mod should consume:

```text
MyMod
  ↓
OniFramework facade
  ↓
OniExtMessages / native ABI
  ↓
ONI-Sim-Custom
```

rather than implementing its own P/Invoke layer.

This keeps native ABI details centralized and gives the framework a place to handle compatibility, validation, versioning, persistence, and future implementation changes.

## 4. Use the appropriate subsystem

Examples:

```text
Need gas composition?       → GasMixtureFacade
Need breathable air?        → AtmosphereFacade
Need material constants?    → MaterialPropertyRegistry
Need pipe state?             → PipeNetworkFacade
Need pipe matter operations? → PipeMatterFacade
Need extra heat capacity?   → ThermalMassBonus
Need building waste heat?   → PowerHeat / ExhaustHeat / TurbineHeat
Need phase-change energy?   → ConversionEnthalpy / GasMixtureFacade
Need energy accounting?     → EnergyLedgerFacade
Need custom cell data?       → SimExtCellProperties
Need custom element data?   → SimExtElementAttributes
Need simulation events?      → SimExtEventStreams
Need deterministic tests?    → SimRandom / SimScheduling / SimCorpus
```

---

# Why the facade architecture matters

The framework is deliberately more than a convenience wrapper.

It creates a single public home for capabilities introduced by the custom simulation. That means a new mod does not need to know:

- native message IDs;
- payload packing;
- ABI details;
- export-table layout;
- custom persistence rules;
- extension scheduling semantics;
- event-stream wire formats;
- or how the underlying native implementation changes over time.

This also prevents every flagship mod from becoming its own incompatible implementation of the same physics.

---

# Relationship to the vanilla SimDLL

`ONI-Sim` is the frozen 1:1 vanilla-compatible simulation implementation.

`ONI-Sim-Custom` preserves the vanilla interface while adding the extension surface described here.

The intended relationship is:

```text
ONI-Sim
  = frozen vanilla-equivalent baseline

ONI-Sim-Custom
  = baseline + native extension capabilities

oni-framework
  = managed modder-facing contract

oni-flagship
  = flagship demonstrations / content using the framework

SimViz
  = external visualization, diagnostics, replay and validation
```

This separation makes it possible to reason about vanilla compatibility independently from custom simulation development.

---

# Showcase philosophy

The rigs are deliberately physical demonstrations rather than scripted visual effects.

A showcase should answer questions such as:

- Did the simulation actually contain the gas composition being displayed?
- Did pressure actually drive the phase change?
- Did latent heat actually move energy?
- Did standing liquid/solid matter actually contribute to pipe stress?
- Did the managed API observe the same state the simulation calculated?
- Can the result be reproduced offline?

That is why the rigs contain assertions and diagnostics instead of simply changing sprites or displaying a predetermined number.

---

# Pre-release status

This repository is a **private pre-release preview**.

The API surface, naming, signatures, message layouts, and implementation details may change before a public release.

Current native ABI target:

- Windows x64;
- little-endian;
- IEEE-754 float32;
- 8-byte pointers;
- 4-byte packing.

The native extension ABI includes explicit discovery/version functions and compile-time/runtime validation intended to catch ABI mismatches rather than silently corrupting simulation state.

---

# Video upload checklist

Replace each placeholder with the final showcase video when ready:

- [ ] AirLoop showcase video
- [ ] PhaseLoop showcase video
- [ ] PipeStress showcase video
- [ ] SimViz showcase video

Suggested filenames:

```text
airloop.mp4
phaseloop.mp4
pipestress.mp4
simviz.mp4
```

---

# Source projects

This preview documents the architecture and public-facing surfaces implemented across the project's private repositories:

- `ONI-Sim` — frozen vanilla-equivalent SimDLL baseline.
- `ONI-Sim-Custom` — custom native simulation and extension ABI.
- `oni-framework` — managed modder-facing API.
- `oni-flagship` — flagship demonstrations and physical-system mods.
- `oni-simviz` — standalone simulation visualization/diagnostics.
- `oni` — project architecture, research, and integration planning.
- `kanim-asset-pipeline` — asset conversion/tooling.
- `oni-mod-compat` — managed compatibility/surface analysis.
- `oni-private-data` — private reference/development data.

This preview does not redistribute proprietary game assemblies or other private reference material.

---

## Final note

The central idea of this preview is that ONI's simulation can be extended without forcing every mod to reinvent the simulation boundary.

**Native simulation capability → managed framework facade → reusable modding API → demonstrable physical system → externally inspectable simulation state.**
