# API Reference

This is the **pre-release SDK reference** for the managed `oni-framework` surface.

The documentation is being built from source declarations and implementations. A class appearing in the repository is not, by itself, treated as proof that every member is a supported modder API.

## Core physical simulation

- [Gas Mixture](GAS-MIXTURE.md) — multi-species cell composition, pressure, conversion, room promotion, and native-backed thermodynamic calculations.
- [Atmosphere](ATMOSPHERE.md) — partial-pressure breathing, temperature, contamination, and the current combustion-risk stub.
- [Material Properties](MATERIALS.md) — molecular mass, latent heat, vapor-pressure curves, liquid volume, critical pressure, gamma, and working-gas properties.
- [Pipe Matter](MATTER.md) — standing liquid/solid matter that vanilla conduit contents cannot represent.
- [Pipe Networks](PIPES.md) — connected-network state, species, pressure/fill state, stress levels, and backpressure.
- [Thermal & Energy](THERMAL-ENERGY.md) — thermal mass, power heat, exhaust heat, conversion enthalpy, turbine heat, and the energy ledger.
- [Elements & Cells](ELEMENTS.md) — custom element registration and related cell/chunk capabilities.

## More reference areas

The remaining framework surfaces are being audited and will be documented in dedicated pages rather than being collapsed into a filename-derived inventory. Planned/reference areas include:

- simulation extension registration;
- per-cell extension properties;
- element attributes;
- event streams;
- frame and phase scheduling;
- persistence/checkpoints;
- simulation rooms;
- deterministic random state;
- disease-growth state;
- profiling and corpus tooling;
- debug inspector/bulk routes;
- rig/demo infrastructure.

## Documentation standard

For each significant modder-facing surface, the goal is to document:

1. exact public declarations;
2. parameter and return semantics;
3. units;
4. timing/threading considerations;
5. allocation behavior where relevant;
6. stock-SimDLL compatibility behavior;
7. native extension dependency;
8. actual flagship consumers;
9. limitations and deliberate stubs;
10. small practical usage examples.

See the [main preview README](../../README.md) for project context and release status.
