# Autodesk Inventor 2026 on Linux

**Status: WORKING** — Full UI, Part/Assembly/Drawing creation, materials, CPU ray tracing.

Autodesk Inventor 2026 Professional running under Wine 11.4 on Linux. WineHQ rates every version of Inventor as "Garbage". This project demonstrates it can be done, documents every fix in enough detail to reproduce, and provides scripts to rebuild the environment from scratch in ~15 minutes.

This is believed to be the first instance of Inventor running on Linux.

** UPDATE: Harvi564 has forked this and is closing in quickly on a full inventor experience! Go check their project here: https://github.com/Harvi654/Autodesk-Inventor-for-Linux/tree/main **

---

## Table of Contents

- [What Works](#what-works)
- [Requirements](#requirements)
- [Quick Start](#quick-start)
- [Architecture](#architecture)
- [Setup Guide](#setup-guide)
  - [1. Build the Wine Prefix](#1-build-the-wine-prefix)
  - [2. Wine wbemprox Patch](#2-wine-wbemprox-patch)
  - [3. OAuth2 Protocol Handler](#3-oauth2-protocol-handler)
  - [4. Desktop Entry](#4-desktop-entry-optional)
  - [5. Drive Mappings](#5-drive-mappings)
  - [6. Content Center](#6-content-center)
- [Technical Details](#technical-details)
  - [OGSFactory.dll Binary Patch](#ogsfactorydll-binary-patch)
  - [Wine wbemprox / Win32_TimeZone](#wine-wbemprox--win32_timezone)
  - [Key Discoveries](#key-discoveries)
- [Graphics Settings](#graphics-settings)
- [Open Problems](#open-problems)
- [Debug Reference](#debug-reference)
- [Never Do These Things](#never-do-these-things)
- [Session History](#session-history)
- [Contributing](docs/contributing.md)
- [License](#license)

---

## What Works

| Feature | Status | Notes |
|---------|--------|-------|
| Full ribbon UI | ✅ Working | File, Tools, Customize, Macros, VBA Editor, Add-Ins |
| Part / Assembly / Drawing creation | ✅ Working | Standard modelling operations |
| Materials | ✅ Working | Slightly darker than Windows (OGS patch side-effect) |
| CPU ray tracing | ✅ Working | Real non-flat results; requires wbemprox patch |
| Autodesk SSO login | ✅ Working | Firefox → OAuth2 → Identity Manager |
| Licensing | ✅ Working | Token cached; survives restarts |
| D3D11 hardware acceleration | ✅ Working | DXVK 2.7.1, Feature Level 12.1 |
| WebView2 | ✅ Working | Home page, login UI |
| Content Center | ✅ Working | 7.1 GB standard hardware libraries |
| ClearType font antialiasing | ✅ Working | |
| Realistic Visual Style | ❌ Broken | "Material Library not available" |
| GPU ray tracing | ❌ Broken | Requires D3D12/DXR (VKD3D-Proton) |
| iLogic | ❌ Broken | COM interop E_FAIL on DesignProjectManager |
| Material brightness | ⚠️ Degraded | OGS patch skips lighting init; faces render darker |
| GPU identity (DXVK) | ⚠️ Cosmetic | Reports AMD RX 6700 XT regardless of actual GPU |

---

## Requirements

**Hardware:**
- x86\_64 CPU
- GPU with Vulkan/D3D11 support (tested: NVIDIA RTX 4080 + Intel Arc Meteor Lake)

**Software:**
- Linux, kernel 6.x+ (tested: CachyOS / Arch)
- Wine 11.4 (vanilla — not Proton)
- DXVK 2.7.1
- `winetricks`
- ImageMagick (`convert`) — for icon extraction
- A working Autodesk Inventor 2026 Professional installation on Windows, accessible at `/mnt/windows/`

**Disk space:**
- ~20 GB — Wine prefix + Inventor binaries
- ~7 GB — Content Center (optional)

**Account:**
- Valid Autodesk account with Inventor 2026 subscription (SSO required on first launch; subsequent launches use cached token)

---

## Quick Start

```bash
# Install system prerequisites (CachyOS/Arch)
sudo bash scripts/phase0-setup.sh

# Export Autodesk registry keys from your Windows installation
bash scripts/export-registry.sh

# Build the Wine prefix (~15 min; requires Windows partition mounted at /mnt/windows)
bash scripts/rebuild-prefix.sh

# Apply the system-wide Wine wbemprox patch (required for CPU ray tracing)
# See: Setup Guide → Wine wbemprox Patch

# Register the OAuth2 protocol handler (required for first-time login)
# See: Setup Guide → OAuth2 Protocol Handler

# Launch
bash scripts/launch-inventor.sh
```

On first launch Inventor opens a Firefox window for Autodesk SSO login. After login the license is cached and subsequent launches skip the browser entirely.

---

## Architecture

Inventor requires three Windows services running in order inside the Wine prefix:

```
┌─────────────────────────────────────────────────────────────────┐
│  Wine Prefix (~/.wine-inventor2026)                              │
│                                                                  │
│  1. AdskIdentityManager.exe  ←──►  Firefox (OAuth2 login)       │
│            │                              ↓                      │
│            │                    adskidmgr:// callback            │
│            │                    (Linux xdg-mime handler)         │
│            ↓                                                     │
│  2. AdskLicensingService.exe  ←──►  .asr license cache          │
│            │                                                     │
│            ↓                                                     │
│  3. Inventor.exe                                                 │
│            ├── OGSFactory.dll  (D3D11 / DXVK 3D viewport)       │
│            ├── ClrAddinLoader.dll  →  .NET 8 CoreCLR            │
│            └── WebView2  (home page, login UI)                   │
└─────────────────────────────────────────────────────────────────┘
```

**Wine prefix:** `~/.wine-inventor2026`

**Drive mappings:**
| Drive | Path |
|-------|------|
| `C:` | `~/.wine-inventor2026/drive_c` |
| `D:` | `/mnt/Storage` (user data) |
| `Z:` | `/` (Linux root) |

---

## Setup Guide

### 1. Build the Wine Prefix

`scripts/rebuild-prefix.sh` copies the Inventor installation from `/mnt/windows/` into a fresh Wine prefix and applies all known fixes. This is the authoritative setup procedure.

**What it does (28 steps):**
1. Wipes and recreates the Wine prefix
2. Sets Windows 10 compatibility mode
3. Installs DXVK (D3D9/D3D11 → Vulkan translation)
4. Copies Inventor binaries, ASC SharedComponents, MSVC/MFC runtimes, RealDWG
5. Installs WebView2 runtime, Identity Manager, Licensing Service, .NET 8
6. Imports registry keys: Inventor, SharedComponents, licensing, protocol handler
7. Installs IDSDK SSO plugin into the Licensing Agent (`IDSDKVersionCompatibility.config`)
8. Disables non-essential and Wine-broken addins (see `notes/disabled-addins.md`)
9. Moves `InteractiveTutorial.dll` — triggers wine-mono fatal assertion if left in place
10. Disables CER crash reporter (`senddmp.exe`) — causes false-positive process termination
11. Removes `bcp47langs.dll` stubs — Proton placeholder causes NULL crash in all Autodesk apps
12. Copies Design Data, Templates, Textures
13. **Applies OGSFactory.dll binary patch** (§[OGSFactory.dll Binary Patch](#ogsfactorydll-binary-patch))

```bash
bash scripts/rebuild-prefix.sh
```

> Requires `/mnt/windows/` to be mounted with the source Windows installation.

---

### 2. Wine wbemprox Patch

Wine's WMI implementation (`wbemprox.dll`) is missing `Win32_TimeZone`. OGSFactory.dll queries this class during 3D viewport initialisation; without it, the rendering pipeline initialises in a degraded state and CPU ray tracing produces flat results.

This patch modifies the system-wide Wine installation and must be reapplied after any Wine upgrade.

**Build and install:**

```bash
cd /tmp
curl -LO https://dl.winehq.org/wine/source/11.x/wine-11.4.tar.xz
tar xf wine-11.4.tar.xz && cd wine-11.4

patch -p1 < ~/Projects/Inventor\ on\ linux/patches/wbemprox-timezone.patch

./configure --enable-win64 --without-x --without-freetype
make -C dlls/wbemprox

sudo cp dlls/wbemprox/x86_64-windows/wbemprox.dll \
    /usr/lib/wine/x86_64-windows/wbemprox.dll
```

**Verify:**
```bash
wine cscript //NoLogo Z:\\tmp\\test_tz.vbs
# Expected output: Win32_TimeZone: found 1 items
```

> `Win32_TimeZone` is a standard WMI class used by many applications. This patch is a strong candidate for upstreaming to Wine.

---

### 3. OAuth2 Protocol Handler

The `adskidmgr://` URI scheme passes the OAuth2 authorization code from Firefox back to the running Identity Manager server. Register the Linux handler:

```bash
# Create the .desktop handler
cat > ~/.local/share/applications/adskidmgr-handler.desktop << EOF
[Desktop Entry]
Type=Application
Name=Autodesk Identity Manager
Exec=$HOME/.local/bin/adskidmgr-callback %u
MimeType=x-scheme-handler/adskidmgr;
NoDisplay=true
EOF

# Symlink the callback script
ln -sf ~/Projects/Inventor\ on\ linux/scripts/adskidmgr-callback.sh \
    ~/.local/bin/adskidmgr-callback

# Register
xdg-mime default adskidmgr-handler.desktop x-scheme-handler/adskidmgr
update-desktop-database ~/.local/share/applications
```

The callback script (`scripts/adskidmgr-callback.sh`) creates the `AdOAuth2Code` IPC file then launches a client Identity Manager instance to deliver the auth code. It must **not** kill `wineserver` — that would destroy the running Inventor session.

---

### 4. Desktop Entry (optional)

```bash
# Extract icon from Inventor binary
convert \
    ~/.wine-inventor2026/drive_c/Program\ Files/Autodesk/Inventor\ 2026/Bin/idv.ico \
    -thumbnail 256x256 -flatten \
    ~/.local/share/icons/hicolor/256x256/apps/autodesk-inventor-2026.png

# Symlink for a spaces-free executable path
ln -sf ~/Projects/Inventor\ on\ linux/scripts/launch-inventor.sh \
    ~/.local/bin/launch-inventor

# Create the .desktop entry
cat > ~/.local/share/applications/autodesk-inventor-2026.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Autodesk Inventor 2026
Exec=launch-inventor
Icon=autodesk-inventor-2026
Categories=Engineering;Graphics;
EOF

update-desktop-database ~/.local/share/applications
```

---

### 5. Drive Mappings

```bash
# Map your data drive (adjust path as needed)
ln -sf /mnt/Storage ~/.wine-inventor2026/dosdevices/d:
```

---

### 6. Content Center

Required for standard hardware components (bolts, bearings, etc.) in assemblies. 7.1 GB.

```bash
cp -r "/mnt/windows/ProgramData/Autodesk/Inventor 2026/Content Center" \
    "$HOME/.wine-inventor2026/drive_c/ProgramData/Autodesk/Inventor 2026/"
```

---

## Technical Details

### OGSFactory.dll Binary Patch

#### Root Cause

OGSFactory.dll queries WMI during 3D viewport creation. The function at RVA `0x57968`:

1. Calls `IWbemServices::CreateInstanceEnum()` (vtable slot 18) for `Win32_TimeZone`
2. Calls `IEnumWbemClassObject::Next(2000, 1, &ppObject, &uReturned)` (vtable slot 4)
3. Uses the returned object to set up the Protein material/lighting pipeline

Under Wine, `Next()` returns `S_OK` but leaves `ppObject = NULL`. The function then unconditionally dereferences `[ppObject]`, causing access violation 0xC0000005.

The crash occurs inside a Win32 window message callback. Wine's `user_callback_handler` logs `"ignoring exception c0000005"` and continues with corrupted state. On Windows, `legacyCorruptedStateExceptionsPolicy` would catch this. Under Wine it is fatal.

#### The Fix

Two code-cave patches redirect the crash sites to null-check trampolines written into `.text` section padding:

**Patch 1** — file offset `0x56E5B` (RVA `0x57A5B`):

```asm
; Original (crashes when rcx = NULL):
;   mov  rcx, [rsp+0x70]
;   mov  rdx, [rcx]         ← access violation
;
; Replaced with: jmp to code cave at 0x110A60
e9 00 9c 0b 00   ; jmp  +0xB9C00
90 90 90         ; nop  nop nop  (alignment padding)

; Code cave at file offset 0x110A60:
48 8b 4c 24 70   ; mov  rcx, [rsp+0x70]
48 85 c9         ; test rcx, rcx
0f 84 11 64 f4 ff; je   0x57A7F          ; skip to cleanup if NULL
48 8b 11         ; mov  rdx, [rcx]       ; safe dereference
e9 ed 63 f4 ff   ; jmp  0x57A63         ; return to normal flow
```

**Patch 2** — file offset `0x56EA8` (RVA `0x57AA8`): same pattern, second dereference of the same pointer. Code cave at `0x110A76`.

**Apply (already integrated into rebuild-prefix.sh step 27):**

```bash
DLLPATH="$HOME/.wine-inventor2026/drive_c/Program Files/Autodesk/Inventor 2026/Bin/OGSFactory.dll"
cp "$DLLPATH" "${DLLPATH}.original"

# Patch 1: redirect crash site
printf '\xe9\x00\x9c\x0b\x00\x90\x90\x90' \
    | dd of="$DLLPATH" bs=1 seek=$((0x56E5B)) conv=notrunc 2>/dev/null
# Patch 1: null-check trampoline
printf '\x48\x8b\x4c\x24\x70\x48\x85\xc9\x0f\x84\x11\x64\xf4\xff\x48\x8b\x11\xe9\xed\x63\xf4\xff' \
    | dd of="$DLLPATH" bs=1 seek=$((0x110A60)) conv=notrunc 2>/dev/null

# Patch 2: redirect crash site
printf '\xe9\xc9\x9b\x0b\x00\x90\x90\x90' \
    | dd of="$DLLPATH" bs=1 seek=$((0x56EA8)) conv=notrunc 2>/dev/null
# Patch 2: null-check trampoline
printf '\x48\x8b\x4c\x24\x70\x48\x85\xc9\x0f\x84\x48\x64\xf4\xff\x48\x8b\x11\xe9\x24\x64\xf4\xff' \
    | dd of="$DLLPATH" bs=1 seek=$((0x110A76)) conv=notrunc 2>/dev/null
```

#### Side Effects

The trampolines skip the Protein material/lighting pipeline initialisation:
- Materials render slightly darker than on Windows
- Realistic Visual Style reports "Material Library not available"
- GPU ray tracing capability detection may be affected

Fixing these requires identifying and implementing the Wine interface that currently returns NULL. See §[Open Problems](#open-problems).

---

### Wine wbemprox / Win32_TimeZone

`patches/wbemprox-timezone.patch` adds a complete `Win32_TimeZone` implementation to Wine's `dlls/wbemprox/builtin.c`:

| Addition | Description |
|----------|-------------|
| `col_timezone[]` | 21-property column definition matching the MSDN `Win32_TimeZone` schema |
| `struct record_timezone` | Record layout matching the column definition |
| `fill_timezone()` | Calls `GetTimeZoneInformation()` and populates all fields |
| Table entry | Registered in `cimv2_builtin_classes[]` alphabetically before `Win32_VideoController` |

See `patches/wbemprox-timezone.patch` for the complete diff.

---

### Key Discoveries

Non-obvious facts found during the porting work. Read these before investigating any open issue.

**1. WPF GAC removal is mandatory.**
Wine-mono ships stub WPF DLLs in the GAC (`PresentationCore`, `PresentationFramework`, `WindowsBase`, `System.Xaml`). When Inventor loads, mono hits a fatal assertion in the stub and calls `System.Environment.Exit()`. Deleting only the DLL files (not the directories) is the fix. `wineboot -u` recreates directories but not files, so the fix persists.

**2. bcp47langs.dll Proton stub causes a NULL crash.**
Proton ships a stub `bcp47langs.dll` that returns NULL from all calls. This crashes all Autodesk apps. Remove the file or disable the DLL override entirely.

**3. Identity Manager uses Boost.Interprocess for IPC.**
The IDSDK IPC directory (`$WINEPREFIX/drive_c/ProgramData/Autodesk/IDSDK/<USERNAME_HEX>/interprocess/01000000/`) must be wiped before each launch. Stale lock files cause Identity Manager to deadlock indefinitely. The directory name is the ASCII username encoded as uppercase hex (e.g. `alice` → `616C696365`); the launch script computes this dynamically.

**4. OAuth2 callback: launch a client, never kill wineserver.**
The `adskidmgr://` handler must spawn a new Identity Manager *client* instance (which passes the auth code and exits). Killing `wineserver` destroys the running Inventor session.

**5. CER crash reporter terminates the process.**
Autodesk's `senddmp.exe` intercepts native crashes, shows a report dialog, then exits the process. Disabling it by renaming allows Inventor to survive recoverable crashes.

**6. .NET 8 CoreCLR runs under Wine 11.4.**
Load chain: `ClrAddinLoader.dll` → `hostfxr.dll` → `hostpolicy.dll` → `coreclr.dll`. Use `WINEDLLOVERRIDES="msquic="` to suppress MSQUIC networking crashes.

**7. Wine's user_callback_handler swallows exceptions.**
A native crash inside a Win32 window message callback logs `"ignoring exception c0000005"` and continues. On Windows the exception propagates to the managed handler. Under Wine it is silently absorbed, leaving corrupted state that eventually terminates the process.

**8. DXVK always reports AMD Radeon RX 6700 XT.**
Regardless of actual GPU hardware. May affect OGS GPU capability detection but has no functional impact on D3D11 rendering.

**9. IDSDK SSO plugin must be manually placed in the Licensing Agent.**
The `IDSDKVersionCompatibility.config` file and `SSOPlugin/Current/*.dll` must exist under the Licensing Agent directory. They are not discovered from the Identity Manager installation path.

**10. OGS queries three WMI classes during graphics init.**
`Win32_TimeZone`, `Win32_VideoController`, and `Win32_BIOS` are queried from `OGSFactory.dll!fn@0x57968`. Missing classes cause silent pipeline degradation or a crash.

---

## Graphics Settings

Stored in `user.cfg` inside the Wine prefix. **Do not change these unless you understand the consequences** — certain combinations produce black or greyscale rendering.

```ini
UseINVBasicMaterial = 1    ; basic material system — working
UsePrismMaterial    = 0    ; PBR — causes issues when enabled alone
UseUberMaterial     = 0    ; advanced materials — causes black faces
GammaCorrect        = 0    ; gamma correction — causes inverted brightness
DMEffectType        = 0x80 ; shading mode — changing to 0 breaks face colours
OGSCon              = 2    ; D3D11 mode
UseAccel            = 1    ; hardware acceleration
UseGPURaytracing    = 1    ; GPU RT — greyed out until D3D12 works
UseRRT              = 1    ; RapidRT engine
```

---

## Open Problems

Known issues with investigation leads. Each entry has enough context for a skilled developer to pick it up cold.

### 1. Materials Too Dark (highest priority)

**Symptom:** Material faces render noticeably darker than on Windows.

**Root cause:** The binary patch skips the portion of `OGSFactory.dll!fn@0x57968` that initialises the Protein material/lighting pipeline, because the COM call at RVA `0x57A40` returns `ppObject = NULL`.

**Investigation path:**
1. Use `OGSFactory.dll.original` (pre-patch) for analysis — it crashes but reveals the real call sequence
2. Set a `winedbg` breakpoint at RVA `0x57A40` (the `IEnumWbemClassObject::Next()` call)
3. Inspect `rax`/`rcx`/`rdx` after the call to identify the IID and vtable slot
4. Search Wine source for that IID — implement it if missing, or fix the return value if stubbed incorrectly
5. If the interface is a sub-query on the `Win32_TimeZone` instance, the fix belongs in `wbemprox`

**Fixing this will likely also fix:** Realistic Visual Style ("Material Library not available")

---

### 2. iLogic (COM Interop E_FAIL)

**Symptom:** iLogic addin fails with `E_FAIL 0x80004005` on `DesignProjectManager.get_ActiveDesignProject`.

**Investigation path:**
1. Re-enable the addin (move `autodesk.ilogic.inventor.addin` from `Bin/Addins/disabled/` back to `Bin/Addins/`)
2. Run with `WINEDEBUG=+ole,+rpc,err+all` and capture the COM trace
3. The failure is on a .NET → COM boundary via `ClrAddinLoader`
4. Check whether Wine's `ole32`/`oleaut32` correctly implements the specific `IDispatch` interface used by iLogic
5. May be a .NET 8 / wine-mono interop issue at the `IDispatch` marshalling layer

---

### 3. GPU Ray Tracing

**Symptom:** GPU ray tracing option is greyed out in Inventor's render settings.

**Root cause:** Requires D3D12 + DXR (DirectX Raytracing). Wine does not ship VKD3D-Proton by default.

**Fix path:** Install VKD3D-Proton, add `d3d12 = native` to DLL overrides, and test whether OGS detects the D3D12 device and enables the GPU RT path.

**Risk:** VKD3D-Proton and DXVK coexistence for D3D11/D3D12 needs testing.

---

### 4. Identity Manager Intermittent Deadlock

**Symptom:** Occasionally Identity Manager starts but never reaches "SSO Server is ready". Hangs in Boost.Interprocess lock acquisition.

**Current mitigation:** The launch script wipes the IPC directory before each start. This prevents the issue most of the time.

**Remaining edge case:** If the system is hard-killed while Identity Manager is running, stale `IDSDKQuit-v2-*` files from a different session path may survive and block the next start.

**Fix path:** Wipe all subdirectories under `.../IDSDK/<USERNAME_HEX>/interprocess/` before launch, not just the single session directory.

---

## Debug Reference

### Environment Variables

```bash
export WINEPREFIX="$HOME/.wine-inventor2026"
export WINEARCH=win64

# Logging levels
export WINEDEBUG=err+all,fixme-all              # normal operation
export WINEDEBUG=+loaddll,err+all,fixme-all     # trace DLL loading
export WINEDEBUG=+seh,err+all,fixme-all         # trace exception handling
export WINEDEBUG=+ole,+rpc,err+all,fixme-all    # trace COM calls
export WINEDEBUG=+wbemprox,err+all              # trace WMI queries

export DXVK_LOG_LEVEL=info
export DXVK_ASYNC=1
export WINE_LARGE_ADDRESS_AWARE=1
export WINEDLLOVERRIDES="msquic="               # prevent MSQUIC crash in .NET 8
```

### Useful Commands

```bash
# Is Inventor running?
ps aux | grep Inventor.exe | grep -v grep

# Follow Identity Manager log
tail -f ~/.wine-inventor2026/drive_c/users/$USER/AppData/Local/Autodesk/\
"Identity Services/Log/IdServices.log"

# Follow licensing log
tail -f ~/.wine-inventor2026/drive_c/users/$USER/AppData/Local/Autodesk/\
"Inventor 2026/Logs/AutodeskInventor2026License*.log"

# Trace DLL loading (find which DLL crashes on startup)
WINEDEBUG=+loaddll,err+all wine Inventor.exe 2>&1 | grep "loaddll.*Loaded"

# Verify Win32_TimeZone patch
wine cscript //NoLogo Z:\\tmp\\test_tz.vbs

# Kill everything (emergency)
WINEPREFIX=~/.wine-inventor2026 wineserver -k
```

### Log Files

`scripts/launch-inventor.sh` writes timestamped logs to `logs/` (gitignored):
- `logs/inventor-YYYYMMDD-HHMMSS.log`
- `logs/idmgr-YYYYMMDD-HHMMSS.log`
- `logs/licensing-YYYYMMDD-HHMMSS.log`

---

## Never Do These Things

These actions permanently break the Wine prefix or silently corrupt Inventor's state. There is no recovery path short of a full `rebuild-prefix.sh` run.

| Action | Consequence |
|--------|-------------|
| `winetricks dotnet48` | Removes wine-mono; all .NET loading breaks permanently |
| `mscoree = native` DLL override | Prevents PE loading entirely |
| Copy Windows .NET assemblies into wine-mono GAC | Triggers mono fatal assertion on every startup |
| Kill `wineserver` in OAuth callback | Destroys the running Inventor session |
| Place Proton `bcp47langs.dll` in system32 | NULL crash in all Autodesk apps |
| `UseUberMaterial=1` or `GammaCorrect=1` | Black / inverted face rendering |
| Update Wine without rebuilding wbemprox | Loses `Win32_TimeZone`; breaks CPU ray tracing |
| Leave IDSDK IPC directory dirty across restarts | Identity Manager deadlock on next launch |

---

## Session History

**2026-03-20 — Session 1:** First successful Inventor launch. WPF GAC removal discovered as the enabling fix for .NET loading. Identity Manager SSO working via Firefox + OAuth2. Inventor ran 4+ minutes at 242 MB RSS. Identity Manager intermittent deadlock identified as a Boost.Interprocess stale lock issue.

**2026-03-20 — Session 2:** Identity Manager crash traced to `bcp47langs.dll` Proton stub. Full ribbon UI achieved. OGS crash on Part creation traced to `OGSFactory.dll` null pointer via module load tracing and disassembly. Binary code cave patch developed and applied. Design Data, Templates, Textures copied. **Part creation working.**

**2026-03-21 — Session 3:** Wine wbemprox patched to add `Win32_TimeZone` WMI class (built from Wine 11.4 source). CPU ray tracing produces real non-flat results. Shared Materials Library (535 MB), Backgrounds, Home page, Content Center (7.1 GB), and full Autodesk registry imported. Default `.ipj` project file configured with library search paths. Desktop entry and icon created. Drive D: mapped to `/mnt/Storage`. ClearType font AA enabled. **Materials, assemblies, and CPU ray tracing working.**

---

## Contributing

See [docs/contributing.md](docs/contributing.md) for development environment setup, debugging techniques, and guidance on the open problems above.

---

## License

Scripts and documentation are provided as-is under the MIT License for educational and interoperability purposes.

Autodesk Inventor is proprietary software requiring a valid license. This project does not circumvent copy protection — it uses Autodesk's legitimate SSO authentication flow.
