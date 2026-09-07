# Material Properties API

## `MaterialProperties`

`MaterialProperties` is the framework's per-substance physical-property record for quantities vanilla ONI does not expose in its element table but the extended simulation needs.

### Verified property set

```csharp
public readonly float MolecularMassGPerMol;
public readonly float LatentHeatOfVaporizationJPerMol;
public readonly double EvaporationCoefficientA;
public readonly double EvaporationCoefficientB;
public readonly float MinLiquidPressurePa;
public readonly float CriticalPointPressurePa;
public readonly float FreezingTemperatureOverrideK;
public readonly float HeatCapacityRatio;
public readonly float LiquidMolarVolumeLPerMol;
public readonly float FusionToVaporizationDenominator;
public readonly float WorkingGasThermalEfficiency;
```

These are intentionally centralized. Phase-change and pipe code should not carry its own copies of latent heats, vapor curves, molecular masses, or critical-pressure constants.

### Molecular mass

`MolecularMassGPerMol` is the molecular mass used for mole calculations. This is important because ONI's own element data can represent diatomic gases with atomic rather than molecular mass. The framework therefore provides a single explicit conversion basis for mole-side calculations.

### Latent heat

`LatentHeatOfVaporizationJPerMol` is the energy associated with vaporization. Fusion latent heat is derived using the configured fusion-to-vaporization denominator rather than requiring every material to duplicate a second constant.

### Vapor pressure

The evaporation curve is represented as:

```text
P(kPa) = A × T^B
```

The registry can invert that relationship to obtain an evaporation/boiling temperature for a specified pressure.

### Minimum liquid pressure

`MinLiquidPressurePa` describes the minimum pressure at which a liquid phase exists in the model. This is particularly important for CO₂ and the planned Mars environment, where pressure determines whether liquid CO₂ is physically possible.

### Critical point

`CriticalPointPressurePa` is the pressure used to evaluate the model's critical-temperature relationship.

### Heat-capacity ratio

`HeatCapacityRatio` (`gamma = Cp/Cv`) is used by adiabatic compression calculations.

### Liquid molar volume

`LiquidMolarVolumeLPerMol` supports conversion between liquid amount and physical volume.

### Working-gas efficiency

`WorkingGasThermalEfficiency` describes how effectively the substance functions as a Stirling-cycle working gas. It is dimensionless, 0–1.

## Construction

```csharp
public MaterialProperties(
    float molecularMassGPerMol,
    float latentHeatOfVaporizationJPerMol,
    double evaporationCoefficientA,
    double evaporationCoefficientB,
    float minLiquidPressurePa,
    float criticalPointPressurePa,
    float heatCapacityRatio,
    float liquidMolarVolumeLPerMol,
    float freezingTemperatureOverrideK = float.NaN,
    float fusionToVaporizationDenominator = 5f,
    float workingGasThermalEfficiency =
        MaterialPropertyRegistry.DefaultWorkingGasThermalEfficiency);
```

## `MaterialPropertyRegistry`

The registry is the lookup layer used by the rest of the framework. The source defines dedicated lookups for properties rather than requiring callers to understand the storage representation.

The verified registry surface includes lookups for molecular mass, latent heats, evaporation temperature/pressure behavior, freezing temperature, liquid density, and working-gas thermal efficiency.

### Example: use the framework's physical density

```csharp
if (MaterialPropertyRegistry.TryGetLiquidDensityKgPerM3(
        elementId, out float densityKgPerM3))
{
    float volumeM3 = massKg / densityKgPerM3;
    UpdateLiquidVolume(volumeM3);
}
```

The exact overloads should be taken from the matching released framework version; the important contract is that physical constants come from the central registry rather than being duplicated by each consumer.

## Design provenance

The current seeded values were taken from the project's decompiled Stationeers reference and checked against the corresponding source structures. The framework does not claim these game-tuned constants are universal real-world thermodynamic data.

The registry is intentionally managed-only today. The native phase-change calculation receives the material constants it needs as arguments. That keeps the native ABI smaller until there is an actual native consumer for a persistent per-element material table.

## Modder guidance

Use the registry whenever a mod needs:

- vapor pressure;
- boiling/condensation temperature at a pressure;
- freezing behavior;
- latent heat;
- liquid volume/density;
- mole conversion;
- adiabatic compression parameters;
- Stirling working-gas behavior.

Do not create another per-mod table for the same physical properties; doing so would allow two machines to disagree about the same substance.
