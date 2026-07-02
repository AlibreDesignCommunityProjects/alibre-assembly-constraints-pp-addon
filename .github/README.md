# Assembly Constraints++ for Alibre Design

A starter/template add-on for Alibre Design that ships a complete, runnable scaffold you replace with your own tooling.

This is a placeholder. The name "Assembly Constraints++" describes where the project is headed. No assembly-constraint logic exists yet. You get a working add-on shell that registers a ribbon menu, hosts IronPython, and runs a bundled sample command. That sample is a Windows Forms gear-sketch generator, included to show the pipeline running end to end. Swap it for your own implementation.

The add-on shell is VB.NET on .NET Framework 4.8.1 (net481), x64. It targets Alibre Design 29.0.0.29060 and hosts IronPython 2.7.10 to run scripts against the Alibre Script API.

## Table Of Contents

- [What Is Here](#what-is-here)
- [Official Alibre Resources](#official-alibre-resources)
- [Requirements](#requirements)
- [Quick Start](#quick-start)
- [Installation](#installation)
- [Usage](#usage)
- [Key Files](#key-files)
- [Key Folders](#key-folders)
- [Notes](#notes)
- [License](#license)

## What Is Here

- A managed VB.NET add-on shell (`AlibreAddOn.vb`) that registers an "Assembly Constraints++" ribbon menu and one command.
- A hosted IronPython runtime that runs `scripts/Template.py` when the command fires.
- A sample command: `Template.py` opens a Windows Forms dialog that generates external or internal involute spur-gear sketches on a plane you pick in the workspace.
- A script bootstrap (`alibre_setup.py`) that binds `CurrentPart` and `CurrentAssembly` from the active session.
- Packaging and registration assets: the `.adc` manifest, a registry template, and an Inno Setup installer template.

## Official Alibre Resources

Alibre's official resources for API development and AI/LLM/agent workflows: <https://www.alibre.com/api/>

## Requirements

- Alibre Design 29.0.0.29060. The build references `AlibreX.dll`, `AlibreAddOn.dll`, and the bundled AlibreScript add-on (`AlibreScriptAddOn.dll`).
- .NET Framework 4.8.1 (net481), x64.
- IronPython 2.7.10, pulled in as a NuGet package reference and copied into the build output. No separate install.
- Visual Studio or the .NET SDK with MSBuild to build the `source/` project. Inno Setup if you use the installer template.

## Quick Start

1. Open `source/alibre-assembly-constraints-pp-addon-solution.sln` (or `.slnx`) and build Release, net481.
2. Register the build output folder with Alibre using `source/alibre-assembly-constraints-pp-addon.reg` as the format reference.
3. Restart Alibre Design. The "Assembly Constraints++" ribbon menu confirms the add-on loaded.

## Installation

1. Build `source/alibre-assembly-constraints-pp-addon.vbproj` (Release, net481). The output folder holds `alibre-assembly-constraints-pp-addon.dll`, the `.adc` manifest, the IronPython and Microsoft.Scripting/Dynamic DLLs, `logo.ico`, and the `scripts` folder.
2. Register the add-on so Alibre discovers it. Add a value under `HKEY_LOCAL_MACHINE\SOFTWARE\Alibre Design Add-Ons` whose name is the add-on name and whose data is the build output folder path. See `source/alibre-assembly-constraints-pp-addon.reg` for the exact format.
3. Restart Alibre Design.

For a packaged installer, fill in the `{{PLACEHOLDER}}` fields in `source/alibre-assembly-constraints-pp-addon.iss` and build it with Inno Setup.

## Usage

1. Open a part in Alibre Design.
2. Open the "Assembly Constraints++" ribbon menu and run "Assembly Constraints++ Tool". The add-on runs `scripts/Template.py`.
3. In the sample gear dialog, set teeth, module, pressure angle, profile shift, and thickness, choose external or internal, click the plane list and select a plane in the workspace, then click Create Gear. The script adds the gear sketch and writes pitch-radius and tooth-count parameters onto the part.

To build your own tool, replace the body of `scripts/Template.py` and update the menu and command text in `AlibreAddOn.vb` and the `.adc` manifest.

## Key Files

| File | Purpose |
| --- | --- |
| `source/AlibreAddOn.vb` | Add-on shell: registers the ribbon menu and command, hosts an IronPython engine, and runs the bundled script. |
| `source/alibre-assembly-constraints-pp-addon.vbproj` | .NET SDK project targeting net481 and x64; references AlibreX/AlibreAddOn/AlibreScript and IronPython 2.7.10. |
| `source/alibre-assembly-constraints-pp-addon-solution.sln` | Visual Studio solution (`.slnx` variant also present). |
| `source/alibre-assembly-constraints-pp-addon.adc` | Alibre add-on manifest: managed DLL, load at startup, icon, and GUID identifier. |
| `source/alibre-assembly-constraints-pp-addon.reg` | Registry template that points Alibre at the add-on's build output folder. |
| `source/alibre-assembly-constraints-pp-addon.iss` | Inno Setup installer template with placeholder fields. |
| `source/scripts/Template.py` | Sample command: Windows Forms gear-sketch generator, wired end to end as the reference implementation. |
| `source/scripts/alibre_setup.py` | Script bootstrap that binds `CurrentPart` and `CurrentAssembly` from the active session. |
| `source/logo.ico` | Add-on and command icon. |
| `LICENSE.txt` | MIT license. |

## Key Folders

| Folder | Purpose |
| --- | --- |
| `source/` | VB.NET project, hosted scripts, add-on manifest, registry template, and installer template. |
| `source/scripts/` | IronPython scripts the add-on runs (`alibre_setup.py`, `Template.py`). |
| `documentation/` | Placeholder for documentation (currently empty). |
| `reviews/` | Dated code-review notes. |
| `submodules/` | Git submodule mount point; references `alibre-script-adk`. |
| `.github/` | This README plus issue, pull-request, contributing, and conduct templates. |

## Notes

- This is scaffolding. The gear generator only demonstrates the pipeline. It does no assembly-constraint work.
- Scripts run under IronPython 2.7.10, so keep them Python 2.7 compatible.
- The runtime pulls `AlibreScript.API` and the Alibre `PythonLib` search paths from the installed Alibre Design 29.0.0.29060 folder. A different install path or version can break resolution.
- The `.reg` value data and the `.iss` fields are examples; set them to match your machine and build.

## License

See [LICENSE](../LICENSE.txt).
