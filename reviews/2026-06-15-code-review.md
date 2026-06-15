# Code Review — alibre-assembly-constraints-pp-addon

- **Date:** 2026-06-15
- **Branch:** `review/2026-06-15-code-review` (branched from `the-tool-store` @ `aa5eac1` — "cleanup push")
- **Reviewer:** Claude (Opus 4.8)
- **Scope:** Full repository review (VB.NET add-on host + IronPython modeling/UI scripts + installer/registry/manifest tooling)

---

## 1. Summary

This is an Alibre Design add-on. The VB.NET layer (`source/AlibreAddOn.vb`) is the COM-facing
host: it registers a single ribbon menu (`RootMenuItem` 901, one command 1001) and dynamically
runs an IronPython script via an embedded engine. The Python layer (`source/scripts/`) is the
actual tool.

The single most important finding is an **identity crisis**: every name-facing string in the host
says "Assembly Constraints++", the repo is named `alibre-assembly-constraints-pp-addon`, but the
script that actually runs (`source/scripts/Template.py`) is an **involute spur-gear generator**
("Alibre Gear Generator Enhanced") with no assembly-constraint code anywhere. Either the wrong
script was committed or the whole add-on is mislabeled. This is a correctness/shipping blocker
because the menu item does not do what it claims.

Beyond that, the packaging story is broken: the `.iss` installer is an **unfilled template** full
of `{{PLACEHOLDER}}` tokens (won't compile), the `.reg` file points at a **personal hard-coded
`T:\local\...\src\bin\Debug\net481`** path that doesn't even match this repo's `source\` layout,
and the `.vbproj` pins three references to one exact Alibre version directory. The host code
itself is reasonably defensive (session lookup, try/catch around script execution) and notably
resolves the Alibre install path at runtime — better than the sibling repo.

**Overall:** The host plumbing is in decent shape, but the add-on ships the wrong tool under the
wrong name, and none of the installer/registry artifacts are usable as committed. Address the
Critical items before this can be considered shippable.

### Findings by severity

| Severity | Count |
|----------|-------|
| Critical | 2 |
| High     | 3 |
| Medium   | 4 |
| Low / Nit| 6 |

---

## 2. Critical

### C-1. The add-on is named/labeled "Assembly Constraints" but runs a gear generator
**Files:** [source/AlibreAddOn.vb:130](source/AlibreAddOn.vb), [source/AlibreAddOn.vb:142](source/AlibreAddOn.vb), [source/scripts/Template.py:641](source/scripts/Template.py), [source/alibre-assembly-constraints-pp-addon.adc:6](source/alibre-assembly-constraints-pp-addon.adc)

The host advertises:

```vb
Case CMD : Return "Assembly Constraints++ Tool"   ' MenuItemText / MenuItemToolTip
```

…and the manifest `friendlyName`/`Menu`/`Description` all say
`alibre-assembly-constraints-pp-addon`. But the only command (1001) runs `Template.py`
([line 74](source/AlibreAddOn.vb)), which is an involute spur/internal **gear generator**:

```python
form.Text = 'Alibre Gear Generator Enhanced'   # Template.py:641
add_control_row(create_professional_label("py-gear", True), ...)  # Template.py:683
```

There is no assembly-constraint logic anywhere in the repo (no `Constraint`, no mate/align code —
the script only creates sketches and gear parameters on a part). Whatever the intended product is,
the menu lies about what the button does. Either commit the real assembly-constraints script or
rename the project/menu strings to match the gear tool. Until then the deliverable is wrong.

### C-2. Installer `.iss` is an unfilled template and cannot compile
**File:** [source/alibre-assembly-constraints-pp-addon.iss:1-85](source/alibre-assembly-constraints-pp-addon.iss)

The Inno Setup script is the raw boilerplate template — every value is a literal placeholder:

```ini
AppName={{APP_NAME}}
AppVersion={{APP_VERSION}}
DefaultDirName={autopf}\{{INSTALL_FOLDER_NAME}}
LicenseFile={{LICENSE_FILE}}
...
Source: "{{BUILD_DIR}}\{{APP_BASENAME}}.dll"; DestDir: "{app}"; Flags: ignoreversion
```

`ISCC.exe` will fail immediately — `{{...}}` is not valid Inno syntax for `AppVersion`,
`MinVersion`, `ArchitecturesAllowed`, etc. There is no build/CI step that fills these in
(see I-note below: no `.github/workflows`). Replace all `{{PLACEHOLDERS}}` with real values, set
`BUILD_DIR` to the actual output (`bin\Release\net481`), and add `Flags: skipifsourcedoesntexist`
to the `.pdb` line so a Release build doesn't break the compile.

---

## 3. High

### H-1. `.reg` hard-codes a personal `T:\` drive path that doesn't match the repo layout
**File:** [source/alibre-assembly-constraints-pp-addon.reg:4](source/alibre-assembly-constraints-pp-addon.reg)

```reg
[HKEY_LOCAL_MACHINE\SOFTWARE\Alibre Design Add-Ons]
"alibre-assembly-constraints-pp-addon"="T:\\local\\alibre-assembly-constraints-pp-addon\\src\\bin\\Debug\\net481"
```

Three problems: (1) it points at a developer-specific `T:\local\...` drive that won't exist on any
user's machine; (2) it references `src\bin\...` but this repo's project lives under `source\`
(there is no `src\`); (3) it registers a **Debug** output path. Alibre reads this key to discover
the add-on, so as shipped it registers a non-existent folder and the add-on never loads. The
installer's `[Registry]` section should write `{app}` instead, e.g.
`ValueData: "{app}"` — and this static `.reg` should be deleted or regenerated by the installer.

### H-2. Version-pinned, hard-coded Alibre reference paths in the project file
**File:** [source/alibre-assembly-constraints-pp-addon.vbproj:36,40,44](source/alibre-assembly-constraints-pp-addon.vbproj)

```xml
<HintPath>C:\Program Files\Alibre Design 28.1.1.28227\Program\AlibreAddOn.dll</HintPath>
<HintPath>C:\Program Files\Alibre Design 28.1.1.28227\Program\Addons\AlibreScript\AlibreScriptAddOn.dll</HintPath>
<HintPath>C:\Program Files\Alibre Design 28.1.1.28227\Program\AlibreX.dll</HintPath>
```

The build only succeeds on a machine with that exact Alibre version at the default install path.
Anyone with a different version or install location gets `MSB3245` reference-not-found errors.
The runtime code already resolves the install root dynamically
([AlibreAddOn.vb:87](source/AlibreAddOn.vb)) — apply the same idea at build time via an MSBuild
property/env var (e.g. `$(AlibreInstallDir)`) with the pinned path only as a fallback.

### H-3. `Template.py` ignores the host-provided session and re-grabs `TopmostSession`
**Files:** [source/scripts/alibre_setup.py:9-12](source/scripts/alibre_setup.py), [source/scripts/Template.py:39-45](source/scripts/Template.py)

`alibre_setup.py` runs first and carefully builds `CurrentPart`/`CurrentAssembly` from the
injected `CurrentSession` that the host resolved from the menu invocation:

```python
if CurrentSession and isinstance(CurrentSession, AlibreX.IADPartSession):
    CurrentPart = Part(CurrentSession)
```

But `Template.py` throws all of that away and re-acquires its own root and part:

```python
alibre = Marshal.GetActiveObject("AlibreX.AutomationHook")
root = alibre.Root
...
MyPart = Part(root.TopmostSession)   # not necessarily the invoked session
```

`TopmostSession` is whatever window is frontmost, which may not be the session the menu command
was fired on, so a gear can land in the wrong document. The script also never uses `CurrentPart`
(prepared for it) and only checks `MyPart`. Use the `CurrentPart` that `alibre_setup.py` already
established, and skip the redundant `GetActiveObject` round-trip. (Minor consequence: if no part
is open, `alibre_setup.py` already shows an error, then `Template.py` shows a second one.)

---

## 4. Medium

### M-1. `ExecuteScript` is passed a full path but re-combines it as a filename
**Files:** [source/AlibreAddOn.vb:74](source/AlibreAddOn.vb), [source/AlibreAddOn.vb:99,105](source/AlibreAddOn.vb)

`InvokeCommand` passes an already-absolute path as the `mainScriptFileName` argument:

```vb
runner.ExecuteScript(session, Path.Combine(scriptsPath, "Template.py"))
```

`ExecuteScript` then re-combines it with the scripts dir and also stores it as `ScriptFileName`:

```vb
Dim mainScriptPath As String = Path.Combine(ScriptsPath, mainScriptFileName) ' line 99
scope.SetVariable("ScriptFileName", mainScriptFileName)                      ' line 105
```

It happens to *work* because `Path.Combine(dir, absolutePath)` returns the absolute second
argument, but it's confusing and fragile, and `ScriptFileName` ends up being a full path rather
than a filename (any script relying on it being a bare name would break). Pass just `"Template.py"`
from `InvokeCommand` and let `ExecuteScript` own the combine, or rename the parameter to reflect
that it's a full path.

### M-2. `InvokeCommand` silently falls back to the *first* session when the id doesn't match
**File:** [source/AlibreAddOn.vb:60-65](source/AlibreAddOn.vb)

```vb
If session Is Nothing Then
    For Each s As IADSession In _AlibreRoot.Sessions
        session = s
        Exit For
    Next
End If
```

If `sessionIdentifier` doesn't match any open session, the code grabs whatever session enumerates
first and runs the tool there. Combined with H-3 (`TopmostSession`), this further increases the
chance of modeling into the wrong document. If the invoked session can't be found, it's safer to
show a clear error and abort than to guess. Also note `session.Identifier` is dereferenced at
[line 107](source/AlibreAddOn.vb) with no null guard — if `Sessions` is empty, `session` stays
`Nothing` and this throws an NRE that surfaces only as a generic "Python Execution Error" box.

### M-3. Gear math edge cases can divide-by-zero / produce degenerate profiles
**File:** [source/scripts/Template.py:121,255,282,288,297](source/scripts/Template.py)

Several loops compute `t = float(i) / (num_points[x] - 1)`. The `num_points` defaults are ≥ 5, so
this is safe with defaults, but `generate_internal_tooth_profile` computes
`half_num_points_lower = num_points[2] // 2` ([line 280](source/scripts/Template.py)) and then
divides by `half_num_points_lower - 1` ([line 282](source/scripts/Template.py)). With the default
`num_points[2] = 5`, `half = 2`, so `2 - 1 = 1` is fine — but any caller passing a small/odd
`num_points` (e.g. `[*, *, 2, *]` → `half = 1` → divide by zero) crashes. These helpers are
public-looking API with `num_points` parameters, so guard `n > 1` (and `half > 1`) before the
loops, or clamp the inputs. Low real-world risk given the fixed UI, but it's an unguarded landmine.

### M-4. Dead / duplicated gear-creation code path
**File:** [source/scripts/Template.py:437-471](source/scripts/Template.py) vs [source/scripts/Template.py:816-850](source/scripts/Template.py)

`create_gear_with_plane` (module level) and the nested `create_gear_with_unique_name` inside
`create_gear_click` are near-identical: same parameter add-and-regenerate block copy-pasted. The
UI path only ever calls the nested copy (with `original_create = create_gear_with_plane` assigned
on [line 816](source/scripts/Template.py) and then never used). Delete the dead
`create_gear_with_plane` (and the unused `original_create` binding) or factor the shared
parameter-writing logic into one helper that both can call.

---

## 5. Low / Nits

### L-1. `MenuIcon` returns `"logo.ico"` for the command but the manifest icon is at the app root
[source/AlibreAddOn.vb:148](source/AlibreAddOn.vb) returns the bare name `"logo.ico"`. The
`.vbproj` copies `logo.ico` to the output root ([vbproj:19-21](source/alibre-assembly-constraints-pp-addon.vbproj)),
and the `.adc` also points `<Icon location="logo.ico"/>`. This is probably fine, but confirm
Alibre resolves the menu-icon name relative to the add-on directory (not the `scripts\` subdir) so
the command actually gets an icon; the root menu (901) deliberately returns `""`.

### L-2. Module/class named `AlibreAddOn` collides with the imported namespace
[source/AlibreAddOn.vb:3,9](source/AlibreAddOn.vb): `Imports AlibreAddOn` and
`Public Module AlibreAddOn` share a name. VB resolves it here, but it's the same self-inflicted
ambiguity the sibling repo had in C#. Rename the module (e.g. `AddOnEntryPoint`) for clarity.

### L-3. `printTraceBack()` is a no-op
[source/scripts/alibre_setup.py:9-11](source/scripts/alibre_setup.py): it imports `traceback` and
immediately `return`s without formatting or showing anything. Every `include_trace=True` call site
therefore does nothing useful. Either implement it (`traceback.format_exc()` into the message) or
drop the parameter.

### L-4. `safe_try` wrapper loses event-handler identity / `time` import unused
[source/scripts/Template.py:26-34,8](source/scripts/Template.py): `safe_try` is fine, but
`import time` (line 8) is never used. Minor dead import. (`scale_size` at
[line 481](source/scripts/Template.py) also ignores its `scale_factor` argument and just returns
`int(base_size)` — dead parameter.)

### L-5. Empty `submodules/` and `documentation/` (only `.gitkeep`); declared submodule not vendored
[.gitmodules:1-3](.gitmodules) declares `submodules/alibre-script-adk`, but the working tree only
contains `submodules/.gitkeep` — the submodule is not initialized/committed as a gitlink. If the
build or scripts depend on it, a fresh clone won't have it. If it's not needed, remove the
`.gitmodules` entry. The trailing blank lines in `.gitmodules` are also cruft.

### L-6. `My Project/launchSettings.json` pins the same exact Alibre version
[source/My Project/launchSettings.json:8](source/My%20Project/launchSettings.json) sets the debug
executable to `C:\Program Files\Alibre Design 28.1.1.28227\Program\Alibre Design.exe`. Same
portability issue as H-2, but debug-only so low impact. `Application.myapp` references a `Form1`
that doesn't exist ([Application.myapp:4](source/My%20Project/Application.myapp)) — harmless
WinForms scaffolding leftover since this is a class library, but worth pruning.

---

## 6. What looks good

- **Runtime install-path resolution.** [AlibreAddOn.vb:87](source/AlibreAddOn.vb) derives the
  Alibre root from the loaded `AlibreX.dll` location instead of hard-coding it — strictly better
  than the sibling repo's pinned runtime path.
- **Defensive script execution.** `ExecuteScript` checks both script files exist before running,
  injects session context into the scope, and wraps execution in try/catch with a clear error box.
- **`.csproj`/`.vbproj` content copying is correct.** Both `alibre_setup.py` and `Template.py`
  (and `logo.ico`, `.adc`) have `CopyToOutputDirectory` entries
  ([vbproj:18-28,49-51](source/alibre-assembly-constraints-pp-addon.vbproj)) — the missing-setup-script
  bug from the sibling repo is not present here.
- **No committed build artifacts.** `git ls-files` shows no `.dll`/`.exe`/`.pdb`/`bin`/`obj`; the
  `.gitignore` is the standard comprehensive VS template.
- **The gear math itself is substantial and well-commented** (involute/trochoid generation with
  docstrings), and the WinForms UI handler code is carefully wrapped to avoid crashing the host.
- **Lifecycle hygiene:** `AddOnUnload` nulls `_addOnHandle` and `AlibreRoot`.

---

## 7. Recommended fix order

1. **C-1** — resolve the identity mismatch: ship the real assembly-constraints script or rename
   the project/menu to "Gear Generator". Nothing else matters until the button does the right
   thing. (blocking)
2. **C-2 + H-1** — make the installer compile (fill the `.iss`) and write the registry key to
   `{app}` instead of a personal `T:\` Debug path, so the add-on actually loads after install.
   (blocking for distribution)
3. **H-2** — un-pin the build-time Alibre reference paths.
4. **H-3 / M-2** — use the host-provided `CurrentSession`/`CurrentPart`; fail loudly instead of
   guessing the session, and null-guard `session.Identifier`.
5. **M-1 / M-3 / M-4** — tidy the path-passing contract, guard the gear-math divisors, and delete
   the duplicated gear-creation function.
6. Sweep the **L-*** nits (dead imports, no-op `printTraceBack`, empty submodule, debug-only path
   pins) when touching the respective files.
