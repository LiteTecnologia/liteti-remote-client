# Windows Builder (Portable Installer) — Design

**Date:** 2026-06-03
**Branch:** `feature/liteti-branding`
**Status:** Approved (pending spec review)
**Approach:** A — align `build.py` to the real internal names (`rustdesk`), brand only the surface.

## Problem

The local Windows build path (`build.py build_flutter_windows`) cannot compile. The
rebrand commit `f7a9492cf` renamed the **internal** crate/lib/binary names to
`litetiremote` everywhere — including `build.py`. A later commit `8e481168c`
("keep internal crate/package names for build compatibility") reverted the **Cargo
side** (`Cargo.toml [lib] name`, `BINARY_NAME`, portable-packer package name) back to
`rustdesk`, but **did not revert `build.py`**, leaving it expecting `litetiremote` names.

On Linux these mismatches were non-fatal (the `strip` step fails silently via
`os.system`, so the `.deb` shipped an unstripped `librustdesk.so`). On Windows the
same mismatches are fatal — `build.py` does `exit(-1)` when it cannot find the DLL.

Authoritative reference: the pre-rebrand `build.py` (`git show f7a9492cf^:build.py`)
used `librustdesk.dll`, `rustdesk.exe`, and `rustdesk-portable-packer.exe`.

## Confirmed blockers (local Windows path)

| # | `build.py` line | Expected (broken) | Real producer | Effect |
|---|-----------------|-------------------|---------------|--------|
| 1 | 437 | `target/release/liblitetiremote.dll` | `Cargo.toml [lib] name = "librustdesk"` → `librustdesk.dll` | `exit(-1)` after cargo build |
| 2 | 450 | `…/liteti-remote.exe` | `flutter/windows/CMakeLists.txt:7` `BINARY_NAME = "rustdesk"` → `rustdesk.exe` | packer cannot find exe |
| 3 | 453 / 456 | `liteti-remote-portable-packer.exe` | `libs/portable/Cargo.toml:2` `name = "rustdesk-portable-packer"` → `rustdesk-portable-packer.exe` | `os.rename` fails |

## Solution (Approach A)

Keep internal names as `rustdesk` (per `8e481168c`); fix `build.py` to expect them.
User-facing branding comes from `Runner.rc`, the portable packer's winres metadata,
`APP_NAME`, and the final artifact name.

### Change set

**1. `build.py` — revert 3 internal references**
- Line 437: `liblitetiremote.dll` → `librustdesk.dll`
- Line 450: `…/liteti-remote.exe` → `…/rustdesk.exe`
- Lines 453 + 456: source `liteti-remote-portable-packer.exe` → `rustdesk-portable-packer.exe`
- **Preserve:** intermediate `liteti-remote_portable.exe` and final
  `liteti-remote-{version}-install.exe` (line 460). The delivered artifact stays Liteti.

**2. `flutter/windows/runner/Runner.rc` — file properties of `rustdesk.exe`**
- `CompanyName`: `Purslane Ltd` → `Liteti Tecnologia`
- `LegalCopyright`: `Copyright © 2025 Purslane Ltd. All rights reserved.` → `Copyright © 2025 Liteti Tecnologia`
- Keep `InternalName` / `OriginalFilename` = `rustdesk` (truthful to the binary).
- `ProductName` / `FileDescription` already "Liteti Remote" — no change.

**3. `libs/portable/Cargo.toml` `[package.metadata.winres]` — file properties of the installer `.exe`**
- `ProductName`: `RustDesk` → `Liteti Remote`
- `FileDescription`: `RustDesk Remote Desktop` → `Liteti Remote - Acesso Remoto Seguro`
- `OriginalFilename`: `rustdesk.exe` → `liteti-remote-install.exe`
- Keep `[package] name = "rustdesk-portable-packer"` (build.py now expects it).

### Deliberately unchanged (Approach A)

- `BINARY_NAME` / `project()` in `flutter/windows/CMakeLists.txt` → stay `rustdesk`.
- `APP_PREFIX = "rustdesk"`, `RUSTDESK_APPNAME`, `RuntimeBroker_rustdesk.exe`, and the
  `"rustdesk"` magic handshake (`generate.py` ↔ `bin_reader.rs`) → untouched.
- **Known cosmetic leak:** runtime extraction temp dir is `%TEMP%\rustdesk`.

## Validation

1. **Static (in-repo):** every name the Windows `build.py` path expects must have a
   matching producer in the sources (DLL, exe, packer). Verify by grep/diff.
2. **Real (Windows machine):** user runs `python3 build.py --flutter --portable
   [--hwcodec --vram]` on Windows; iterate on any remaining errors. Cannot be compiled
   from the Linux dev host.

## Out of scope (parity with the `.deb`)

CI workflow (`flutter-build.yml`), MSI/WiX packaging, code signing, ARM64, runtime
URI-scheme / service-name rebrand. Artifact is **unsigned**, single portable `.exe`.

## Follow-ups (not now)

- `build.py:429` `strip …/liblitetiremote.so` → `librustdesk.so` (Linux path; currently
  fails silently, shipping an unstripped 43 MB lib). Beneficial but touches the Linux build.
- `build.py:411` macOS dylib copy has the same `litetiremote` mismatch — defer to the macOS session.
