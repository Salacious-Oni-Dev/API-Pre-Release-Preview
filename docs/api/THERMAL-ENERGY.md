# Thermal & Energy API

This section documents the framework surfaces that turn heat and energy from implicit side effects into explicit, inspectable simulation quantities.

## `ThermalMassBonus`

Adds extra heat capacity to an individual simulation cell in J/K, on top of the element's normal `mass × specificHeatCapacity` contribution.

### API

```csharp
public static unsafe void Set(int cell, float bonusJoulesPerKelvin);
public static void Clear(int cell);
```

`Set` replaces the previous bonus for the cell; it does not accumulate with it. `Clear` is equivalent to setting the bonus to zero.

### Example

```csharp
// Give a machinery room an additional 50 kJ/K of thermal mass.
ThermalMassBonus.Set(roomCell, 50_000f);

// Later, remove the extension.
ThermalMassBonus.Clear(roomCell);
```

The message is native-backed. A stock SimDLL does not understand the extension message, so the call is a no-op there rather than a managed exception.

## `PowerHeat`

`PowerHeat` derives building heat from live electrical draw rather than relying exclusively on hand-authored self-heat constants.

### Core configuration

```csharp
public const float DefaultRatio = 1f;
public static float GlobalRatio { get; set; }
public static bool IsInstalled { get; }
```

The default ratio is 1.0: one unit of electrical energy drawn becomes one unit of building heat unless the building has an override.

### Per-building controls

```csharp
public static void SetRatio(
    string prefabId,
    float ratio,
    string reason);

public static void Exempt(
    string prefabId,
    string reason);

public static void SetBias(
    string prefabId,
    float kilowatts,
    string reason);

public static void ClearOverride(string prefabId);

public static float RatioFor(string prefabId);
public static float BiasFor(string prefabId);
public static IEnumerable<string> DescribeOverrides();
```

Overrides are intentionally named. The framework treats unexplained energy coefficients as a maintenance problem and records a human-readable reason for each override.

### Installation

The framework does not automatically patch the game merely by loading. A consuming mod opts in:

```csharp
public override void OnLoad(Harmony harmony)
{
    base.OnLoad(harmony);
    OniFramework.PowerHeat.Install(harmony);
}
```

Installation is idempotent.

### Reading the derived heat

```csharp
public static float DerivedKilowatts(GameObject go);
```

This returns the same derived power-to-heat quantity the rule uses, allowing a UI, sensor, or automation integration to report the value without duplicating the derivation.

### Native relationship

The managed side reads ONI's power-grid state because the SimDLL does not own the game's wire/electrical system. It sends the derived rate to the custom SimDLL using the building-waste-heat extension. The native simulation then integrates it with its own building heat-exchange timestep.

This separation is deliberate: managed code knows what the machine is consuming; native code knows how that energy affects the simulated thermal state.

## `ExhaustHeat`

Redirects vanilla building exhaust heat that would otherwise be lost when the destination cell cannot accept it.

### API

```csharp
public static bool IsInstalled { get; }
public static void Install(Harmony harmony);
```

Installation is idempotent and validates that the expected vanilla methods still exist. If a game update moves the patched methods, installation throws rather than silently leaving a deletion path active or a stale rate running.

The framework tracks useful diagnostics:

```csharp
public static int LastSweepCount { get; }
public static float LastSweepKilowatts { get; }
public static long SweepCount { get; }
```

### Native relationship

Exhaust is sent as a rate through the custom extension message. The native side performs the same fundamental per-cell delivery logic as vanilla, but undelivered energy can be redirected into the building body rather than disappearing.

The rate is integrated on the simulation's own clock rather than being reconstructed from managed `GameClock` timing.

## `TurbineHeat`

The turbine heat facade exposes the framework's Steam Turbine waste-heat behavior. The goal is to keep extracted energy represented as heat in the turbine/building system rather than deleting the rejected fraction.

The exact installation/configuration surface should be consulted from the versioned source reference before depending on it; it is treated separately from the more general `PowerHeat` rule.

## `ConversionEnthalpy`

Makes `ElementConverter` transformations account for input/output heat capacity and an explicit calibrated reaction-enthalpy term.

### Configuration

```csharp
public const float DefaultCalibrationTemperature = 300f;
public static float GlobalCalibrationTemperature { get; set; }
public static bool IsInstalled { get; }

public static void SetCalibrationTemperature(
    string prefabId,
    float kelvin,
    string reason);
```

The calibration temperature is a design point, not a claim that it is a measured reaction constant.

At the calibration point, a calibrated recipe reproduces vanilla's output temperature. Away from that point, the output temperature shifts according to the input heat capacity and output heat capacity while preserving the calibrated energy relationship.

### Rule tiers

`ConversionEnthalpyRule.Tier` classifies recipes as:

- `Source` — no inputs; there is no input sensible enthalpy to carry.
- `PhysicalSeparation` — mass-conserving separation with zero output floors; true enthalpy conservation.
- `Calibrated` — reaction enthalpy calibrated to reproduce vanilla at the design point.
- `Exempt` — deliberately left at vanilla behavior.

### Pure calculation API

```csharp
public static float Solve(
    Tier tier,
    float floorK,
    float inputHeatCapacity,
    float inputTemperature,
    OutputProfile profile,
    float calibrationK);

public static float EnthalpyKJPerConversion(
    Tier tier,
    float inputHeatCapacity,
    OutputProfile profile,
    float calibrationK);

public static bool MassConserves(
    float inputMassRate,
    float outputMassRate);
```

`ConversionEnthalpyRule` intentionally has no Unity/game types, making the arithmetic suitable for standalone regression tests.

### Example: inspect the rule tier

```csharp
ConversionEnthalpyRule.OutputProfile profile = BuildOutputProfile(converter);

if (ConversionEnthalpyRule.MassConserves(inputKgPerSecond, outputKgPerSecond) &&
    profile.AllFloorsZero)
{
    // This recipe is a physical separation candidate.
}
```

## `EnergyLedgerFacade`

Reads the custom SimDLL's energy ledger.

### Snapshot

The `Snapshot` includes instantaneous reservoirs such as:

- `GridKJ`
- `BuildingsKJ`
- `ConduitsKJ`
- `ChunksKJ`

and accounting buckets such as:

- `BuildingOperatingKJ`
- `BuildingWasteHeatKJ`
- `BuildingExhaustKJ`
- `RadiatedKJ`
- `PhaseChangeKJ`
- `ChunkExchangeKJ`
- `EmittedEnergyKJ`
- `ConsumedEnergyKJ`
- `MoverKJ`
- `WorldInitKJ`

The snapshot also exposes `PublishedFieldCount` and `HasExtendedBuckets` so an older custom DLL can be distinguished from a genuine zero-valued extended bucket set.

### Derived values

```csharp
snapshot.TotalKJ
snapshot.AccountedInflowKJ
snapshot.ExtendedInflowKJ
snapshot.AnchorKJ
```

`TotalKJ` is the sum of the four main reservoirs. `AnchorKJ` is `TotalKJ - AccountedInflowKJ` for the original ledger accounting boundary. It must not be interpreted as a completed global zero-drift proof: the source explicitly documents remaining unaccounted simulation channels.

### Example: diagnostic energy report

```csharp
if (EnergyLedgerFacade.TryRead(out var ledger))
{
    Debug.Log($"Stored energy: {ledger.TotalKJ:F2} kJ");
    Debug.Log($"Power-derived heat: {ledger.BuildingWasteHeatKJ:F2} kJ");
    Debug.Log($"Phase-change energy: {ledger.PhaseChangeKJ:F2} kJ");
}
```

The actual `TryRead` overload/signature should be taken from the matching framework build when implementing against a released package; the important contract is that unavailable ledger support returns failure rather than requiring a stock DLL to implement the extension export.

## Design principle

These APIs share a common rule: **energy should have an identifiable owner and path.**

A mod adding a heat source should be able to identify whether that energy entered through building operation, an explicit heat message, a phase transition, an emitter, or another named bucket. The ledger is diagnostic infrastructure; it does not pretend that every remaining vanilla loss has already been eliminated.
