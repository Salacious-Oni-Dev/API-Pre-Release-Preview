# Atmosphere API

## `AtmosphereFacade`

`AtmosphereFacade` provides one composition-aware assessment of a cell's air. It is designed for systems such as breathing, environmental hazards, sensors, and combustion analysis.

The facade does not apply gameplay effects. It measures and grades the atmosphere; gameplay code decides what those grades mean.

## Grades

### Oxygen

```csharp
public enum OxygenGrade
{
    None = 0,
    Critical = 1,
    Low = 2,
    Safe = 3
}
```

Thresholds:

| Grade | Oxygen partial pressure |
|---|---:|
| Safe | ≥ 16 kPa |
| Low | 12–16 kPa |
| Critical | 5–12 kPa |
| None | < 5 kPa |

### Air temperature

```csharp
public enum AirTemperatureGrade
{
    Frigid = 0,
    Cold = 1,
    Comfortable = 2,
    Hot = 3
}
```

The documented thresholds are 273.15 K (0 °C) for the comfortable minimum, 323.15 K (50 °C) for the comfortable maximum, and 263.15 K (-10 °C) for the severe cold threshold.

### Contaminants

```csharp
public enum ContaminantGrade
{
    Clear = 0,
    Headache = 1,
    Impaired = 2,
    Suffocating = 3
}
```

The documented thresholds are 0.5%, 1%, and 7% mole fraction.

## `AirAssessment`

```csharp
public struct AirAssessment
{
    public int Cell;
    public bool IsMixture;
    public float TotalPressurePa;
    public float OxygenPartialPressurePa;
    public float OxygenMoleFraction;
    public float CarbonDioxideMoleFraction;
    public float ToxicMoleFraction;
    public float TemperatureK;
    public float TotalMassKg;
    public OxygenGrade Oxygen;
    public AirTemperatureGrade Temperature;
    public ContaminantGrade CarbonDioxide;
    public ContaminantGrade Toxic;
    public bool IsHarmful { get; }
    public bool IsFullySafe { get; }
}
```

`IsMixture` tells a consumer whether the assessment came from the custom multi-species mixture state or from vanilla's single-element cell representation.

The measured values are kept alongside the grades. A mod can therefore apply its own thresholds without re-querying the gas state.

`IsHarmful` is true for absent usable oxygen, suffocating contamination, or harmful temperature. `IsFullySafe` is stricter: all four axes must be in their safe/comfortable states. Warning-level air can therefore be not harmful while still not being fully safe.

## Partial pressure

### `TryGetPartialPressurePa`

```csharp
public static bool TryGetPartialPressurePa(
    int cell,
    ushort elementIdx,
    out float pressurePa);
```

Reads the partial pressure of one species in pascals. The source implementation uses the same native ideal-gas pressure calculation used for total pressure rather than maintaining a second independent formula.

Returns `false` for invalid/no-gas/non-present cases; zero pressure remains a valid numeric result and therefore is not overloaded to mean “no data.”

## Why partial pressure matters

A composition-aware cell can contain oxygen and an inert gas simultaneously. Oxygen mass alone cannot express the physiological effect of dilution; oxygen partial pressure can.

That is the foundation for a modder-facing breathing system that can distinguish:

- 100% oxygen at low total pressure;
- ordinary air at normal pressure;
- oxygen diluted by nitrogen;
- oxygen diluted by CO₂;
- adequate oxygen with dangerously high contaminant concentration.

## Example: breathing assessment

The intended pattern is to obtain one assessment and consume its fields rather than independently implementing four unrelated sensors:

```csharp
AirAssessment air = GetAirAssessment(cell);

if (air.IsHarmful)
{
    ApplyBreathingHazard(cell, air);
}
else if (!air.IsFullySafe)
{
    ShowBreathingWarning(cell, air);
}
```

For a mod that needs a direct species pressure rather than the framework's grade:

```csharp
if (AtmosphereFacade.TryGetPartialPressurePa(
        cell, oxygenElementIdx, out float oxygenPa))
{
    UpdateOxygenGauge(oxygenPa);
}
```

## Combustion-risk stub

### `CombustionRisk`

```csharp
public struct CombustionRisk
{
    public int Cell;
    public float OxygenMoleFraction;
    public float FuelMoleFraction;
    public ushort DominantFuelIdx;
    public float TemperatureK;
    public bool IsOxygenEnriched;
    public bool IsFlammableMixture;
}
```

The current fuel set is Hydrogen and Methane. The oxygen-enrichment threshold is 23.5% mole fraction, and the coarse fuel threshold is 5% mole fraction.

**Important:** this is explicitly a stub. It measures the ingredients a future combustion model needs; it does not ignite materials, calculate flame propagation, or apply fire damage.

## Design boundary

`AtmosphereFacade` is deliberately a measurement/grading facade. It does not own duplicant health effects, status items, ignition gameplay, or other player-facing consequences.

That separation lets multiple mods consume the same atmospheric measurements without forcing them to share the same gameplay interpretation.

## Related APIs

- [Gas Mixture](GAS-MIXTURE.md)
- [Material Properties](MATERIALS.md)
- [Elements](ELEMENTS.md)
