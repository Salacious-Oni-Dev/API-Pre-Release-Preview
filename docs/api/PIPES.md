# Pipe Network API

## `PipeNetworkFacade`

`PipeNetworkFacade` exposes the connected-run view of gas and liquid conduits. Vanilla ONI's basic conduit query is naturally tile-oriented; this facade aggregates the connected network into the unit a Stationeers-style pipe system actually reasons about.

## Content types

```csharp
public enum PipeContentType
{
    Gas,
    Liquid
}
```

## Stress levels

```csharp
public enum PipeStressLevel
{
    None = 0,
    Stressed = 1,
    Critical = 2
}
```

`Stressed` represents the warning tier. `Critical` represents the point at which the pipe damage model applies real damage.

## Stress kinds

```csharp
public enum PipeStressKind
{
    None = 0,
    Overpressure = 1,
    Condensation = 2,
    Freezing = 3,
    Vaporization = 4,
    StandingLiquid = 5,
    StandingSolid = 6
}
```

The distinction between **approach** and **aftermath** matters:

- `Condensation` means the gas is approaching its pressure-dependent condensation condition.
- `StandingLiquid` means liquid is already physically sitting in a gas network.
- `Freezing` is the thermal approach to solidification.
- `StandingSolid` means solid matter is already obstructing the network.
- `Vaporization` is the liquid-network counterpart of condensation.

This prevents a finished phase transition from making a pipe appear healthy simply because its original gas state no longer exists.

## `PipeSpecies`

```csharp
public readonly struct PipeSpecies
{
    public readonly int ElementIdx;
    public readonly float MassKg;
    public readonly float Moles;
    public readonly float MoleFraction;
}
```

`Moles` uses the framework's molecular-mass registry rather than blindly using ONI's element `molarMass` field. This matters for species where ONI's content data represents atomic mass for a diatomic gas.

## `PipeNetworkState`

Aggregate network state includes:

```csharp
public PipeContentType ContentType;
public int[] Cells;
public float VolumeLitres;
public float TotalMassKg;
public float TemperatureK;
public float PressurePa;
public float TotalMoles;
public float FillFraction;
public float LiquidVolumeLitres;
public PipeSpecies[] Contents;
```

`Cells` is ordered in ascending cell order and is a borrowed cache array. Consumers must not modify it.

Gas pressure is reported in Pa. Liquid networks use volume/fill state instead of ideal-gas pressure; `PressurePa` is zero for a liquid network.

Liquid `FillFraction` is intentionally **not clamped**. A value above 1 is evidence of over-capacity and should not be hidden by the API.

## `PipeNetworkReading`

For hot paths, the framework provides a value-style network reading:

```csharp
public struct PipeNetworkReading
{
    public PipeContentType ContentType;
    public int NetworkId;
    public int[] Cells;
    public int CellCount;
    public int SpeciesCount;
    public float VolumeLitres;
    public float TotalMassKg;
    public float TotalMoles;
    public float TemperatureK;
    public float PressurePa;
    public float LiquidVolumeLitres;
    public float FillFraction;
}
```

The cells array is borrowed from the network cache. `NetworkId` is only stable for the lifetime of the current topology version and must not be persisted across construction/deconstruction changes.

## `PipeStressReport`

```csharp
public readonly struct PipeStressReport
{
    public readonly PipeStressLevel Level;
    public readonly PipeStressKind Kind;
    public readonly int WorstCell;
    public readonly int ElementIdx;
    public readonly float Value;
    public readonly float Limit;
    public readonly float Severity;
}
```

`Severity` is normalized so different failure kinds can be ranked without comparing incompatible units.

`WorstCell` identifies the member that should receive the warning/notification marker.

## Stress model

The facade uses Stationeers-derived stress thresholds while adapting them to ONI's split gas/liquid conduit architecture. Gas pressure is evaluated against the surrounding environment; liquid networks use their available volume/flow representation. Thermal and standing-matter checks use the same quantities that the phase-change machinery produces.

The important SDK property is **one answer for both diagnostics and gameplay**. A mod should not independently invent a pressure threshold for its warning UI and a different threshold for rupture logic.

## Example: network safety monitor

```csharp
if (PipeNetworkFacade.TryEvaluateStress(cell, out var stress))
{
    if (stress.Level == PipeStressLevel.Critical)
    {
        ShowPipeWarning(stress.WorstCell, stress.Kind, stress.Severity);
    }
}
```

The exact overloads available in the released build should be taken from the versioned source/API package; this example illustrates the intended consumption pattern: ask the framework for the authoritative network verdict and use the report rather than reimplementing the thresholds.

## `ConduitBackpressure`

`ConduitBackpressure` addresses a separate vanilla assumption: `pipesHaveRoom` historically means that an output tile is **empty**, not merely that it has remaining capacity.

### API

```csharp
public static bool IsInstalled { get; }
public static void Install(Harmony harmony);
public static bool HasRoom(ConduitType conduitType, int cell);
```

The replacement checks ONI's own effective-capacity calculation and only changes a vanilla refusal into an acceptance when the tile actually has room. Existing vanilla `ignoreFullPipe` behavior remains authoritative.

Solid conduits are deliberately left on vanilla behavior.

### Example

```csharp
if (ConduitBackpressure.HasRoom(ConduitType.Liquid, outputCell))
{
    // The connected liquid conduit can accept more matter.
    ContinueProduction();
}
```

## Performance notes

The network cache exists specifically to avoid repeated flood fills and per-tick allocations. Borrowed arrays must be treated as read-only. For frequent sensors, prefer the value-style reading and caller-owned composition buffers where available.

## Related APIs

- [Pipe Matter](MATTER.md)
- [Gas Mixture](GAS-MIXTURE.md)
- [Material Properties](MATERIALS.md)
- [Thermal & Energy](THERMAL-ENERGY.md)
