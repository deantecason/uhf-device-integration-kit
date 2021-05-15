## UhfWrapper DLL and Tools

`UhfWrapper.dll` is a native Windows wrapper around the vendor‑supplied UHF reader SDK (`SWHidApi.dll`).  
It provides a **stable, versioned `UHF_*` API**, a **diagnostic CLI**, and optional **.NET P/Invoke bindings** so application teams can integrate UHF readers without depending directly on the vendor DLL surface.

- **Language/runtime**: C++ (DLL + CLI), .NET bindings via P/Invoke  
- **Platforms**: Windows 10/11, x86 and x64  
- **Transport**: USB/HID readers supported by the vendor SDK

For detailed usage and reference material, see the `docs/` directory (links below).

---

## Repository Contents

- **`src/uhf_wrapper/`** – Core wrapper implementation and tools  
  - `uhf_wrapper.cpp` / `uhf_wrapper.h`: implementation of the `UHF_*` API and dynamic loading of `SWHidApi.dll`  
  - `uhf_cli.cpp`: command‑line tool (`UhfWrapperCli.exe`) for diagnostics, field checks, and scripted testing  
  - `dotnet/`: minimal .NET projects exposing the same `UHF_*` surface via P/Invoke
- **`docs/`** – Public documentation (user guide, API, CLI, architecture, structure)
- **`scripts/`** – Helper PowerShell scripts for quick hardware/stream tests on x86/x64
- **`vendor/`** – Local copy of the vendor SDK (not distributed from this repository)
- **`build-x64/`, `build-x86/`** – Local build output directories (not committed)

For a visual overview of the layout, see `docs/STRUCTURE.md`.

---

## High‑Level Design

The wrapper is intentionally **thin** and **predictable**:

- Dynamically loads `SWHidApi.dll` at runtime (path can be overridden via `UHF_VENDOR_DLL`)
- Re‑exports **all** vendor `SWHid_*` functions, normalizing x86/x64 naming differences
- Adds a **friendly `UHF_*` API** with:
  - Consistent success/failure conventions
  - Centralized error handling (`UHF_GetLastError`, `UHF_GetLastErrorCode`)
  - Safer helpers for transport configuration, work modes, selection, and write flows
- Provides a CLI which drives the same `UHF_*` surface for:
  - Smoke tests and diagnostics
  - Multi‑tag‑safe select/write sequences
  - Power/RSSI calibration workflows

For a deeper architectural description, see `docs/ARCHITECTURE.md`.

---

## Getting Started

### Prerequisites

- Windows 10 or Windows 11
- CMake and Visual Studio Build Tools (or a full Visual Studio installation)
- Access to the vendor SDK containing `SWHidApi.dll`  
  (this DLL is **not** redistributed from this repo; see `NOTICE_VENDOR.md`)

### Cloning

```powershell
git clone <this-repository-url>
cd IN53xx_workflow_DLL_x64
```

If you are working behind a corporate proxy or with mirrored remotes, configure Git remotes as needed before cloning.

---

## Build Instructions

The wrapper and CLI are built via CMake for both x64 and x86. Below is a typical workflow using out‑of‑source builds:

```powershell
# x64
cmake -S src/uhf_wrapper -B build-x64 -A x64
cmake --build build-x64 --config Release

# x86
cmake -S src/uhf_wrapper -B build-x86 -A Win32
cmake --build build-x86 --config Release
```

Build artifacts (DLL + CLI) will be produced under `build-x64/` and `build-x86/` respectively.

### Vendor DLL Placement

At runtime the wrapper must be able to locate `SWHidApi.dll`:

- **Default**: place `SWHidApi.dll` next to `UhfWrapper.dll` in the output directory, or
- **Override**: set an absolute path via environment variable:

```powershell
set UHF_VENDOR_DLL=C:\full\path\to\SWHidApi.dll
```

If the vendor DLL cannot be loaded, `UHF_*` calls will fail with an appropriate error code/message.

---

## Using the CLI

The CLI (`UhfWrapperCli.exe`) is the primary tool for:

- Verifying connectivity to a reader
- Exercising read/stream operations
- Running select/write flows that are safe in multi‑tag environments
- Calibrating power and RSSI for a known tag

### Typical checks

```powershell
UhfWrapperCli.exe --friendly status
UhfWrapperCli.exe --friendly read-once
UhfWrapperCli.exe --friendly read-stream --duration 2000
```

To clear lingering selection masks and use safer timing for unstable USB‑over‑IP links:

```powershell
UhfWrapperCli.exe --friendly select-clear
UhfWrapperCli.exe --friendly --interval 1000 --timeout 10000 read-count 1
```

For write operations, always use select flows to avoid touching unintended tags:

```powershell
UhfWrapperCli.exe --friendly select-epc <EPC_HEX>
UhfWrapperCli.exe --friendly write-epc <NEW_EPC_HEX> 00000000
UhfWrapperCli.exe --friendly select-clear
```

For detailed CLI options and scenarios, see `docs/CLI.md` and `docs/USER_GUIDE.md`.

---

## Integrating the `UHF_*` API

### Native (C/C++)

Link against `UhfWrapper.dll` and consume the `UHF_*` exports. The API follows these conventions:

- **Action‑style** functions:
  - Return `1` on success
  - Return `0` on failure
- **Value‑returning** functions:
  - Return `>= 0` on success
  - Return `-1` on failure
- On any failure, call:
  - `UHF_GetLastError()` for a human‑readable error string
  - `UHF_GetLastErrorCode()` for a stable numeric error code

Transport helpers (e.g. `UHF_GetTransport`, `UHF_SetTransportUsb`, `UHF_EnsureUsbTransport`) and work‑mode helpers (e.g. `UHF_SetWorkModeAnswer`, `UHF_SetWorkModeActive`) are exposed to keep integration code straightforward and consistent across applications.

### .NET (C#)

The `src/uhf_wrapper/dotnet/` folder contains example projects that:

- Declare the `UHF_*` P/Invoke signatures
- Build per‑architecture wrappers (`UhfWrapperNet*.csproj`)

You can either reference these projects directly, or copy the P/Invoke signatures into your own solution. Ensure that the appropriate `UhfWrapper.dll` (x86/x64) is deployed next to your .NET application.

API reference details, including the full function list, are documented in `docs/API.md`.

---

## Documentation Map

If you are new to the project, the recommended reading order is:

1. **User guide** – `docs/USER_GUIDE.md`  
   How to build, configure the vendor DLL, and use the CLI and common flows.
2. **API reference** – `docs/API.md`  
   Complete `UHF_*` interface, data types, and error codes.
3. **CLI reference** – `docs/CLI.md`  
   CLI subcommands, flags, and usage patterns.
4. **Architecture** – `docs/ARCHITECTURE.md`  
   Layering, data flow, error handling and compatibility notes.
5. **Structure** – `docs/STRUCTURE.md`  
   Directory layout and mapping between components and artifacts.

Policy and support documents:

- `SECURITY.md` – How to report vulnerabilities securely.
- `SUPPORT.md` – Support model and channels for usage questions and bug reports.
- `NOTICE_VENDOR.md` – Vendor DLL distribution and licensing boundaries.

---

## Contributing

Contributions are welcome. For feature work or significant refactors:

- **Open an issue** first to discuss requirements and design, using the templates under `.github/ISSUE_TEMPLATE/`.
- **Follow the existing coding style** in the C++ and C# sources.
- **Add or update tests** where applicable under `src/uhf_wrapper/tests/`.
- **Run the full build** (both x64 and x86) before opening a pull request.

Pull requests should include:

- A concise summary of the change and motivation
- Any relevant diagnostics (logs, CLI sequences, or traces)
- Notes on breaking changes or behavioral differences, if any

Refer to `.github/pull_request_template.md` for the expected PR shape.

---

## Security and Compliance

- Do **not** commit vendor binaries or proprietary SDK content outside the designated `vendor/` area.
- Treat `SWHidApi.dll` and any associated SDK materials according to the vendor license terms.
- Report potential security issues via the process described in `SECURITY.md` rather than filing a public bug.

Automated security checks (e.g. CodeQL, gitleaks) are configured under `.github/workflows/` and should be allowed to run on all pull requests targeting the main branch.

---

## License and Third‑Party Notices

This project’s licensing terms are defined in the repository’s license file (if present).  
The vendor SDK and `SWHidApi.dll` are **not** covered by this project’s license; see `NOTICE_VENDOR.md` for details and obligations.

If you are unsure whether your intended use is permitted, consult your legal or licensing contact before distributing binaries that include or depend on the vendor SDK.

