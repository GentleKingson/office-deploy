# Office Deploy Script

A Windows batch script that automates the installation and activation of **Microsoft 365 Apps for enterprise**. Built on the [MAS (Microsoft Activation Scripts)](https://github.com/massgravel/Microsoft-Activation-Scripts) v3.10 framework, it combines Office deployment via Microsoft's official CDN with the Ohook activation method.

## Requirements

- **Administrator privileges** — the script must be run as Administrator
- **Windows 7 or later** (tested on Windows 7–11)
- **Internet connection** — required to download `setup.exe` from Microsoft CDN
- **64-bit or 32-bit Windows** — auto-detects architecture and re-launches under native arch if needed

## Quick Start

1. Place `officeDeploy.bat` in any directory
2. Right-click → **Run as administrator**
3. Select `[1]` to install Microsoft 365 Apps for enterprise
4. Wait for installation and activation to complete

## Execution Flow

The script follows a linear, label-driven flow:

```
[Start] → [Arch Check] → [Admin Check] → [Menu] → [Install] → [Activate] → [Exit]
```

### Phase 1: Environment Setup (lines 1–44)

| Step | What It Does |
|---|---|
| **Sysnative path resolution** | Detects 32-bit vs 64-bit CMD and sets correct system paths for registry access |
| **WOW64 re-launch** | If running under 32-bit CMD on 64-bit Windows, re-launches via `Sysnative\cmd.exe` to access 64-bit registry. Similarly handles ARM64 via `SysArm32` |
| **Admin check** | Runs `net session` to verify elevated privileges; exits with instructions if not admin |

### Phase 2: Installation (lines 63–120)

| Step | What It Does |
|---|---|
| **Version selection menu** | Presents a `choice` prompt — currently option `[1]` for Microsoft 365 Apps for enterprise or `[2]` to exit |
| **Download setup.exe** | If `setup.exe` doesn't exist in the script directory, downloads it from `https://officecdn.microsoft.com/pr/wsus/setup.exe` via PowerShell `Invoke-WebRequest` |
| **Generate Configuration.xml** | Writes an Office Deployment Tool XML configuration file with these settings: |
| | - `OfficeClientEdition="64"` — 64-bit only |
| | - `Channel="MonthlyEnterprise"` — Monthly Enterprise update channel |
| | - `Product ID="O365ProPlusRetail"` — Microsoft 365 Apps for enterprise |
| | - `Language ID="zh-cn"` — Simplified Chinese |
| | - Excluded apps: Groove (OneDrive sync), Lync (Skype), OneDrive, OutlookForWindows, Teams |
| | - `Updates Enabled="FALSE"` — disables automatic Office updates |
| | - `RemoveMSI` — removes existing MSI-based Office installations |
| | - **AppSettings** — sets default file formats: Excel → `.xlsx` (Value=51), PowerPoint → `.pptx` (Value=27), Word → default |
| | - `Display Level="Full" AcceptEULA="TRUE"` — shows UI, auto-accepts EULA |
| **Run installer** | Executes `setup.exe /configure Configuration.xml` |
| **Error handling** | Checks `%ERRORLEVEL%`; proceeds to activation on success, jumps to `:fail` on failure |

### Phase 3: Activation (lines 122–274)

Activation uses the **Ohook** method — a DLL hook technique that intercepts Office's license validation calls. This is sourced from MAS v3.10.

#### 3.1 Initialization (`:oh_activate_core`, lines 142–187)

Sets up script path variables, calls a series of diagnostic and setup subroutines:

| Subroutine | Purpose |
|---|---|
| `dk_setvar` | Initializes PowerShell path, Windows build number, color variables, and service names |
| `dk_reflection` | Sets up .NET reflection code for P/Invoke calls |
| `dk_ckeckwmic` | Detects whether `wmic.exe` is available (removed in Windows 11 24H2+) |
| `dk_product` | Gets Windows edition name via `winbrand.dll` |
| `dk_showosinfo` | Displays OS info, build number, and architecture |
| `oh_setspp` | Determines which licensing service to use: `SoftwareLicensingProduct` (Win 8+) or `OfficeSoftwareProtectionProduct` (Win 7 / Office 2010) |
| `oh_getpath` | Scans registry for installed Office versions (C2R and MSI, versions 14/15/16) |

#### 3.2 Version Detection and Processing

The script processes Office in order of version:

1. **Office 16.0 C2R** (`:oh_activate_o16c2r`, lines 190–232) — Office 2016/2019/2021/365 Click-to-Run
2. **Office 15.0 C2R** (`:oh_activate_o15c2r`, lines 235–270) — Office 2013 Click-to-Run
3. **Office MSI** (`:oh_processmsi`, lines 723–777) — Office 2010/2013/2016 MSI installations

For each version, it:
- Queries registry for install path, architecture, version, and product IDs
- Determines hook DLL paths (`sppc32.dll` for x86, `sppc64.dll` for x64)
- Checks for expired preview licenses
- Fixes `ProductReleaseIds` registry entries if corrupted

#### 3.3 License Installation and Hook Setup (`:oh_process`, lines 678–719)

For each detected product ID:

1. Looks up the matching product key and activation ID from the embedded `:ohookdata` table (lines 2047–2598)
2. Instains missing license files via `oh_installlic` (uses `integrator.exe` or PowerShell/WMI fallback)
3. Installs the product key via WMI `InstallProductKey`
4. Installs the Ohook hook via `oh_hookinstall` or `oh_hookinstall_ospp`

#### 3.4 Ohook Installation (`:oh_hookinstall`, lines 474–527)

The core activation mechanism:

1. Removes any previous hook installation
2. Creates a **symlink** from `%OfficeRoot%\vfs\System\sppcs.dll` → system `sppc.dll`
3. Extracts a custom `sppc.dll` (embedded as base64 in the script) to `%OfficeRoot%\vfs\System\sppc.dll`
4. Modifies the DLL's hash to avoid detection
5. For older Office (pre-Win8), hooks `OSPPC.DLL` instead via `:oh_hookinstall_ospp`

#### 3.5 Cleanup (`:oh_clearblock`, `:oh_uninstkey`, `:oh_licrefresh`, lines 781–996)

| Cleanup Step | What It Does |
|---|---|
| **Clear vNext/license blocks** | Removes Office subscription license registry keys, local license cache files, and SharedComputerLicensing entries for all user accounts |
| **Clear device-based licensing** | Removes device licensing registry entries |
| **Clear OEM keys** | Removes OEM activation registry keys |
| **Skip license check registry** | Sets `TimeOfLastHeartbeatFailure` to `2040-01-01` to prevent "license status" banners |
| **Uninstall other/grace keys** | Removes competing product keys (except Project/Visio if installed) |
| **Refresh licenses** | Restarts `sppsvc` and reinstalls system tokens |

### Phase 4: Exit (lines 272–274)

Displays "Press any key to exit..." and terminates.

## Embedded Resources

### Product Key Database (`:ohookdata`, lines 2047–2598)

The script contains an embedded table of Office product keys for versions 2010–2024, with format:

```
Version_ActivationID_Key_LicenseType_ProductName
```

Keys are matched by product ID during activation. The table covers Retail, Volume (MAK/GVLK), OEM, Subscription, and Preview editions.

### Embedded DLLs

Two custom DLLs are embedded as base64-encoded data within the script:

| Label | Architecture | Purpose |
|---|---|---|
| `:sppc32.dll:` (lines 2601–2664) | 32-bit | Hook DLL for x86 Office — intercepts `sppcs.dll` calls to bypass license validation |
| `:sppc64.dll:` (lines 2671–2734) | 64-bit | Hook DLL for x64 Office — same function, 64-bit ABI |

These are "Ohook" DLLs from MAS — they implement a subset of the `sppc.dll` API to return successful license responses without requiring a real product key.

## Subroutine Reference

### Activation Subroutines (`oh_*`)

| Subroutine | Line | Purpose |
|---|---|---|
| `oh_activate_core` | 142 | Main activation orchestrator |
| `oh_activate_o16c2r` | 190 | Activate Office 16.0 Click-to-Run |
| `oh_activate_o15c2r` | 235 | Activate Office 15.0 Click-to-Run |
| `oh_reset` | 276 | Clear activation variables |
| `oh_getpath` | 299 | Detect installed Office versions via registry |
| `oh_expiredpreview` | 326 | Check for expired preview licenses |
| `oh_ppcpath` | 348 | Determine hook DLL and OSPP paths by architecture |
| `oh_fixprids` | 394 | Fix corrupted ProductReleaseIds registry |
| `oh_installlic` | 424 | Install license files (.xrm-ms) |
| `oh_hookinstall` | 474 | Install Ohook DLL via symlink + extract |
| `oh_hookinstall_ospp` | 531 | Install Ohook for OSPP (Office 2010 / Win 7) |
| `oh_setspp` | 658 | Set SPP/OSPP service names |
| `oh_process` | 678 | Process each product: install key + license + hook |
| `oh_processmsi` | 723 | Process MSI Office installations |
| `oh_clearblock` | 781 | Remove vNext/shared/device license blocks |
| `oh_uninstkey` | 948 | Uninstall competing product keys |
| `oh_licrefresh` | 990 | Refresh Windows Insider Preview licenses |
| `oh_checkapps` | 1002 | Check for running Office applications |
| `oh_hookinstall_error` | 627 | Handle hook installation errors |

### Diagnostic Subroutines (`dk_*`)

| Subroutine | Line | Purpose |
|---|---|---|
| `dk_setvar` | 1030 | Initialize variables (PowerShell path, colors, services) |
| `dk_reflection` | 1248 | Set up .NET reflection for P/Invoke |
| `dk_ckeckwmic` | 1199 | Check if WMIC is available |
| `dk_product` | 1230 | Get Windows edition name |
| `dk_showosinfo` | 1084 | Display OS version, build, architecture |
| `dk_actids` | 1138 | Get all activation IDs for an application |
| `dk_actid` | 1164 | Get activated (key-installed) product IDs |
| `dk_inskey` | 1110 | Install a product key via WMI |
| `dk_refresh` | 1102 | Refresh license status via WMI |
| `dk_errorcheck` | 1300 | Comprehensive system diagnostic (services, WMI, SPP, tokens, WPA registry, etc.) |
| `dk_chkmal` | 1257 | Check for malware/PUP activators (KMSpico, blocked AV URLs, file infector signs) |
| `dk_color` | 1970 | Print colored console output |
| `dk_color2` | 1981 | Print two-color console output |
| `dk_done` | 1994 | Final prompt and optional support webpage links |

## Supported Office Versions

| Version | Type | Support |
|---|---|---|
| Office 2010 (14.0) | MSI | Full — via OSPP hook |
| Office 2013 (15.0) | C2R + MSI | Full |
| Office 2016 (16.0) | C2R + MSI | Full |
| Office 2019 (16.0) | C2R + MSI | Full |
| Office 2021 (16.0) | C2R + MSI | Full |
| Office 2024 (16.0) | C2R + MSI | Full |
| Microsoft 365 Apps | C2R | Full (installs + activates) |

## Customization

To modify the installation, edit the XML configuration block generated at lines 87–108:

- **Language**: Change `Language ID="zh-cn"` to `en-us` or another locale
- **Edition**: Change `Product ID="O365ProPlusRetail"` to `VisioPro2024Volume`, `ProjectPro2024Volume`, etc.
- **Architecture**: Change `OfficeClientEdition="64"` to `"32"`
- **Channel**: Change `MonthlyEnterprise` to `Current`, `Deferred`, etc.
- **Excluded apps**: Add/remove `<ExcludeApp ID="..." />` entries
