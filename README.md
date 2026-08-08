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

- `[1] Install` performs the normal interactive Microsoft 365 installation flow.
- `[2] Activate installed Office` only acts on Office that is already installed. It detects the installed Office, reports its license state, and activates it by reusing the script's existing Ohook activation capability — the same implementation the `[1]` post-install path uses. It does not download or reinstall Office.

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
Generate Configuration.xml
  ↓
Validate / obtain Office Deployment Tool
  ↓
Install Microsoft 365 Apps
  ↓
Verify installation
  ↓
Legacy interactive post-install path
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

The runtime suite validates the Batch control flow without downloading ODT or installing Microsoft 365.

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
- interactive menu rendering and Exit mapping (`T11`).

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
- execute the activation path;
- execute Ohook;
- test ARM64 / `SysArm32`.

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
8. add or update Windows Runtime CI assertions for behavior changes.

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
