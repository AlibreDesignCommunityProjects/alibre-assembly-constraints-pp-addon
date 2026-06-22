# Assembly Constraints++ for Alibre Design

A starter/template project for building an Alibre Design add-on. It provides a working add-on scaffold — menu registration, IronPython hosting, packaging, and installer — with a sample command you can replace with your own assembly-constraint tooling.

> **Status:** Placeholder / starter. The "Assembly Constraints++" name describes the intended direction; the bundled command is sample scaffolding (a parametric gear-sketch generator) included to demonstrate a complete, runnable add-on. Replace it with your own implementation.

## What's Included
- A managed (VB.NET) add-on shell that registers an "Assembly Constraints++" menu and command in Alibre Design.
- A hosted IronPython runtime with a sample `scripts/Template.py` command, wired end to end as a reference implementation.
- Packaging and registration assets: `.adc` manifest, registry template, and an Inno Setup installer script.

## Requirements
- Alibre Design 29.0.0.29060 (requires the bundled AlibreScript add-on).
- .NET Framework 4.8.1 (net481), x64.
- IronPython 2.7.10 (bundled with the add-on; no separate install required).

## Getting Started
1. Build the `source/alibre-assembly-constraints-pp-addon.vbproj` project (Release, net481), or use the included Inno Setup template (`source/alibre-assembly-constraints-pp-addon.iss`).
2. Copy the build output (the `.adc` manifest, `alibre-assembly-constraints-pp-addon.dll`, the IronPython / Microsoft.Scripting DLLs, `logo.ico`, and the `scripts` folder) to a folder on your machine.
3. Register the add-on so Alibre can discover it by adding its folder path under `HKEY_LOCAL_MACHINE\SOFTWARE\Alibre Design Add-Ons` (see `source/alibre-assembly-constraints-pp-addon.reg` for the format).
4. Restart Alibre Design. The "Assembly Constraints++" menu and command confirm the scaffold loads.

## Customizing
- Replace the logic in `source/scripts/Template.py` with your own command implementation.
- Update the menu and command names in the add-on shell (`AlibreAddOn`) and the `.adc` manifest to match your tool.

## License
See [LICENSE](../LICENSE.txt).
