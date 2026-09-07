# Gas Mixture API

> **Pre-release SDK reference.** This document describes the verified managed surface of `GasMixtureFacade` from source. It is intentionally more detailed than a class summary; implementation behavior is included where it affects mod authors.

## Purpose

`GasMixtureFacade` exposes the custom SimDLL's per-cell multi-species gas state to managed ONI mods.

Vanilla ONI represents a cell with one element, one mass, and one temperature. The custom simulation adds a composition layer so a cell can contain multiple gas species. `GasMixtureFacade` is the managed boundary to that state.

## Availability

Check `GasMixtureFacade.Available` before relying on native-backed functionality. The facade deliberately degrades when the required custom exports are unavailable rather than turning a missing extension into an unexplained mod crash.

The custom SimDLL is required for the native mixture operations. Mods should use the facade rather than importing SimDLL exports themselves.

## Core types

### `GasComponent`

A single gas species in a cell:

```csharp
public readonly struct GasComponent
{
    public readonly ushort ElementIdx;
    public readonly float MassKg;
}
```

- `ElementIdx` — ONI element-table index.
- `MassKg` — mass of that species in kilograms.

### `MaxSpeciesPerCell`

```csharp
public const int MaxSpeciesPerCell = 8;
```

The native composition query supports up to eight species per cell.

## Reading composition

### `TryGetComposition`

```csharp
public static GasComponent[] TryGetComposition(int cell);
```

Returns the composition for a cell. This form returns an array and is convenient for occasional inspection, UI, or diagnostics, but it is not the preferred pattern for a hot per-tick loop when allocations matter.

### `ReadComposition`

```csharp
public static int ReadComposition(
    int cell,
    ushort[] species,
    float[] massKg);
```

The caller supplies buffers. The method writes species indices and masses into those buffers and returns the number of entries.

Each buffer must be large enough for `MaxSpeciesPerCell` entries. This form exists specifically to avoid per-call allocation and is preferable for sensors, duplicant breathing ticks, or other repeated queries.

### Example: allocation-conscious gas sensor

```csharp
ushort[] species = new ushort[GasMixtureFacade.MaxSpeciesPerCell];
float[] masses = new float[GasMixtureFacade.MaxSpeciesPerCell];

int count = GasMixtureFacade.ReadComposition(cell, species, masses);
for (int i = 0; i < count; i++)
{
    ushort element = species[i];
    float massKg = masses[i];
    ProcessGasReading(element, massKg);
}
```

Allocate the buffers once and reuse them rather than allocating every simulation tick.

## Dominant gas

### `TryGetDominant`

```csharp
public static bool TryGetDominant(
    int cell,
    out ushort elementIdx,
    out float totalMassKg);
```

Returns the dominant gas species and the total gas mass represented by the query.

This is useful when a mod only needs a quick classification rather than the complete composition.

## Pressure

### `TryGetPressure`

```csharp
public static bool TryGetPressure(
    int cell,
    out float pressurePa);
```

Reads gas pressure in pascals.

The pressure path uses the simulation's native pressure calculation rather than a second, independently implemented C# equation. This keeps managed observations aligned with the simulation's own pressure model.

### Example: overpressure sensor

```csharp
if (GasMixtureFacade.TryGetPressure(cell, out float pressurePa) &&
    pressurePa > 500_000f)
{
    TriggerOverpressureWarning(cell, pressurePa);
}
```

## Vanilla/custom matter conversion

### `RemoveVanillaMass`

```csharp
public static void RemoveVanillaMass(int cell, float massKg);
```

Removes mass from the vanilla representation before custom mixture operations consume it.

### `ConvertFromVanilla`

```csharp
public static void ConvertFromVanilla(
    int cell,
    int speciesIdx,
    float massKg);
```

Moves specified vanilla mass into the custom mixture representation.

### `ConvertToVanilla`

```csharp
public static void ConvertToVanilla(
    int cell,
    int speciesIdx,
    float massKg,
    float temperatureK);
```

Moves custom mixture mass back into vanilla representation.

The conversion is deliberately conservation-oriented: the native operation removes the amount that actually exists and only converts what was successfully removed. A refused destination is refunded rather than silently destroying the removed mass.

This is important when integrating a custom system with ordinary ONI buildings that still expect `Grid.Element`, `Grid.Mass`, and `Grid.Temperature`.

## Room promotion

### `PromoteRoom`

```csharp
public static void PromoteRoom(int cell);
```

Promotes a cell into the custom mixture-backed room representation. The promotion is deferred by one simulation tick.

### `IsRoomOwned`

```csharp
public static bool IsRoomOwned(int cell);
```

Reports whether the custom mixture layer owns the cell.

This is a simulation-thread rendezvous and is appropriate for relatively infrequent queries such as a breathing evaluation, not for arbitrary per-frame polling of hundreds of cells.

### `DebugRoomOwnedRaw`

```csharp
public static int DebugRoomOwnedRaw(int cell);
```

Returns the raw room ownership state, including the diagnostic `-1` state used when the volume-fraction room graph cannot answer for the cell.

A known limitation is that the room graph is built at first activation; rooms dug later can be invisible to that graph. The raw diagnostic exists specifically to distinguish that condition from an ordinary unowned cell.

## Conduit constants

```csharp
public const float GasConduitVolumeM3 = 0.01f;
public const float LiquidConduitVolumeM3 = 0.02f;
```

These represent 10 L and 20 L respectively and are used by the conduit-facing calculations.

## Phase-change and compression calculations

The facade also exposes native-backed calculations for physical operations used by the flagship thermodynamics systems, including:

- pressure-driven phase-change steps with latent heat;
- adiabatic fill/compression temperature calculations;
- conduit pressure and liquid-fill observations.

These functions are intentionally simulation-facing calculations rather than gameplay rules. A mod can use them to build an evaporator, condenser, gas compressor, pressure regulator, or other machine while keeping the physical calculation in the simulation layer.

## Error and compatibility behavior

The facade distinguishes between an ordinary physical result and an unavailable native capability. Pure native-backed calculations may throw `InvalidOperationException` when the required export is unavailable; callers should check `Available` where appropriate.

This is different from the private native symbols such as `SIM_GasComposition` and `SIM_ComputeGasPressure`. Those are implementation plumbing and are not the modder contract.

## Design guidance

**Use this facade when:**

- you need actual multi-species gas state;
- you need pressure based on that state;
- you need controlled movement between vanilla and custom representations;
- you are implementing gas processing or atmospheric mechanics;
- you need phase-change calculations performed by the simulation layer.

**Do not:**

- call private SimDLL exports directly from a gameplay mod;
- recreate the native pressure equation independently;
- allocate composition arrays every simulation tick when `ReadComposition` can be used;
- assume `Grid.Element[cell]` tells you the complete atmosphere once a cell is mixture-backed.

## Related APIs

- [Atmosphere API](ATMOSPHERE.md)
- [Material Properties](MATERIALS.md)
- [Pipe Matter](MATTER.md)
- [Pipe Networks](PIPES.md)
- [Native ABI](../native/ABI.md)
