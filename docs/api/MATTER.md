# Pipe Matter API

## `PipeMatterFacade`

`PipeMatterFacade` is the framework's managed side-car store for matter that ONI's ordinary `ConduitContents` cannot represent.

Vanilla conduit contents have a single element/mass/temperature shape. A two-phase pipe needs to be able to retain condensed liquid or frozen solid while the conduit network itself is still represented by its normal flow model. `PipeMatterFacade` supplies that missing per-tile storage.

The store is deliberately keyed by **cell and conduit type**. A single cell can legitimately contain both a gas conduit and a liquid conduit, so those states are independent.

## Core type

### `TrappedMatter`

```csharp
public struct TrappedMatter
{
    public int ElementIdx;
    public float MassKg;
    public float TemperatureK;
}
```

The element is a real ONI element. Phase is therefore derived from the element's own phase state rather than tracked as a second, potentially inconsistent flag.

## Reading stored matter

### `TryGet`

```csharp
public static bool TryGet(
    bool gasConduit,
    int cell,
    out TrappedMatter matter);
```

Returns the standing matter stored for that conduit type and cell.

### `TrackedCells`

```csharp
public static IEnumerable<int> TrackedCells(bool gasConduit);
```

Returns a snapshot of currently tracked cells. The returned collection is a copy so callers can mutate the store while walking the snapshot.

### Example: inspect standing matter

```csharp
if (PipeMatterFacade.TryGet(true, cell, out var matter))
{
    Element element = ElementLoader.elements[matter.ElementIdx];
    Debug.Log($"Standing {element.name}: {matter.MassKg:F3} kg @ {matter.TemperatureK:F1} K");
}
```

## Writing matter

### `Add`

```csharp
public static void Add(
    bool gasConduit,
    int cell,
    int elementIdx,
    float massKg,
    float temperatureK);
```

Adds matter to the side-car store and handles temperature/phase transitions using the framework's material-property and energy rules.

A particularly important conservation fix in this implementation is that an existing standing entry is not silently overwritten when a new arrival has a different element representing another phase of the same substance. The store has one entry per tile, so compatible phases are resolved into the existing physical substance rather than deleting the old mass.

## Persistence boundary

Persistence is intentionally **not owned by `PipeMatterFacade`**.

The in-memory store is connected to persistence through:

```csharp
public static Action<bool, int> MirrorHook;
```

The hook is called after mutations. It is optional; without a hook the store remains valid in memory but is not persisted by this facade.

This separation is deliberate because moving the serialized `KMonoBehaviour` between assemblies would change KSerialization's component identity and could invalidate existing saves.

## Temperature floor diagnostics

The facade uses:

```csharp
public const float MinimumConduitTemperatureK = 1f;

public static int TemperatureFloorHits { get; }
public static double TemperatureFloorUnbilledJoules { get; }
```

The 1 K floor matches the simulation kernel's own minimum rather than allowing a managed phase-change path to leave real mass at 0 K.

If an endothermic bill would drive a tile below the floor, the occurrence is counted and the energy that could not be delivered is recorded rather than silently disappearing.

## Matter accounting diagnostics

The facade exposes cumulative counters useful to deterministic test rigs:

```csharp
public static float ReleasedGasConduitKg { get; }
public static float ReleasedLiquidConduitKg { get; }
public static float SolidFormedGasConduitKg { get; }
public static float SolidFormedLiquidConduitKg { get; }
public static float CondensedInGasConduitKg { get; }
public static float BoiledInLiquidConduitKg { get; }
public static float DrainedFromGasConduitKg { get; }
public static float DrainedFromLiquidConduitKg { get; }
```

These counters solve a practical testing problem: transient phase products may be drained or released before a later assertion can observe them standing in the pipe. Counting the deterministic formation/release/drain event gives a stable assertion target.

## Why this is an SDK capability

This is intentionally not buried in a condensation-valve implementation. Any mod that needs matter the vanilla conduit representation cannot hold can use the same framework store and physics boundary.

Typical applications:

- condensers;
- evaporators;
- purge valves;
- phase separators;
- cryogenic storage;
- pipe-freezing systems;
- pressure relief;
- custom fluid machinery.

## Related APIs

- [Gas Mixture API](GAS-MIXTURE.md)
- [Pipe Networks](PIPES.md)
- [Material Properties](MATERIALS.md)
- [Thermal & Energy](THERMAL-ENERGY.md)
