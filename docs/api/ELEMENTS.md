# Element & Cell API

## `ElementRegistry`

`ElementRegistry` provides a single managed registration seam for custom ONI elements. It avoids every mod implementing its own patch against the element-loading pipeline and makes element registration order-independent at the framework boundary.

## Registration timing

Registration must happen before `ElementLoader.Load` builds the game's element table. In practice this means a consuming mod should register from `OnLoad`/`UserMod2.OnLoad` or another initialization path reached before element loading.

Late registration returns `false` and logs a warning rather than pretending the element exists.

## `Register`

```csharp
public static bool Register(ElementEntry entry);
```

The generic form accepts a fully specified `ElementEntry`.

Returns `false` for an invalid entry, duplicate element id, or registration after the element table has already been built.

## `RegisterGas`

```csharp
public static bool RegisterGas(
    string elementId,
    string localizationID,
    float specificHeatCapacity,
    float thermalConductivity,
    float molarMass,
    string condensesTo,
    float condensationPointK,
    float toxicity = 0f,
    float defaultTemperatureK = 300f,
    string materialCategory = "Unbreathable",
    float lightAbsorptionFactor = 0.1f,
    float radiationAbsorptionFactor = 0.08f);
```

The helper fills phase-appropriate defaults for a gas and establishes its low-temperature transition target.

## `RegisterLiquid`

```csharp
public static bool RegisterLiquid(
    string elementId,
    string localizationID,
    float specificHeatCapacity,
    float thermalConductivity,
    float molarMass,
    string freezesTo,
    float freezingPointK,
    string boilsTo,
    float boilingPointK,
    float toxicity = 0f,
    float defaultTemperatureK = 300f,
    string materialCategory = "Unbreathable",
    float lightAbsorptionFactor = 0.7f,
    float radiationAbsorptionFactor = 0.8f);
```

This helper supplies the standard liquid-flow and mass defaults while defining both freezing and boiling transitions.

## Localization

### `AddStrings`

```csharp
public static void AddStrings(
    string elementId,
    string name,
    string description);
```

Creates the element's `.NAME` and `.DESC` string keys using the framework's `STRINGS.ELEMENTS.<ID>` convention.

Call this before element loading attempts to resolve the element's localization key.

## Registration state

```csharp
public static bool ElementTableBuilt { get; }
public static bool Suppressed { get; set; }
public static IEnumerable<string> Registered { get; }
```

`Suppressed` exists for simulation-corpus recordings that must reproduce the shipped Klei element table exactly. It is not a normal modding switch.

## Example: register a custom gas

```csharp
public override void OnLoad(Harmony harmony)
{
    base.OnLoad(harmony);

    ElementRegistry.AddStrings(
        "ExampleNitrogen",
        "Example Nitrogen",
        "A nitrogen-like test gas.");

    ElementRegistry.RegisterGas(
        "ExampleNitrogen",
        "STRINGS.ELEMENTS.EXAMPLENITROGEN",
        specificHeatCapacity: 1.04f,
        thermalConductivity: 0.025f,
        molarMass: 28.014f,
        condensesTo: "ExampleLiquidNitrogen",
        condensationPointK: 77.36f);
}
```

The exact physical constants in a real mod should come from the material definition appropriate to that substance; the example is intentionally a small integration demonstration, not a recommendation for fictional-material balancing.

## Why the registry matters

The native simulation receives the complete element table and uses element indices internally. A centralized registration seam prevents multiple mods from independently modifying the same loading method and accidentally creating order-dependent element tables.

The element id is hashed by the game's loader; it does not require a new `SimHashes` enum member.

The framework also relies on the game's existing substance manifestation path. A gas or liquid can therefore be registered without the framework inventing a parallel art/substance system.

## `ElementChunkFacade`

`ElementChunkFacade` is the managed home for interaction with vanilla `ElementChunk` / `SimTemperatureTransfer` behavior. It exists so a mod that handles off-grid matter can use the game's established thermal interaction rather than reimplementing it in gameplay code.

Its primary use cases are:

- chunks of matter outside the grid;
- thermal interaction with surrounding cells;
- custom machinery that needs to inspect or move chunk matter;
- simulation/debug tooling.

## `ThermalMassBonus`

For additional per-cell thermal capacity, see [Thermal & Energy](THERMAL-ENERGY.md). It is intentionally separate from element registration because the bonus represents an environmental/cell property, not a material definition.

## Related APIs

- [Material Properties](MATERIALS.md)
- [Gas Mixture](GAS-MIXTURE.md)
- [Simulation Extensions](SIM-EXTENSIONS.md)
