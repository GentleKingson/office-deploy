# office-deploy

A single-file Windows batch deployment utility for **Microsoft 365 Apps for enterprise**, with interactive and unattended execution, deterministic exit codes, Office Deployment Tool validation, and Windows runtime CI.

> **Project status**
>
> Windows runtime behavior is continuously validated on **Windows Server 2022** and **Windows Server 2025**.  
> Full Microsoft 365 installation E2E testing is intentionally handled separately from the lightweight runtime test suite.

## Overview

`office-deploy` provides a repeatable way to deploy Microsoft 365 Apps for enterprise from a Windows batch script.

The project focuses on:

- interactive and unattended execution;
- a read-only Office installation preflight for interactive option `[1]`;
- Microsoft Office Deployment Tool-based installation;
- script-relative configuration and download paths;
- unattended configuration generation;
- deterministic exit codes for automation;
- structured runtime logging;
- Office Deployment Tool integrity validation;
- recovery from interrupted or invalid ODT downloads;
- WOW64 → native Windows command relaunch;
- Windows runtime validation through GitHub Actions.

The project is intentionally kept as a **single primary Batch file** so it remains easy to copy, inspect, and run without an additional runtime or package manager.

## Embedded Ohook Baseline

**Embedded Ohook core synchronized with MAS 3.12, with Office Deploy integration changes.**

| Item | Pinned value |
|---|---|
| Embedded Ohook version | MAS 3.12 |
| Upstream file commit | [`f34d025d5102a790c75c839c8d46b672284729a5`](https://github.com/massgravel/Microsoft-Activation-Scripts/blob/f34d025d5102a790c75c839c8d46b672284729a5/MAS/Separate-Files-Version/Activators/Ohook_Activation_AIO.cmd) |
| Upstream file blob | `cec64c62a39cb61254bf5d39091f399b6d1d4de7` |
| Local integration base | `d109bef1bde90eb2004927b5a6e6358eb0db4de2` |

Office Deploy retains ownership of:

- its CLI, menu, and native-process relaunch contract;
- the interactive installation preflight;
- the fail-closed supported-Office detector;
- the bounded licensing-readiness gate;
- the no-Office mutation guard and CI safety boundary.

The MAS standalone argument, elevation, QuickEdit, and `_cmdf` startup state machines are intentionally not embedded. The legacy `BIN\sppc32.dll` / `BIN\sppc64.dll` override has also been removed: hook installation always extracts the pinned in-script payload. Office Deploy adds two explicit local hardening deviations: `try`/`finally` cleanup for the MAS 3.12 PE writer's GUID-named temporary file, and an Office application key check before installation and after a failed installation so an already-present generic key does not turn a repeat run into a provider-specific `0x80070005` failure.

At this pin, `:ohookdata` and `:msiofficedata` match the MAS 3.12 source label-for-label. The decoded embedded payloads remain 9,216 bytes each and have these SHA-256 values:

| Payload | SHA-256 |
|---|---|
| `sppc32.dll` | `09865ea5993215965e8f27a74b8a41d15fd0f60f5f404cb7a8b3c7757acdab02` |
| `sppc64.dll` | `393a1fa26deb3663854e41f2b687c188a9eacd87b23f17ea09422c4715cb5a9f` |

---

## Quick Start

### Interactive mode

Run **Command Prompt or Windows Terminal as Administrator**, then execute:

```bat
officeDeploy.bat
```

The interactive mode displays an action menu:

```text
[1] Install Microsoft 365 Apps for enterprise
[2] Activate installed Office
[3] Exit
```

- `[1] Install` first performs a read-only Office installation preflight. It uses the same exact `O365ProPlusRetail` x64 product/file probe as post-install verification and the same broader supported-Office detector as activation. If the target Microsoft 365 Apps for enterprise installation already exists, the utility reports its detected version and platform and asks whether to return or continue deployment. Other or incomplete Office installations produce an explicit warning; a detector failure stops before configuration generation. Continuing performs the normal interactive installation flow, verifies the installed files, then enters the shared activation core. Before any product key or Ohook change, that core runs the MAS 3.12 licensing-health diagnostics and a bounded licensing-readiness gate.
- `[2] Activate installed Office` only acts on Office that is already installed. It detects Office through the same MAS-derived detector the activation engine itself uses (`:oh_check_supported_office` → `:oh_getpath`, which requires the registry key **and** the Office marker file, both 32/64-bit aware, **plus** the upstream ClickToRun service validity check), then uses the same diagnostic, readiness, and Ohook core as `[1]`. It does not download or reinstall Office. Ohook is idempotent, so re-running `[2]` is safe; it reinstalls cleanly.

> Both activation paths (`[1]` post-install and `[2]` Activate installed Office) share the repository's existing MAS-derived Ohook activation implementation. GitHub Actions does not execute Office activation.
>
> Review the source and applicable software licensing requirements before using either activation path.

### Unattended installation

For automated deployment:

```bat
officeDeploy.bat --unattended
```

Unattended mode:

- requires administrator privileges;
- deliberately bypasses the interactive installation preflight and still runs ODT to converge the deployment configuration;
- generates an unattended Office configuration;
- downloads or reuses a validated Office Deployment Tool;
- installs Microsoft 365 Apps without installation UI;
- validates the installed Office files and Click-to-Run state;
- writes a structured log;
- returns a deterministic process exit code;
- does **not** enter the legacy activation path.

### Generate configuration only

To generate and validate `Configuration.xml` without downloading or installing Office:

```bat
officeDeploy.bat --unattended --config-only
```

This mode:

- does not require administrator privileges;
- does not run the Office installation preflight;
- does not download `setup.exe`;
- does not install Office;
- does not modify licensing state;
- does not enter the activation path.

### Help

```bat
officeDeploy.bat --help
```

---

## Command Line

| Command | Description |
|---|---|
| `officeDeploy.bat` | Interactive menu: install Microsoft 365, activate installed Office, or exit |
| `officeDeploy.bat --unattended` | Full unattended deployment |
| `officeDeploy.bat --unattended --config-only` | Generate and validate configuration only |
| `officeDeploy.bat --help` | Display command usage |

The parser accepts unattended/config-only arguments in either order.

For example:

```bat
officeDeploy.bat --config-only --unattended
```

is equivalent to:

```bat
officeDeploy.bat --unattended --config-only
```

---

## Default Office Configuration

The generated `Configuration.xml` currently deploys:

| Setting | Value |
|---|---|
| Product | Microsoft 365 Apps for enterprise |
| Product ID | `O365ProPlusRetail` |
| Architecture | 64-bit |
| Update channel | `MonthlyEnterprise` |
| Language | `zh-cn` |
| Office updates | Disabled by configuration |
| Existing MSI Office | Removed through `RemoveMSI` |

The following applications are excluded by default:

- Groove
- Lync
- OneDrive
- Outlook for Windows
- Teams

Display behavior depends on execution mode:

| Mode | ODT display level |
|---|---|
| Interactive | `Full` |
| Unattended | `None` |

Both modes generate the same Office deployment configuration apart from the display level.

---

## Deployment Flow

### Interactive

```text
Start
  ↓
Parse arguments
  ↓
Native architecture check
  ↓
Administrator check
  ↓
Product menu
  ↓
Read-only Office installation preflight
  ├─ No Office → continue automatically
  ├─ Target Microsoft 365 x64 → Return / Continue / Exit
  ├─ Other Office → Continue / Return / Exit
  ├─ Incomplete or broken Office → Continue / Return / Exit
  └─ Detection error → Retry / Return / Exit
  ↓
Generate Configuration.xml
  ↓
Validate / obtain Office Deployment Tool
  ↓
Install Microsoft 365 Apps
  ↓
Verify installation
  ↓
MAS licensing-health diagnostics
  ↓
Detect Office and Windows Server
  ↓
Wait for ClickToRun / SPP / WMI readiness
  ↓
Ohook activation and cleanup
  ↓
Exit
```

### Unattended

```text
Start
  ↓
Parse arguments
  ↓
Native architecture check
  ↓
Administrator check
  ↓
Generate unattended Configuration.xml
  ↓
Validate / obtain Office Deployment Tool
  ↓
Install Microsoft 365 Apps
  ↓
Verify installation
  ↓
Write result and exit code
```

Unattended execution deliberately bypasses the legacy activation path.

### Interactive installation preflight

The preflight is limited to interactive option `[1]`. It is read-only: it does not generate XML, download ODT, uninstall Office, install Office, modify licensing state, or activate Office.

It combines two existing definitions rather than introducing another Office detector:

- `:probe_target_office` checks the Click-to-Run configuration for `O365ProPlusRetail`, `x64`, a non-empty version and installation path, and the `WINWORD.EXE`, `EXCEL.EXE`, and `POWERPNT.EXE` files. Post-install verification delegates to this same probe.
- `:oh_check_supported_office` remains the broader shared detector for supported C2R/MSI and 32/64-bit Office. It requires the corresponding registry and marker-file evidence and rejects a C2R candidate whose required service is missing. Its explicit `SUPPORTED_PROBE_OK` / `SUPPORTED_PROBE_ERROR` operational channel distinguishes expected absence from failures of `reg.exe`, registry access, SCM access, service queries, or internal detector state.

The resulting interactive states are `NONE`, `TARGET_INSTALLED`, `OTHER_OFFICE`, `BROKEN_OFFICE`, and `DETECTION_ERROR`. Failures from either the exact target probe or the broader supported-Office probe become `DETECTION_ERROR`, fail closed, and never enter installation automatically. These are internal states only; the documented CLI exit-code contract is unchanged.

---

## Office Deployment Tool Handling

The script does not blindly trust an existing `setup.exe`.

ODT acquisition follows a recoverable flow:

```text
Existing setup.exe
      ↓
Validate file
   ↙       ↘
Valid     Invalid
  ↓          ↓
Reuse      Remove
              ↓
      Download temporary file
              ↓
        Validate temporary
              ↓
      Promote to setup.exe
```

Validation includes:

- file existence;
- non-zero file size;
- valid Windows Authenticode signature.

Downloads are first written to:

```text
setup.exe.download.tmp
```

The temporary file is promoted to:

```text
setup.exe
```

only after validation succeeds.

This prevents an interrupted or incomplete download from becoming the cached executable used by a later run.

---

## Generated Files

| Path | Purpose |
|---|---|
| `Configuration.xml` | Generated Office Deployment Tool configuration |
| `setup.exe` | Validated Office Deployment Tool executable |
| `setup.exe.download.tmp` | Temporary ODT download before validation |

Unattended execution also writes:

```text
%TEMP%\office-deploy\office-deploy.log
```

The log contains execution information such as:

```text
MODE
CONFIG_ONLY
SCRIPT_PATH
WINDOWS_VERSION
PROCESSOR_ARCHITECTURE
NATIVE_RELAUNCH
ADMIN_CHECK
ODT_SOURCE
ODT_EXIT_CODE
VERIFY_RESULT
RESULT
EXIT_CODE
```

---

## Exit Codes

Automation can rely on the following stable exit-code contract:

| Code | Meaning |
|---:|---|
| `0` | Success |
| `2` | Invalid command-line arguments |
| `10` | Administrator privileges required |
| `20` | Environment validation failed |
| `30` | ODT download / transport failed |
| `31` | ODT validation, cache recovery, or promotion failed |
| `40` | Configuration generation or validation failed |
| `50` | Office Deployment Tool installation failed |
| `60` | Post-install verification failed |
| `70` | Internal or unexpected failure |

Underlying PowerShell or ODT error values may also be recorded in the unattended log for diagnostics.

---

## Architecture Handling

The script detects whether it is running under a redirected Windows command environment.

On x64 Windows, a 32-bit invocation can be relaunched synchronously through:

```text
SysWOW64 cmd.exe
    ↓
Sysnative cmd.exe
    ↓
native officeDeploy.bat
```

The native child exit code is propagated back to the original caller.

The parser preserves `%0`, so `%~f0` and `%~dp0` continue to refer to the real batch file after argument processing.

An ARM64 / `SysArm32` path is also present in the script, but it is not currently covered by the x64 GitHub-hosted runtime test matrix.

---

## Runtime CI

The repository includes:

```text
.github/workflows/windows-runtime.yml
```

The workflow runs on:

| Environment | Runtime coverage |
|---|---|
| Windows Server 2022 | ✅ |
| Windows Server 2025 | ✅ |

The runtime suite validates contracts `T1` through `T24` without downloading ODT or installing Microsoft 365.

Coverage includes:

- `--help`;
- invalid argument handling;
- config-only argument validation;
- forward and reverse argument ordering;
- runtime generation and parsing of `Configuration.xml`;
- unattended log output;
- script-relative paths;
- execution from another working directory;
- paths containing spaces;
- invalid cached ODT recovery;
- deterministic exit code `31`;
- real 32-bit `SysWOW64\cmd.exe` execution;
- WOW64 → `Sysnative` relaunch;
- native child → parent exit-code propagation;
- native child `SCRIPT_PATH` preservation;
- interactive menu rendering and Exit mapping (`T11`);
- option `[2]` Office detection through the same MAS-derived detector the activation engine uses (`:oh_check_supported_office`), against mocked registry **and** Office marker files **and** the ClickToRun service: 64-bit Click-to-Run and 32-bit `Wow6432Node` MSI pass; registry-without-marker is rejected; registry+marker without the ClickToRun service is rejected (`T12`, instruments a test bat copy — no Ohook activation is executed);
- the `:oh_activate_core` fail-fast guard, which must abort before the mutating cleanup routines when no supported Office is present (`T13`);
- the repository blob line-ending contract: the exact bytes Git stores for `officeDeploy.bat` are asserted, byte-for-byte, to be CRLF (`T14`);
- running that same blob end-to-end via `call` and asserting a clean config-only run, with no batch-control-flow-break signature (English and Chinese) in the output (`T15`/`T15b`);
- interactive option `[1]` configuration-generation regression: the real `choice [1] -> preflight NONE -> :write_config` control flow is driven through the interactive menu and must generate a valid `Configuration.xml` without `CONFIG_WRITE_FAILED`, stopping before ODT / network / install (`T16`, instruments a test bat copy with a deterministic `NONE` detector result);
- the `:write_config` ambient `ERRORLEVEL` contract: `:write_config` must succeed and emit a valid `Configuration.xml` regardless of any `ERRORLEVEL` inherited from its caller (`T16b`, instruments a test bat copy);
- the activation-core ordering contract: the single early `error` reset, MAS 3.12 preflight, Office detection, Server detection, and the immediately adjacent `readiness call -> non-zero abort` contract cannot be reordered or silently dropped (`T17`);
- the licensing provider against the runners' real `sppsvc`, `Winmgmt`, `SoftwareLicensingService`, and `RefreshLicenseStatus`, plus a copied-script non-zero `RefreshLicenseStatus.ReturnValue` fixture (`T18`);
- the C2R service state machine in a fully stubbed test copy: missing service fails immediately, stopped service is started once and succeeds, an unstartable service fails within the bound, and Office 15 accepts a running `OfficeSvc` fallback (`T18c`);
- Windows Server detection against the real Server 2022 and Server 2025 runner registry, which must establish `winserver=1` (`T19`);
- activation-core fail-closed behavior: a copied core with forced readiness failure must return non-zero before product processing, Generic Key installation, Ohook installation, or license cleanup (`T20`);
- the installation-preflight detector state machine against mocked registry, marker-file, application-file, service, architecture, incomplete-install, unavailable-PowerShell, and broad-detector operational-failure fixtures (`T21`);
- the real interactive option `[1]` preflight control flow: `NONE` continues; `TARGET_INSTALLED`, `OTHER_OFFICE`, and `BROKEN_OFFICE` exercise their Return/Continue mappings; and `DETECTION_ERROR` cannot silently enter installation (`T22`, instruments a test bat copy and stops before ODT/network/install);
- the MAS 3.12 static synchronization contract: removed external-hook paths, `[IO.File]` usage, GUID temporary naming, provenance, payload hash comments, local helper presence, activation-core ordering, repeat-run exact-key guard, and absence of production CI seams (`T23`);
- both raw embedded payload hashes and the real `:oh_extractdll` output in a copied script, including MZ/PE structure, architecture, final PE checksum, dynamically observed GUID temporary PE cleanup, and absence of `BIN` use (`T24`).

See:

```text
.github/workflows/windows-runtime.yml
```

for the complete assertions.

### CI safety boundary

The runtime workflow intentionally does **not**:

- download the Office Deployment Tool from the network;
- execute `setup.exe /configure`;
- install Microsoft 365;
- execute Office product activation or licensing mutation;
- execute Ohook activation or install a hook into an Office directory;
- test ARM64 / `SysArm32`.

`T12` exercises option `[2]` only up to the point where Ohook would be invoked. There is no test seam in the production script: CI instruments a **copied** bat at runtime to make the Ohook call unreachable, then drives the read-only `:oh_check_supported_office` detector against mocked fixtures. `T13`, `T16`, `T16b`, `T18`, `T18c`, `T19`, `T20`, `T21`, `T22`, and `T24` use the same copy-instrumentation approach. `T21` directly drives the combined installation-preflight classifier; `T22` drives the real option `[1]` menus and configuration path with deterministic detector states and a stop before ODT. `T18` directly exercises the bounded provider readiness helper and calls the real `RefreshLicenseStatus`; its negative fixture preserves the real provider query but substitutes a non-zero method result. `T18c` stubs all service/provider behavior. `T20` enters the copied activation core with mutating diagnostics stubbed and readiness forced to fail, proving that no product, key, hook, or cleanup routine is reached. `T23` is static-only. `T24` calls only `:oh_extractdll` and writes only runner-temporary PE files. Real Office installation and activation stay outside the CI boundary.

Actual C2R/MSI activation, repeat-run behavior, and ARM64 / `SysArm32` remain independent Windows VM acceptance tests and are required before merging an Ohook baseline upgrade. They are never moved onto GitHub-hosted runners.

These belong to separate validation stages.

---

## Compatibility

The project distinguishes between **implemented code paths** and **runtime-verified environments**.

| Platform | Status |
|---|---|
| Windows Server 2022 x64 | Runtime CI verified |
| Windows Server 2025 x64 | Runtime CI verified |
| Windows 10 / 11 x64 | Expected target environment; not currently covered by GitHub-hosted runtime CI |
| Windows ARM64 | Architecture path implemented; runtime validation pending |
| Legacy Windows / PowerShell environments | Not part of the current verified compatibility baseline |

Do not interpret older README history or legacy activation code paths as a current support guarantee.

---

## Security Model

The deployment path includes several defensive controls:

- ODT paths are relative to the script rather than the caller's working directory;
- network downloads use a temporary file;
- ODT is validated before execution;
- invalid cached ODT files are removed and recovered automatically;
- post-install activation waits for ClickToRun, Software Protection, WMI, and the licensing provider before installing a key or modifying Ohook;
- unattended execution returns deterministic failure codes;
- CI runs with read-only repository permissions;
- CI uses `pull_request`, not `pull_request_target`;
- runtime CI does not use repository secrets;
- runtime CI never executes Office installation or activation.

As with any script that installs system-wide software, inspect the source before running it with administrator privileges.

---

## Repository Layout

```text
office-deploy/
├── .github/
│   └── workflows/
│       └── windows-runtime.yml
├── .gitignore
├── README.md
└── officeDeploy.bat
```

`officeDeploy.bat` is intentionally the primary implementation file.

---

## Development

When modifying the Batch script:

1. keep interactive and unattended behavior separate where their requirements differ;
2. preserve the stable exit-code contract;
3. avoid depending on the current working directory;
4. capture `%ERRORLEVEL%` immediately after commands whose result matters;
5. preserve `%0` during argument parsing;
6. do not bypass ODT validation;
7. keep unattended execution free of `choice`, `pause`, and activation;
8. keep installation preflight read-only and limited to interactive option `[1]`; it must not alter unattended or config-only behavior;
9. add or update Windows Runtime CI assertions for behavior changes.

Before merging, Windows Runtime CI should pass on both configured Windows runners.

---

## Current Validation Boundary

The following areas are **not yet covered by automated E2E validation**:

- live ODT network download;
- full Microsoft 365 installation;
- post-install verification against a freshly installed Office environment;
- Windows client editions through GitHub-hosted runners;
- ARM64 / `SysArm32`;
- activation.

A future Office E2E workflow should remain separate from the lightweight Runtime CI.

---

## Attribution

Parts of the legacy activation-related implementation were derived from the **Microsoft Activation Scripts (MAS)** project.

Upstream project:

`massgravel/Microsoft-Activation-Scripts`

The deployment and unattended-control layers in this repository should be treated separately from the inherited activation implementation.

---

## Licensing Notice

This repository currently does not include a standalone `LICENSE` file.

Before redistribution, packaging, or accepting broader third-party contributions, review the licensing requirements of all upstream-derived code and add an explicit repository license where appropriate.

---

## Disclaimer

This project is not affiliated with or endorsed by Microsoft.

Microsoft, Windows, Office, and Microsoft 365 are trademarks of Microsoft and/or its affiliates.

Users are responsible for complying with applicable Microsoft licensing terms and for using properly licensed software.
