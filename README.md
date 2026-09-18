# IntuneWin ContextPrep

Adds a **Package as .intunewin** entry to the Windows Explorer context menu. Right-click a setup
file or a folder and get a finished `.intunewin` package, without opening a console and typing out
`-c`, `-s` and `-o` paths for IntuneWinAppUtil.exe.

The Microsoft [Win32 Content Prep Tool](https://github.com/microsoft/Microsoft-Win32-Content-Prep-Tool)
does the packaging. This project handles everything around it: locating the source folder, picking
the setup file, warning about what is about to be swept into the package, and reading the result
back so you know what to type into the Intune portal afterwards.

## Why package an MSI at all?

Intune accepts a bare `.msi` as a line-of-business app, so wrapping one as a Win32 app looks like
extra work. Microsoft's guidance runs the other way: use the Win32 app type exclusively for Windows
apps, [particularly for multi-file installers](https://learn.microsoft.com/intune/app-management/deployment/win32).

The line-of-business type accepts [a single command-line argument](https://learn.microsoft.com/intune/app-management/deployment/add-lob-windows),
with `/i` and `/x` implied and rejected - so a transform, a property and `/qn` already exceed it. It
has no requirement rules, no dependencies, no supersedence, no custom return codes, fixed detection,
and an 8 GB ceiling against the Win32 type's 30 GB. Mixing the two types during Autopilot enrollment
can also fail, because both compete for the Trusted Installer service.

What the Win32 route costs you is the Program and Requirements pages: install command, uninstall
command, detection rule, architecture. Those are the fields this tool fills in for you.

## What it adds on top of IntuneWinAppUtil.exe

- **No path juggling.** Right-click the setup file and its parent folder becomes the source folder.
  Right-click a folder and it is used directly. If a folder holds several setup files you are asked
  which one, and the answer is remembered per folder, so repackaging after a version bump is one
  click.
- **A confirmation before anything is packaged.** Everything in the source folder ends up inside
  the package. The prompt shows the file count and total size, flags personal and system folders by
  name, calls out any existing `.intunewin` files that would be bundled back in, and warns when the
  source exceeds the 30 GB ceiling Intune enforces on a Win32 app.
- **A path length pre-check.** The packaging tool copies your files through a temp folder first, so
  deeply nested files can fail there even though they open fine where they are. The paths are
  measured up front and you are told to move the source folder closer to the drive root, instead of
  the run failing halfway through.
- **Output kept away from the source.** Packages land in a timestamped folder under
  `%LOCALAPPDATA%\IntuneWinContextPrep\Output`, so a second run never wraps the previous package
  into the next one.
- **Version in the filename.** The MSI product version, or the exe's file version resource, is
  appended: `remotehelpinstaller_5.2.1040.0.intunewin`.
- **Portal handoff.** After packaging, `Detection.xml` is read back out of the package and a `.json`
  file is written next to it with the publisher, a suggested detection rule, and install and
  uninstall commands. For an MSI the detection rule carries both the product code and the product
  version.
- **Architecture for the requirement rule.** The setup file is read to work out whether it targets
  x86, x64 or Arm64 - from the PE header for an exe, from the Template summary property for an MSI -
  and the handoff reports what to select under **Operating system architecture**. A 32-bit installer
  reports `x86, x64`, since it still installs on 64-bit Windows.
- **Real silent switches for exe installers.** Rather than emitting a `<silent switch>` placeholder,
  the wrapper looks inside the exe to work out which installer built it, and reports the switches
  that installer actually documents.

| Detected as | Install command | Uninstall |
| --- | --- | --- |
| WiX Burn bundle | `/quiet /norestart` | `/uninstall /quiet /norestart` |
| NSIS | `/S` (case sensitive) | `Uninstall.exe /S`, path from `QuietUninstallString` |
| Inno Setup | `/VERYSILENT /SUPPRESSMSGBOXES /NORESTART` | `unins000.exe` with the same switches |
| InstallShield | `/s /v"/qn"` | From `UninstallString` |
| MSI | `msiexec /i "<file>" /qn` | `msiexec /x <ProductCode> /qn` |
| Unknown | File name only | Check after a test install |

## Requirements

- Windows PowerShell 5.1
- .NET Framework 4.7.2 or later, which IntuneWinAppUtil.exe depends on. The installer checks this
  and stops if it is missing.
- Local administrator rights for a per-machine install. Not needed for a per-user install.

## Install

```powershell
.\Install-IntuneWinContextPrep.ps1
```

Elevated, this installs per machine to `%ProgramFiles%\IntuneWinContextPrep` and registers the menu
entry in `HKLM`. Without elevation it installs per user to `%LOCALAPPDATA%\IntuneWinContextPrep` and
`HKCU`. Force either with `-Scope Machine` or `-Scope User`.

Per-machine is the better default: the files Explorer executes are then writable only by
administrators, and the menu entry is available to everyone on the box. A per-user install leaves a
script that Explorer runs automatically in a location the signed-in user can modify.

The installer downloads IntuneWinAppUtil.exe, verifies it, copies it and both scripts into the
install folder, and adds the menu entry for `.exe`, `.msi`, `.msp`, folders, and the background of
an open folder.

### Where IntuneWinAppUtil.exe comes from

Microsoft's repository offers no proper downloads page, only the file sitting in the source tree,
and they have swapped that file for a different build without changing the version number. So the
installer downloads a specific tagged version, `v1.8.7` by default, and refuses to install it unless
Windows confirms it is a program signed by Microsoft. Use `-ToolVersionTag` to pick a different one.

Tags can still be moved, so if you want one exact build and nothing else, pass its SHA256:

```powershell
.\Install-IntuneWinContextPrep.ps1 -ExpectedHash 'C1BA45B5CB939E84AF064BB7FF4B38FB3DFE33C8DC1078FD9B157672EAE671F6'
```

If downloading is blocked in your environment, point the installer at a copy you have already
approved. It gets the same checks:

```powershell
.\Install-IntuneWinContextPrep.ps1 -ToolPath 'C:\Approved\IntuneWinAppUtil.exe' -ExpectedHash '<sha256>'
```

## Use it

Right-click any of the following and choose **Package as .intunewin**. On Windows 11 it sits under
**Show more options**.

- an `.exe`, `.msi` or `.msp` file
- a folder containing one
- the empty background of an open folder

Confirm the prompt. Explorer opens the output folder when the run finishes, and a dialog reports the
package path, content and package size, MSI product code, version and execution context where
available, and the suggested portal values.

## Output

Everything is written under `%LOCALAPPDATA%\IntuneWinContextPrep`, under both install scopes, because
`%ProgramFiles%` is not writable at packaging time.

| Path | Contents |
| --- | --- |
| `Output\<setup>_<timestamp>\` | The `.intunewin` package and its `.json` handoff file |
| `Logs\IntuneWinContextPrep_<timestamp>.log` | Per-run log |
| `Logs\IntuneWinAppUtil_<timestamp>_std*.log` | Raw output from the packaging tool |
| `setup-choices.json` | Remembered setup file per source folder |

## AppLocker and App Control

Under an application control policy that does not trust these scripts, PowerShell runs restricted
and neither script can do its job. Since Explorer runs the wrapper with no visible window, that
would otherwise look like nothing happened, so both scripts check for this first and report the
reason - the wrapper by writing it to its log.

Sign `Invoke-IntuneWinContextPrep.ps1` and allow the signer in your policy. If the installed wrapper
is signed, the installer registers the menu entry with `-ExecutionPolicy AllSigned`, so a tampered
copy refuses to run. Unsigned, it falls back to `-ExecutionPolicy Bypass` and warns you.

## Uninstall

```powershell
.\Install-IntuneWinContextPrep.ps1 -Action Uninstall
```

This clears both `HKCU` and `HKLM`, so a per-user and a per-machine install are removed in one pass.
Clearing `HKLM` needs elevation; without it the per-machine entry is left in place and a warning is
written.

The install folder, the packaging tool, the logs and every package generated so far are left alone
unless you ask for them:

```powershell
.\Install-IntuneWinContextPrep.ps1 -Action Uninstall -RemoveFiles
```

A copy of the installer is placed next to the wrapper during install, so the entry can be removed
later without the original download.

## Parameters

| Parameter | Applies to | Description |
| --- | --- | --- |
| `-Action` | Both | `Install` or `Uninstall`. Defaults to `Install`. |
| `-Scope` | Install | `Machine` or `User`. Defaults to `Machine` when elevated. |
| `-InstallPath` | Both | Target folder. Defaults per scope. On uninstall, only used with `-RemoveFiles`. |
| `-ToolVersionTag` | Install | Git tag to download IntuneWinAppUtil.exe from. Defaults to `v1.8.7`. |
| `-ToolPath` | Install | Use an existing local copy instead of downloading. Verified the same way. |
| `-ExpectedHash` | Install | SHA256 the packaging tool must match. |
| `-RemoveFiles` | Uninstall | Also delete the install folder and everything in it. |

## Known limits

- One item at a time. Select several files and the entry handles only the first one. This is
  deliberate: Explorer otherwise starts a separate hidden run for every file selected, and stops
  showing the entry at all past 15 of them.
- No submenu. Submenus defined in the registry do not appear in current Explorer builds, at top
  level or under **Show more options**, so each file type gets one flat entry.
- Detection rules and commands in the handoff JSON are suggestions read from package metadata and
  installer fingerprints. Check them against vendor documentation before deploying.

## Files

| File | Purpose |
| --- | --- |
| `Install-IntuneWinContextPrep.ps1` | Installs and uninstalls the context menu entry |
| `Invoke-IntuneWinContextPrep.ps1` | Packaging wrapper called by the menu entry. Not meant to be run directly |

## Author

Martin Bengtsson
Blog: [www.imab.dk](https://www.imab.dk)
X: [@mwbengtsson](https://x.com/mwbengtsson)
