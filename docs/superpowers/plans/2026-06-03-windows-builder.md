# Windows Builder (Portable Installer) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Make the local Windows build (`build.py build_flutter_windows`) compile and produce `liteti-remote-{version}-install.exe`, by aligning `build.py` to the real internal `rustdesk` names and branding the user-facing surfaces.

**Architecture:** Approach A — internal crate/lib/binary names stay `rustdesk` (per commit `8e481168c`); `build.py`'s Windows path is reverted to expect those real names. Branding lives in `Runner.rc` (the `rustdesk.exe` file properties), `libs/portable/Cargo.toml` winres (the installer `.exe` file properties), and the final artifact name.

**Tech Stack:** Python build orchestration (`build.py`), Rust/Cargo (`libs/portable` packer), Windows resource script (`Runner.rc`), Flutter Windows runner.

**Note on testing:** The changed files are build scripts and resource metadata — there is no unit-testable function. Each task is verified **statically** (grep/diff asserting the expected name is present and the broken name is gone). The **functional** test (an actual Windows compile) is a manual handoff in Task 5, run by the user on a Windows machine; it cannot run on the Linux dev host.

**Spec:** `docs/superpowers/specs/2026-06-03-windows-builder-design.md`

---

### Task 1: Align `build.py` Windows path to real internal names

Fixes the 3 build-breaking mismatches. The delivered artifact name stays Liteti.

**Files:**
- Modify: `build.py` (function `build_flutter_windows`, ~lines 434–462)

- [ ] **Step 1: Capture the before-state (verification baseline)**

Run:
```bash
grep -nE "liblitetiremote\.dll|liteti-remote\.exe|liteti-remote-portable-packer\.exe" build.py
```
Expected (current broken state): matches on lines ~437, ~450, ~453, ~456.

- [ ] **Step 2: Fix the DLL existence check (line ~437)**

Replace:
```python
        if not os.path.exists("target/release/liblitetiremote.dll"):
```
With:
```python
        if not os.path.exists("target/release/librustdesk.dll"):
```

- [ ] **Step 3: Fix the startup exe passed to the packer (line ~450)**

Replace:
```python
        f'python3 ./generate.py -f ../../{flutter_build_dir_2} -o . -e ../../{flutter_build_dir_2}/liteti-remote.exe')
```
With:
```python
        f'python3 ./generate.py -f ../../{flutter_build_dir_2} -o . -e ../../{flutter_build_dir_2}/rustdesk.exe')
```

- [ ] **Step 4: Fix the packer output name in both branches (lines ~453 and ~456)**

Replace BOTH occurrences of:
```python
        os.replace('./target/release/liteti-remote-portable-packer.exe',
```
and
```python
        os.rename('./target/release/liteti-remote-portable-packer.exe',
```
changing only the source path `liteti-remote-portable-packer.exe` → `rustdesk-portable-packer.exe`. Keep the destination `'./liteti-remote_portable.exe'` unchanged.

Result:
```python
    if os.path.exists('./liteti-remote_portable.exe'):
        os.replace('./target/release/rustdesk-portable-packer.exe',
                   './liteti-remote_portable.exe')
    else:
        os.rename('./target/release/rustdesk-portable-packer.exe',
                  './liteti-remote_portable.exe')
```

- [ ] **Step 5: Verify the broken names are gone and the artifact name is preserved**

Run:
```bash
grep -nE "liblitetiremote\.dll|liteti-remote\.exe|liteti-remote-portable-packer\.exe" build.py | grep -v "install.exe"
```
Expected: **no output** (all three internal mismatches removed).

Run:
```bash
grep -nE "librustdesk\.dll|rustdesk\.exe|rustdesk-portable-packer\.exe|liteti-remote-\{version\}-install\.exe|liteti-remote-%s-install|liteti-remote_portable\.exe" build.py
```
Expected: `librustdesk.dll` (1), `rustdesk.exe` (1 in flutter path), `rustdesk-portable-packer.exe` (2), the final `liteti-remote-...-install.exe` rename present, and intermediate `liteti-remote_portable.exe` present.

- [ ] **Step 6: Commit**

```bash
git add build.py
git commit -m "fix(build): align Windows build.py path to real rustdesk internal names"
```
(No AI co-authorship trailer — per project CLAUDE.md.)

---

### Task 2: Brand `Runner.rc` file properties of `rustdesk.exe`

Updates the version-info shown in Windows file Properties → Details for the running process.

**Files:**
- Modify: `flutter/windows/runner/Runner.rc` (~lines 92, 96)

- [ ] **Step 1: Verify current state**

Run:
```bash
grep -nE "CompanyName|LegalCopyright|InternalName|OriginalFilename|ProductName|FileDescription" flutter/windows/runner/Runner.rc
```
Expected: `CompanyName` = "Purslane Ltd", `LegalCopyright` mentions "Purslane Ltd", `InternalName`/`OriginalFilename` = rustdesk, `ProductName`/`FileDescription` = "Liteti Remote".

- [ ] **Step 2: Update CompanyName (line ~92)**

Replace:
```
            VALUE "CompanyName", "Purslane Ltd" "\0"
```
With:
```
            VALUE "CompanyName", "Liteti Tecnologia" "\0"
```

- [ ] **Step 3: Update LegalCopyright (line ~96)**

Replace:
```
            VALUE "LegalCopyright", "Copyright © 2025 Purslane Ltd. All rights reserved." "\0"
```
With:
```
            VALUE "LegalCopyright", "Copyright © 2025 Liteti Tecnologia" "\0"
```

Leave `InternalName` ("rustdesk") and `OriginalFilename` ("rustdesk.exe") unchanged — they truthfully name the binary under Approach A.

- [ ] **Step 4: Verify**

Run:
```bash
grep -nE "Purslane" flutter/windows/runner/Runner.rc
```
Expected: **no output**.

- [ ] **Step 5: Commit**

```bash
git add flutter/windows/runner/Runner.rc
git commit -m "fix(branding): set Liteti company/copyright in Windows Runner.rc"
```

---

### Task 3: Brand `libs/portable/Cargo.toml` winres (installer `.exe` properties)

These winres values become the file properties of `liteti-remote-{version}-install.exe`, the file the end user double-clicks.

**Files:**
- Modify: `libs/portable/Cargo.toml` (`[package.metadata.winres]`, ~lines 30–32; optional package `description` line ~5)

- [ ] **Step 1: Verify current state**

Run:
```bash
grep -nE "^name|^description|ProductName|OriginalFilename|FileDescription" libs/portable/Cargo.toml
```
Expected: `name = "rustdesk-portable-packer"`, `description`/`ProductName`/`FileDescription` = RustDesk, `OriginalFilename = "rustdesk.exe"`.

- [ ] **Step 2: Update the winres ProductName (line ~30)**

Replace:
```toml
ProductName = "RustDesk"
```
With:
```toml
ProductName = "Liteti Remote"
```

- [ ] **Step 3: Update OriginalFilename (line ~31)**

Replace:
```toml
OriginalFilename = "rustdesk.exe"
```
With:
```toml
OriginalFilename = "liteti-remote-install.exe"
```

- [ ] **Step 4: Update FileDescription (line ~32)**

Replace:
```toml
FileDescription = "RustDesk Remote Desktop"
```
With:
```toml
FileDescription = "Liteti Remote - Acesso Remoto Seguro"
```

Leave `[package] name = "rustdesk-portable-packer"` unchanged — `build.py` (Task 1) now expects the `rustdesk-portable-packer.exe` output.

- [ ] **Step 5: Verify the packer package name is intact and brand strings updated**

Run:
```bash
grep -nE "rustdesk-portable-packer|Liteti Remote|liteti-remote-install\.exe" libs/portable/Cargo.toml
```
Expected: `name = "rustdesk-portable-packer"` still present; `ProductName`/`FileDescription` Liteti; `OriginalFilename = "liteti-remote-install.exe"`.

- [ ] **Step 6: Commit**

```bash
git add libs/portable/Cargo.toml
git commit -m "fix(branding): set Liteti winres props on portable installer packer"
```

---

### Task 4: Static name-consistency verification (build.py ↔ sources)

Confirms every name the Windows `build.py` path expects has a real producer in the sources. This is spec validation step 1.

**Files:**
- None modified (verification only)

- [ ] **Step 1: Assert the lib name matches**

Run:
```bash
grep -nE "^name" Cargo.toml | head -3
grep -n "librustdesk.dll" build.py
```
Expected: `Cargo.toml [lib] name = "librustdesk"` ⇒ produces `librustdesk.dll`; `build.py` checks `librustdesk.dll`. Match.

- [ ] **Step 2: Assert the runner exe name matches**

Run:
```bash
grep -nE "BINARY_NAME" flutter/windows/CMakeLists.txt
grep -n "/rustdesk.exe'" build.py
```
Expected: `BINARY_NAME "rustdesk"` ⇒ produces `rustdesk.exe`; `build.py` passes `rustdesk.exe`. Match.

- [ ] **Step 3: Assert the packer exe name matches**

Run:
```bash
grep -nE "^name" libs/portable/Cargo.toml | head -1
grep -n "rustdesk-portable-packer.exe" build.py
```
Expected: packer package `name = "rustdesk-portable-packer"` ⇒ produces `rustdesk-portable-packer.exe`; `build.py` references it. Match.

- [ ] **Step 4: Confirm the delivered artifact stays Liteti**

Run:
```bash
grep -nE "liteti-remote-%s-install|liteti-remote-.*install\.exe" build.py
```
Expected: the final `os.rename(... f'./liteti-remote-{version}-install.exe')` is present.

If any of Steps 1–4 do not match, fix the corresponding source/`build.py` reference before proceeding — the Windows build will fail otherwise.

---

### Task 5: Functional build on Windows (manual handoff)

Cannot run on the Linux dev host. The user runs this on a Windows machine with the toolchain (Flutter, MSVC, vcpkg `x64-windows-static`, Rust).

**Files:**
- None (produces `liteti-remote-{version}-install.exe`)

- [ ] **Step 1: Sync the branch on the Windows machine**

```bash
git fetch origin
git checkout feature/liteti-branding
git pull
git submodule update --init --recursive
```

- [ ] **Step 2: Run the local Windows build**

```bash
python3 build.py --flutter --portable --hwcodec --vram
```
(Drop `--hwcodec`/`--vram` if those toolchains are not set up; the name fixes are independent of those features.)

Expected: build proceeds past the `librustdesk.dll` check, `flutter build windows --release` succeeds, the portable packer runs, and the run ends with:
```
output location: .../liteti-remote-{version}-install.exe
```

- [ ] **Step 3: Smoke-check the artifact**

- Confirm `liteti-remote-{version}-install.exe` exists in the repo root.
- Right-click → Properties → Details: `ProductName` = "Liteti Remote", `FileDescription` = "Liteti Remote - Acesso Remoto Seguro".
- Run the installer; confirm the app launches and the relay/ID server is the Liteti one.

- [ ] **Step 4: Report back**

Paste any build error here. Common expected follow-ups if errors appear: a name still referencing `liteti*` that a source produces as `rustdesk*` (re-run Task 4 greps), or a missing toolchain dependency (vcpkg triplet, Flutter Windows engine).

---

## Self-Review

**Spec coverage:**
- Blocker 1 (DLL) → Task 1 Step 2. ✓
- Blocker 2 (runner exe) → Task 1 Step 3. ✓
- Blocker 3 (packer exe) → Task 1 Step 4. ✓
- Runner.rc branding → Task 2. ✓
- libs/portable winres branding → Task 3. ✓
- Validation 1 (static) → Task 4. ✓
- Validation 2 (real Windows build) → Task 5. ✓
- "Deliberately unchanged" (BINARY_NAME, APP_PREFIX, handshake) → not modified by any task; Task 4 Step 2 asserts `BINARY_NAME` stays `rustdesk`. ✓
- Out-of-scope items (CI, MSI, signing, ARM64) → no tasks, correct. ✓
- Follow-ups (build.py:429 strip, :411 macOS) → intentionally excluded per approval. ✓

**Placeholder scan:** No TBD/TODO; every edit shows exact before/after text and exact verification commands. ✓

**Type/name consistency:** `librustdesk.dll`, `rustdesk.exe`, `rustdesk-portable-packer.exe`, `liteti-remote_portable.exe`, `liteti-remote-{version}-install.exe`, `rustdesk-portable-packer` (package) used identically across Tasks 1, 3, and 4. ✓
