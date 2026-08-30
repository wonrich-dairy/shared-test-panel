# Wonrich.QualityPanel

The milk quality test panel, shared by every checkpoint that judges a sample (SCRUM-50).

CLR temperature correction, SNF and TS composition, the alcohol stability cascade and the KQ colour
scale live here once, so the MCC gate and the Quality Lab cannot reach different verdicts from the
same readings. That is the whole reason the library exists: a delivery is paid for on the strength
of these numbers, and two implementations would eventually disagree about what a sample means.

## Why it is its own repository

It was extracted from `mcc-intake-service`, where it was a project reference. Two services need it —
the MCC & Intake Service today, the Quality Lab Service as SCRUM-20 onwards land — and a project
reference cannot cross a repository boundary. The four commits that shaped the library came with it,
so `git log` still explains why the KQ scale has seven shades and why SNF is derived from the
corrected CLR.

## What is in it

| Type | Responsibility |
| --- | --- |
| `MilkComposition` | Corrects a lactometer reading for temperature, then derives SNF and TS |
| `AlcoholCascade` | The 80% → 75% → 68% → clot-on-boiling state machine, halting at the first negative |
| `KqColour` / `KqColourScale` | The seven-shade dye scale, ordered best to worst |
| `QualityThresholds` | Limits bound from configuration, so a centre can be retuned without a release |
| `QualityPanelEvaluator` | Runs the lot and reports every measure outside its limit |

Numeric enum values are part of the contract. They are stored against panels and compared across
checkpoints, so they must not be renumbered; new shades and stages go on the end.

## Consuming it

The package is published to GitHub Packages by the workflow in `.github/workflows/publish.yml` when
a change lands on `main` or `develop`.

Add the feed once per repository, in a `nuget.config` beside the solution:

```xml
<configuration>
  <packageSources>
    <add key="wonrich" value="https://nuget.pkg.github.com/wonrich-dairy/index.json" />
  </packageSources>
</configuration>
```

GitHub Packages requires authentication even for public packages, so CI supplies `GITHUB_TOKEN` and
a developer uses a personal access token with `read:packages`. Then:

```xml
<PackageReference Include="Wonrich.QualityPanel" Version="1.0.0" />
```

```csharp
builder.Services.AddQualityPanel(builder.Configuration);
```

Thresholds are read from the `QualityThresholds` configuration section.

## Versioning

The version in `src/Wonrich.QualityPanel/Wonrich.QualityPanel.csproj` is the published version, and
`--skip-duplicate` means republishing the same one is a no-op rather than a failure. **Bump it in the
same PR that changes behaviour**, or consumers pin a version whose meaning has quietly moved.

A change to a formula or a threshold's meaning is a breaking change for anyone holding stored panels,
even when the API compiles unchanged. Those deserve a major bump and a note on the PR.

## Development

```powershell
dotnet test Wonrich.QualityPanel.slnx
```

50 tests cover the composition formulae against worked examples, every branch of the cascade
including replay, the KQ scale ordering, and the evaluator against configured thresholds.

> The frontend carries a TypeScript port of the composition formulae, for panels entered offline
> where the service cannot be reached (SCRUM-10). It is pinned to the same worked examples as
> `MilkCompositionTests`, so a change here fails that build too. If you change a formula, change it
> there as well — or better, take the opportunity to remove the port.

## Branching strategy
- `main`: protected, production-ready
- `develop`: protected integration branch
- `feature/SCRUM-<key>-<description>`: work branches, merged into `develop` via reviewed PR
