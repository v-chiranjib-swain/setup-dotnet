# Investigation: `actions/setup-dotnet` Issue #387 — Windows `dotnet.exe` File Lock

## Table of Contents

1. [Background](#1-background)
2. [How `setup-dotnet@v4` Works Internally](#2-how-setup-dotnetv4-works-internally)
3. [The `dotnet` Muxer — What It Is](#3-the-dotnet-muxer--what-it-is)
4. [Root Cause of Issue #387](#4-root-cause-of-issue-387)
5. [Why Windows Failed, Linux/macOS Did Not](#5-why-windows-failed-linuxmacos-did-not)
6. [The v4 Fix — Two-Pass Design (PR #433)](#6-the-v4-fix--two-pass-design-pr-433)
7. [Why the Latest Active LTS Runtime Is Installed First](#7-why-the-latest-active-lts-runtime-is-installed-first)
8. [Bug Replication](#8-bug-replication)
9. [Cross-Platform Verification](#9-cross-platform-verification)
10. [Alternatives That Were Considered (and Why They Don't Work)](#10-alternatives-that-were-considered-and-why-they-dont-work)
11. [Cumulative Answer](#11-cumulative-answer)
12. [Selecting SDK and Runtime Dynamically Without Hardcoding](#12-selecting-sdk-and-runtime-dynamically-without-hardcoding)

---

## 1. Background

**Issue:** [actions/setup-dotnet#387](https://github.com/actions/setup-dotnet/issues/387)  
**Fix:** [PR #433 — Sequential version install fix](https://github.com/actions/setup-dotnet/pull/433)  
**Introduced in:** `actions/setup-dotnet@v4`

When `setup-dotnet@v3` was called more than once in the same job — to install multiple SDK versions — the second call failed on Windows with:

```
The process cannot access the file 'dotnet.exe'
because it is being used by another process.
```

---

## 2. How `setup-dotnet@v4` Works Internally

### Entry point — `src/setup-dotnet.ts`

For each version in `dotnet-version`, a `DotnetCoreInstaller` is constructed and `installDotnet()` is called.

### Version resolution — `DotnetVersionResolver.createDotnetVersion()`

| Input format | `semver.valid()` | Result |
|---|---|---|
| `9.0.203` | `true` | `type='--version'`, `value='9.0.203'` |
| `9.0.x` | `false` | `type='--channel'`, `value='9.0'` |
| `9` | `false` | `type='--channel'`, `value='9.0'` |

### Two-pass install — `DotnetCoreInstaller.installDotnet()`

```
Pass 1:  install-dotnet.sh --skip-non-versioned-files
                           --runtime dotnet
                           --channel LTS
         → Downloads latest LTS runtime (e.g., 10.0.10)
         → Writes dotnet muxer (first time only)
         → Writes shared/Microsoft.NETCore.App/10.0.10/ (versioned)

Pass 2:  install-dotnet.sh --skip-non-versioned-files
                           --version 9.0.203
         → Downloads SDK 9.0.203
         → Skips dotnet muxer (already exists from Pass 1)
         → Writes sdk/9.0.203/ (versioned)
         → Writes shared/Microsoft.NETCore.App/9.0.4/ (versioned)
```

### Final disk layout (ubuntu-latest, dotnet-version: 9.0.203)

```
/usr/share/dotnet/
├── dotnet                                  ← muxer from LTS 10.0.10
├── host/fxr/
│   ├── 10.0.10/
│   └── 9.0.4/
├── shared/Microsoft.NETCore.App/
│   ├── 10.0.10/
│   └── 9.0.4/
└── sdk/
    └── 9.0.203/
```

### PATH and environment

After all installs, `DotnetInstallDir.addToPath()` is called once:

```typescript
core.addPath(process.env['DOTNET_INSTALL_DIR']!)          // adds to PATH
core.exportVariable('DOTNET_ROOT', process.env['DOTNET_INSTALL_DIR'])
```

`DOTNET_INSTALL_DIR` defaults per platform:

| Platform | Default path |
|---|---|
| Linux | `/usr/share/dotnet` |
| macOS | `~/.dotnet` |
| Windows | `%PROGRAMFILES%\dotnet` |

---

## 3. The `dotnet` Muxer — What It Is

`dotnet` / `dotnet.exe` is called the **muxer** (multiplexer). It is a small (~300 KB) launcher that:

1. Reads `global.json` in the current directory
2. Resolves which SDK version to activate
3. Delegates execution to the appropriate SDK

```
dotnet build
  └─→ muxer reads global.json
        └─→ "use sdk/9.0.203/"
              └─→ loads /usr/share/dotnet/sdk/9.0.203/
```

The muxer is **shared** across every installed SDK. All versioned SDK and runtime content lives in separate subdirectories. The muxer itself is not tied to any specific version directory — which is why the install script classifies it as a "non-versioned file".

> **Important:** "Non-versioned" means the file is not inside a versioned subdirectory (`sdk/X.Y.Z/`, `shared/X.Y.Z/`). It does **not** mean the binary itself never changes — the muxer does receive security patches and feature updates with each .NET release.

---

## 4. Root Cause of Issue #387

### Why v3 overwrote `dotnet.exe`

Every SDK/runtime archive Microsoft ships bundles `dotnet`/`dotnet.exe` inside it alongside the versioned content. The archive is designed to be self-contained for a fresh installation:

```
sdk-9.0.203.tar.gz
├── dotnet                     ← muxer (shared, non-versioned)
├── host/fxr/9.0.4/
├── shared/Microsoft.NETCore.App/9.0.4/
└── sdk/9.0.203/
```

In v3, `setup-dotnet` called the install script **without** `-SkipNonVersionedFiles`:

```
install-dotnet.ps1 -Version 9.0.203
→ Extract full archive → overwrite everything including dotnet.exe
```

The `-SkipNonVersionedFiles` flag existed in the install scripts all along but `setup-dotnet@v3` **never passed it**. v3 was designed with the assumption it would be called **once per job** into a fresh directory. When users began calling it multiple times to install several SDKs, the unconditional overwrite of the shared `dotnet.exe` caused conflicts.

### This was a design oversight

The muxer is forward/backward compatible — it can locate and launch any SDK version present in the same directory. Installing a second SDK only needs to add new versioned folders. Nothing about the muxer itself needs to change.

---

## 5. Why Windows Failed, Linux/macOS Did Not

The overwrite was unnecessary on every platform. The platforms reacted differently:

### Windows — sharing violation (mandatory enforcement)

On Windows, replacing an executable that is already open without the required sharing permissions results in a **sharing violation** at the OS kernel level. `ZipFileExtensions.ExtractToFile` throws:

```
Exception calling "ExtractToFile" with "3" argument(s):
"The process cannot access the file 'C:\...\dotnet.exe'
because it is being used by another process."
```

The installation aborts with exit code 1.

The process holding `dotnet.exe` is typically:
- Windows Defender / SmartScreen scanning the newly written file
- Windows file indexer
- The runner agent process itself

These are system services that cannot and should not be terminated.

### Linux / macOS — pathname replacement (advisory semantics)

On Linux and macOS, the filesystem allows the **pathname** to be replaced while running processes continue using the original inode. `cp` overwrites `dotnet` without error. The installation succeeds but the binary is silently replaced with an older copy from the SDK archive.

### Comparison

| Platform | Mechanism | v3 outcome |
|---|---|---|
| Windows | Mandatory sharing violation | **CRASH** — installation fails |
| Linux | Advisory, pathname replacement | Silent overwrite — no crash, unsafe |
| macOS | Advisory, pathname replacement | Silent overwrite — no crash, unsafe |

Windows did not cause the bug. It **exposed** what Linux/macOS silently allowed.

---

## 6. The v4 Fix — Two-Pass Design (PR #433)

PR #433 introduced `-SkipNonVersionedFiles` on both install passes.

### How `-SkipNonVersionedFiles` works

**In `install-dotnet.ps1` (Windows)**

```powershell
# Line 1306
$OverrideNonVersionedFiles = !$SkipNonVersionedFiles   # → $false

# Per-file check during extraction:
$OverrideFiles = $OverrideNonVersionedFiles -Or (-Not (Test-Path $DestinationPath))
#              = $false  -Or  (-Not $true)   ← dotnet.exe exists
#              = $false  → if block SKIPPED — ExtractToFile never called
```

**In `install-dotnet.sh` (Linux/macOS)**

```bash
# override = false when --skip-non-versioned-files is passed
if [ "$override" = true ] || (! ([ -e "$target" ])); then
    cp ...   # only runs if override=true OR file doesn't exist
fi
# dotnet exists → condition false → cp never called → skipped
```

In both cases: if `dotnet`/`dotnet.exe` already exists, it is **never passed to the write operation**. No lock is ever contested. The installation proceeds with only the versioned SDK files.

---

## 7. Why the Latest Active LTS Runtime Is Installed First

Simply adding `-SkipNonVersionedFiles` to the SDK install was not enough on its own. If only the user-requested SDK were installed (e.g., `8.0.100` from November 2022), the muxer written to disk would be the one **bundled at that SDK's release date** — potentially years old with known vulnerabilities.

PR #433 therefore introduced the two-pass design:

- **Pass 1** installs the latest LTS .NET Runtime with `-SkipNonVersionedFiles`
- **Pass 2** installs the user-requested SDK with `-SkipNonVersionedFiles`

This guarantees:

1. `dotnet`/`dotnet.exe` is always written **once** from the most recently patched LTS release
2. All subsequent SDK installs reuse that muxer — they skip it entirely
3. The muxer is never at risk of being an old, vulnerable binary regardless of which older SDK the user requests

From the PR author:

> *"To ensure better compatibility and to avoid vulnerability issues, LTS runtime is now installed first, providing up-to-date unversioned files (such as CLI) for further usage."*

### Why "non-versioned" doesn't mean the muxer never changes

The muxer receives:
- **Security patches** — CVEs in the muxer are fixed in runtime patch releases
- **Feature updates** — new `global.json` fields, `dotnet workload`, `dotnet sdk check`
- **Bug fixes** — SDK resolution logic improvements

A muxer from SDK `8.0.100` (Nov 2022) is ~3 years old today. The LTS pre-install ensures you always get the July 2026 version instead.

---

## 8. Bug Replication

### Why naive approaches failed

Cross-step file locks are **impossible** on GitHub Actions Windows runners because the runner creates a Windows **Job Object** per step. When each step's `pwsh` process exits, the Job Object terminates all child processes — releasing any locks they held.

| Approach | Why it failed |
|---|---|
| `Start-Process` child process | Killed by Job Object at step boundary |
| Thread runspace | Lives inside step process — dies with step |
| Two separate `setup-dotnet@v3` steps | No lock held between steps |

### Working reproduction (Run #5 — 29746822198)

The only reliable approach: hold the lock as a **local PowerShell variable within the same process** as both installs.

```powershell
# Single step — lock lives for the entire step
& "C:\dotnet-install.ps1" -Version "8.0.100" -InstallDir "C:\dotnet-test" -NoPath

$fs = [System.IO.File]::Open(
  "C:\dotnet-test\dotnet.exe",
  [System.IO.FileMode]::Open,
  [System.IO.FileAccess]::Read,
  [System.IO.FileShare]::Read
)

# v3 behavior: no -SkipNonVersionedFiles → ExtractToFile throws IOException
& "C:\dotnet-install.ps1" -Version "9.0.100" -InstallDir "C:\dotnet-test" -NoPath

$fs.Dispose()
```

**Result:** Job FAILED with exit code 1 — bug reproduced.

---

## 9. Cross-Platform Verification

**Workflow:** [v3-v4-all-platforms.yml](https://github.com/chiranjib-swain/test-setup-dotnet/blob/main/.github/workflows/v3-v4-all-platforms.yml)  
**Run:** [29914108680](https://github.com/chiranjib-swain/test-setup-dotnet/actions/runs/29914108680)

### Method

- **Windows:** Lock `dotnet.exe` with `[System.IO.File]::Open`, attempt second install without / with `-SkipNonVersionedFiles`
- **Linux/macOS:** Compare SHA-256 hash of `dotnet` before and after second install (no mandatory locks; test shows overwrite behavior)

### Results

| Job | Outcome | Evidence |
|---|---|---|
| Windows v3 | ❌ FAIL | `IOException` — sharing violation |
| Windows v4 | ✅ PASS | `dotnet.exe` skipped, hash unchanged |
| Ubuntu v3 | ✅ PASS (hash CHANGED) | `dotnet` silently overwritten |
| Ubuntu v4 | ✅ PASS (hash UNCHANGED) | `dotnet` preserved |
| macOS v3 | ✅ PASS (hash CHANGED) | `dotnet` silently overwritten |
| macOS v4 | ✅ PASS (hash UNCHANGED) | `dotnet` preserved |

---

## 10. Alternatives That Were Considered (and Why They Don't Work)

### "Kill the process holding `dotnet.exe` first"

```
Process X holds dotnet.exe → Kill X → overwrite freely
```

The holding process is typically Windows Defender, a file indexer, or the runner agent. These are system services that must not be terminated. Even if termination were possible, it would destabilize the runner.

### "Install each SDK to a separate custom directory"

```
/opt/dotnet-8/dotnet    ← SDK 8's private copy
/opt/dotnet-9/dotnet    ← SDK 9's private copy
```

The muxer is designed to serve all SDKs from a **single shared directory**. With separate directories, you need separate PATH entries, and `global.json` version resolution breaks because each muxer only sees its own directory's SDKs.

### "Run `dotnet` from an explicit custom path"

```bash
/opt/dotnet-9/dotnet build
```

Every build script, CI tool, and SDK invocation calls `dotnet` — not a versioned path. Requiring explicit paths would require changing every script in every project.

### Why all three are the wrong direction

All three try to work around the overwrite. v4 recognised the overwrite was **never necessary**:

```
Wrong question:  "How do we safely overwrite dotnet.exe?"
Right question:  "Why are we overwriting it at all?"
v4 answer:       "We don't. Write it once from LTS. Skip it forever after."
```

---

## 11. Cumulative Answer

> `setup-dotnet@v3` overwrote the `dotnet` muxer during every SDK install because the underlying install script was designed for a single extraction into a fresh directory — it bundled `dotnet` in every archive and extracted everything unconditionally. The action never passed `-SkipNonVersionedFiles` because it was originally intended to be called once. When used multiple times in a single job, it attempted to overwrite a shared binary that was already in use. This was unnecessary: the muxer is forward/backward compatible and only needs to be written once. On Windows, replacing an executable that is already open without the required sharing permissions results in a sharing violation, causing the installation to fail. On Linux and macOS, the filesystem allows the pathname to be replaced while running processes continue using the original file, so the overwrite succeeds without error. Alternative approaches — such as killing the process holding the file or installing each SDK into a separate directory — are impractical: the process holding the binary is often a system service that must not be terminated, and the muxer is explicitly designed to be shared across all SDKs in a single directory, making per-SDK isolation break `global.json` resolution and PATH management. v4 fixed this by always passing `-SkipNonVersionedFiles`, and adding a deliberate LTS runtime pre-install to ensure the muxer is always written once from the latest, most secure available version — after which every subsequent SDK install skips it entirely.

---

## 12. Selecting SDK and Runtime Dynamically Without Hardcoding

### The question

> *"How can I select which runtime to use (`--fx-version`) and which SDK without hardcoding values? Let's say the latest available ones on the image."*

---

### Background: what `--fx-version` is

`--fx-version` is a flag passed to `dotnet exec` or `dotnet run` that tells the muxer which **runtime version** to load when running an application — overriding what the application's `.runtimeconfig.json` specifies.

```bash
dotnet exec --fx-version 9.0.4 myapp.dll
# forces the app to run on runtime 9.0.4
# even if myapp.dll targets a different version
```

It is useful when multiple runtimes are installed and you want to target a specific one without modifying the project.

---

### Querying what is installed on the runner

Both commands work on all three platforms:

```bash
# List all installed SDKs
dotnet --list-sdks
# 8.0.100 [/usr/share/dotnet/sdk]
# 9.0.203 [/usr/share/dotnet/sdk]
# 10.0.301 [/usr/share/dotnet/sdk]

# List all installed runtimes
dotnet --list-runtimes
# Microsoft.AspNetCore.App 9.0.4 [/usr/share/dotnet/shared/...]
# Microsoft.NETCore.App 9.0.4   [/usr/share/dotnet/shared/...]
# Microsoft.NETCore.App 10.0.10 [/usr/share/dotnet/shared/...]
```

---

### Approach 1 — Wildcards in `setup-dotnet` (recommended)

`setup-dotnet@v4` supports version patterns that resolve dynamically at workflow runtime:

| Input | Meaning |
|---|---|
| `'x'` or `'*'` | Latest SDK of any major version |
| `'10.0.x'` | Latest patch of 10.0 |
| `'10.0.3xx'` | Latest 10.0.3xx feature band |
| `'latest'` | Resolves via releases index API — latest non-EOL, non-preview |

```yaml
- uses: actions/setup-dotnet@v4
  with:
    dotnet-version: '10.0.x'    # latest 10.0 patch, no hardcoded patch number
```

These patterns are resolved by `DotnetVersionResolver.createDotnetVersion()` in `src/installer.ts`, which maps them to `--channel 10.0` on the install script rather than `--version 10.0.x`.

---

### Approach 2 — Skip `setup-dotnet` and use the image's preinstalled SDKs

`ubuntu-latest`, `windows-latest`, and `macos-latest` all ship with multiple .NET SDKs preinstalled. If you just want the latest available with no extra download, skip the action entirely:

```yaml
- name: Use latest preinstalled SDK
  run: dotnet --version    # prints the latest installed SDK
```

Without a `global.json`, `dotnet` always selects the **latest installed SDK** automatically. To see exactly what is preinstalled: [GitHub Actions runner-images software lists](https://github.com/actions/runner-images).

---

### Approach 3 — Query and capture at runtime (shell)

**Linux / macOS (bash):**

```bash
# Latest installed SDK
LATEST_SDK=$(dotnet --list-sdks | sort -V | tail -1 | awk '{print $1}')

# Latest Microsoft.NETCore.App runtime
LATEST_RT=$(dotnet --list-runtimes \
  | grep "^Microsoft.NETCore.App" \
  | sort -V \
  | tail -1 \
  | awk '{print $2}')

dotnet exec --fx-version "$LATEST_RT" myapp.dll
```

**Windows (PowerShell):**

```powershell
# Latest installed SDK
$latestSdk = (dotnet --list-sdks | Sort-Object | Select-Object -Last 1).Split(' ')[0]

# Latest Microsoft.NETCore.App runtime
$latestRt = (dotnet --list-runtimes |
  Where-Object { $_ -match '^Microsoft\.NETCore\.App' } |
  Sort-Object |
  Select-Object -Last 1).Split(' ')[1]

dotnet exec --fx-version $latestRt myapp.dll
```

**Expose as a step output for later steps:**

```yaml
- name: Capture latest runtime
  id: dotnet-info
  run: |
    LATEST_RT=$(dotnet --list-runtimes \
      | grep "^Microsoft.NETCore.App" \
      | sort -V | tail -1 | awk '{print $2}')
    echo "runtime=$LATEST_RT" >> $GITHUB_OUTPUT

- name: Run app with latest runtime
  run: dotnet exec --fx-version ${{ steps.dotnet-info.outputs.runtime }} myapp.dll
```

---

### Approach 4 — `global.json` with `rollForward` policy

For development and reproducible CI, `global.json` in the repo root controls SDK selection without hardcoding a full patch version:

```json
{
  "sdk": {
    "version": "9.0.0",
    "rollForward": "latestMinor"
  }
}
```

`rollForward` policies:

| Policy | Behaviour |
|---|---|
| `patch` | Latest patch of specified `major.minor.feature` |
| `feature` | Latest feature band of specified `major.minor` |
| `minor` | Latest minor ≥ specified version |
| `major` | Latest major ≥ specified version |
| `latestPatch` | Absolute latest patch of specified `major.minor` |
| `latestFeature` | Absolute latest feature band of specified `major.minor` |
| `latestMinor` | Absolute latest minor of specified major |
| `latestMajor` | Absolute latest SDK installed — version field is a floor only |
| `disable` | Exact match only — fails if exact version not installed |

To always use the latest installed SDK regardless of version:

```json
{
  "sdk": {
    "version": "0.0.0",
    "rollForward": "latestMajor"
  }
}
```

---

### Which approach to use

| Scenario | Recommended approach |
|---|---|
| Want a specific major/minor, always latest patch | `dotnet-version: '9.0.x'` in `setup-dotnet` |
| Want whatever the image has, no extra download | Skip `setup-dotnet`; use preinstalled SDK directly |
| Need `--fx-version` set dynamically in a script | Shell query: `dotnet --list-runtimes \| sort -V \| tail -1` |
| Repo-wide SDK pinning with flexibility | `global.json` with a `rollForward` policy |
| Absolute latest of everything | `dotnet-version: 'latest'` in `setup-dotnet` |

---

### Practical gotcha: `latestPatch` and feature bands

The `latestPatch` policy does **not** treat the version field as a simple floor. It uses it to identify which **feature band** to search within.

.NET SDK versions follow the format `Major.Minor.FBpatch`:

```
10.0.302
      ↑↑↑
      │└┴─ patch within the band (02)
      └─── feature band (3xx = 300–399)
```

| Version in `global.json` | Feature band targeted | Example match |
|---|---|---|
| `10.0.0` | `0xx` (000–099) | needs `10.0.001`–`10.0.099` |
| `10.0.100` | `1xx` (100–199) | `10.0.110` ✅ |
| `10.0.300` | `3xx` (300–399) | `10.0.302` ✅ |

If the image has `10.0.110`, `10.0.204`, `10.0.302` installed and your `global.json` specifies `"version": "10.0.0"` with `"rollForward": "latestPatch"`, the muxer looks for a `10.0.0xx` SDK — finds none — and exits with **code 155**:

```
A compatible .NET SDK was not found.
Requested SDK version: 10.0.0
Install the [10.0.0] .NET SDK or update global.json to match an installed SDK.
```

**Fix:** set the version to match a feature band that is actually installed:

```json
{ "sdk": { "version": "10.0.300", "rollForward": "latestPatch" } }
→ resolves to 10.0.302 ✅
```

This was confirmed in run [30086461924](https://github.com/chiranjib-swain/test-setup-dotnet/actions/runs/30086461924) — the first attempt with `10.0.0` failed on all three platforms; switching to `10.0.300` resolved to `10.0.302` cleanly.
