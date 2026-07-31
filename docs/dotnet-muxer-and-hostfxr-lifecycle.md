# .NET Muxer & `host/fxr` Lifecycle on GitHub Actions Windows Runners

> Investigation conducted July 2026 while diagnosing `actions/setup-dotnet` issue #642.

---

## Table of Contents

1. [What is the dotnet Muxer?](#1-what-is-the-dotnet-muxer)
2. [The `host/fxr/` Directory — hostfxr.dll](#2-the-hostfxr-directory--hostfxrdll)
3. [How dotnet.exe Picks the Right hostfxr.dll](#3-how-dotnetexe-picks-the-right-hostfxrdll)
4. [What `dotnet --version` Actually Reports](#4-what-dotnet---version-actually-reports)
5. [GitHub-hosted Runner Pre-installed States](#5-github-hosted-runner-pre-installed-states)
6. [Self-hosted Machine Investigation (macOS)](#6-self-hosted-machine-investigation-macos)
7. [What Pass 1 (LTS Runtime Pre-pass) Does to the Muxer](#7-what-pass-1-lts-runtime-pre-pass-does-to-the-muxer)
8. [How `-SkipNonVersionedFiles` Works](#8-how--skipnonversionedfiles-works)
9. [Side-by-side: v4 (unpatched) vs issue_642 (patched)](#9-side-by-side-v4-unpatched-vs-issue_642-patched)
10. [Verified Test Results from GitHub Actions Runs](#10-verified-test-results-from-github-actions-runs)

---

## 1. What is the dotnet Muxer?

`dotnet.exe` (Windows) / `dotnet` (Linux/macOS) is called the **muxer** (multiplexer). It is a
small (~300 KB) native launcher binary located at the root of the .NET install directory:

```
C:\Program Files\dotnet\dotnet.exe     ← Windows
/usr/share/dotnet/dotnet               ← Linux
~/.dotnet/dotnet                       ← macOS
```

Its sole responsibilities are:

1. Read `global.json` in the current directory (and parent directories) to find the requested SDK version
2. Scan the `sdk/` subdirectory for installed SDKs
3. Apply the `rollForward` policy to select the best matching SDK
4. Delegate execution to the selected SDK's own `dotnet.dll`

The muxer is **not tied to any specific SDK version**. It is a shared binary that can dispatch to
any installed SDK version living under the same root directory.

---

## 2. The `host/fxr/` Directory — hostfxr.dll

The muxer does not perform SDK resolution itself. It loads a versioned library called
`hostfxr.dll` (Windows) / `libhostfxr.so` (Linux) / `libhostfxr.dylib` (macOS) from the
`host/fxr/` subdirectory:

```
C:\Program Files\dotnet\
├── dotnet.exe                          ← muxer (non-versioned)
├── host\
│   └── fxr\
│       ├── 8.0.28\
│       │   └── hostfxr.dll            ← versioned (lives under 8.0.28/)
│       ├── 9.0.18\
│       │   └── hostfxr.dll            ← versioned (lives under 9.0.18/)
│       └── 10.0.9\
│           └── hostfxr.dll            ← versioned (lives under 10.0.9/)
├── sdk\
│   ├── 8.0.423\
│   ├── 9.0.316\
│   └── 10.0.301\
└── shared\
    └── Microsoft.NETCore.App\
        ├── 8.0.29\
        ├── 9.0.18\
        └── 10.0.9\
```

`host/fxr/<version>/` **is a versioned directory** — the path contains a semver folder name.
The install script's version regex (`.*/\d+\.\d+[^/]+/`) matches it, so `hostfxr.dll` is
treated as a **versioned file** and is always extracted from the archive. It is never skipped
by `-SkipNonVersionedFiles`.

Each runtime release ships its own `hostfxr.dll`. They are forward/backward compatible —
the 10.0.x version of `hostfxr.dll` can locate and launch 8.x or 9.x SDKs.

---

## 3. How `dotnet.exe` Picks the Right `hostfxr.dll`

This logic is hardcoded inside `dotnet.exe` itself — `setup-dotnet` has no involvement:

```
dotnet.exe boots
    │
    ▼
Scans host\fxr\ for all subdirectories
    │   found: 8.0.28\, 9.0.18\, 10.0.9\
    ▼
Picks the HIGHEST version (10.0.9 > 9.0.18 > 8.0.28)
    │
    ▼
Loads host\fxr\10.0.9\hostfxr.dll into memory
    │
    ▼
hostfxr reads global.json  →  applies rollForward policy
    │
    ▼
hostfxr scans sdk\  →  selects best matching SDK (e.g. 9.0.316)
    │
    ▼
Launches sdk\9.0.316\dotnet.dll
```

**Key point:** `dotnet.exe` always loads the highest available `hostfxr.dll`. If Pass 1
installs `host/fxr/10.0.10/hostfxr.dll`, that becomes the new highest version and will be
loaded on every subsequent `dotnet` invocation in that job.

---

## 4. What `dotnet --version` Actually Reports

`dotnet --version` ≠ muxer binary version.

It shows the **SDK version that hostfxr resolved to**, based on `global.json` and installed SDKs.
The muxer binary version is shown separately in `dotnet --info`:

```
dotnet --info output:

.NET SDK:
  Version:      9.0.316       ← resolved SDK (respects global.json)
  Base Path:    C:\Program Files\dotnet\sdk\9.0.316\

Host:
  Version:      10.0.9        ← loaded hostfxr.dll version (highest in host\fxr\, from runner image)
  Architecture: x64
  Commit:       901ca94124
```

| Field | What it shows |
|---|---|
| `.NET SDK: Version` | SDK selected by hostfxr via global.json resolution |
| `Host: Version` | The `hostfxr.dll` binary version that was loaded |
| `dotnet.exe` PE `ProductVersion` (Windows only) | The muxer binary's own build version — can be **older** than `Host: Version` |

### Muxer binary version vs loaded hostfxr version

The muxer binary's own version does **not** need to match the `Host: Version` it reports.
Observed on a fresh `windows-latest` runner:

```
dotnet.exe  ProductVersion : 10.0.8    ← muxer binary version (Windows PE version resource)
dotnet --info  Host: Version : 10.0.9   ← loaded hostfxr (highest in host\fxr\)
```

```
dotnet.exe (ProductVersion 10.0.8)
      │
      ▼
scans host\fxr\  →  finds 10.0.8\, 10.0.9\ (and older 8.x, 9.x)
      │
      ▼
loads host\fxr\10.0.9\hostfxr.dll  (10.0.9 is highest)
```

### Identifying the muxer binary version per platform

| Platform | How to read muxer binary version | Notes |
|---|---|---|
| Windows | PE version resource (`ProductVersion`, `FileVersion`, commit hash) | `(Get-Item dotnet.exe).VersionInfo.ProductVersion` |
| Linux/macOS | No version embedded in Mach-O / ELF binary | SHA256 hash is the reliable way to detect muxer changes |

---

## 5. GitHub-hosted Runner Pre-installed States

All three GitHub-hosted runner types come with .NET **pre-installed** by the runner image.
The data below was captured from the "Initial Machine State" step before any `setup-dotnet`
action ran.

### `windows-latest`

```
C:\Program Files\dotnet\
├── dotnet.exe              ← muxer binary (167K, ProductVersion: 10.0.8, SHA256: CBB1746F...)
├── host\fxr\
│   ├── 8.0.6\, 8.0.22\, 8.0.28\
│   ├── 9.0.6\, 9.0.17\
│   ├── 10.0.8\
│   └── 10.0.9\  ← highest → Host: 10.0.9     ❌ LTS 10.0.10 NOT present
├── sdk\  8.0.128, 8.0.206, 8.0.319, 8.0.422, 9.0.118, 9.0.205, 9.0.315, 10.0.109, 10.0.204, 10.0.301
└── shared\  NETCore + AspNetCore + WindowsDesktop: 8.0.6, 8.0.22, 8.0.28, 9.0.6, 9.0.17, 10.0.8, 10.0.9
```

`dotnet --info  Host: 10.0.9`  (muxer `ProductVersion: 10.0.8` — see § 4)

### `ubuntu-latest`

```
/usr/share/dotnet/
├── dotnet                  ← muxer (67K, SHA256: f83fb9e3...)
│     /usr/bin/dotnet → /usr/share/dotnet/dotnet  (symlink)
├── host/fxr/
│   ├── 8.0.6/, 8.0.22/, 8.0.29/
│   ├── 9.0.6/, 9.0.18/
│   ├── 10.0.8/
│   └── 10.0.10/  ← highest → Host: 10.0.10   ✅ LTS already present
├── sdk/  8.0.129, 8.0.206, 8.0.319, 8.0.423, 9.0.119, 9.0.205, 9.0.316, 10.0.110, 10.0.204, 10.0.302
└── shared/  NETCore + AspNetCore: 8.0.6, 8.0.22, 8.0.29, 9.0.6, 9.0.18, 10.0.8, 10.0.10
```

`dotnet --info  Host: 10.0.10`  (latest active LTS **already pre-installed** by runner image)

### `macos-latest` (arm64)

```
/Users/runner/.dotnet/
├── dotnet                  ← muxer (138K, SHA256: 49cf4f38...)
│     /usr/local/bin/dotnet → /Users/runner/.dotnet/dotnet  (symlink)
├── host/fxr/
│   ├── 8.0.1/, 8.0.4/, 8.0.7/, 8.0.29/
│   ├── 9.0.1/, 9.0.4/, 9.0.18/
│   ├── 10.0.3/, 10.0.7/
│   └── 10.0.10/  ← highest → Host: 10.0.10   ✅ LTS already present
├── sdk/  8.0.101, 8.0.204, 8.0.303, 8.0.423, 9.0.102, 9.0.203, 9.0.316, 10.0.103, 10.0.203, 10.0.302
└── shared/  NETCore + AspNetCore: 8.0.1–8.0.29, 9.0.1–9.0.18, 10.0.3, 10.0.7, 10.0.10
```

`dotnet --info  Host: 10.0.10`  (latest active LTS **already pre-installed** by runner image)

### Cross-platform summary

| Runner | Install root | Muxer size | Initial highest `host/fxr/` | `Host: Version` | LTS 10.0.10 pre-installed? |
|---|---|---|---|---|---|
| `windows-latest` | `C:\Program Files\dotnet\` | 167 KB | `10.0.9` | `10.0.9` | ❌ No |
| `ubuntu-latest` | `/usr/share/dotnet/` | 67 KB | `10.0.10` | `10.0.10` | ✅ Yes |
| `macos-latest` | `/Users/runner/.dotnet/` | 138 KB | `10.0.10` | `10.0.10` | ✅ Yes |

### Implication for Pass 1

The LTS runtime pre-pass (Pass 1) was:

- **`ubuntu-latest` / `macos-latest`**: entirely redundant — `host/fxr/10.0.10/` was already
  present before the action ran. Pass 1 re-downloaded and re-installed what was already there.
- **`windows-latest`**: `host/fxr/10.0.10/` was not present initially, so Pass 1 added it. The existing `dotnet.exe` muxer (ProductVersion `10.0.8`) was preserved, and on subsequent invocations it correctly loaded the newly installed `hostfxr` `10.0.10`, demonstrating that the muxer is forward-compatible with newer `hostfxr` versions.

Skipping Pass 1 on **all platforms** is therefore correct:
- Windows: eliminates the hostfxr upgrade side-effect and file-lock risk
- Ubuntu / macOS: eliminates a redundant download (the LTS runtime is already there)

---

## 6. Self-hosted Machine Investigation (macOS)

To confirm installer behavior independently of the runner image's pre-installed state, the
investigation was repeated on a personal macOS machine where all file operations were directly
observable.

### Initial system state

```
/usr/local/share/dotnet/
├── dotnet                    ← system muxer  (SHA256: 6f00aa40903edb...)
├── host/fxr/
│   └── 9.0.3/libhostfxr.dylib
└── sdk/
    └── 9.0.202/

dotnet --info  Host:  9.0.3
```

### First `setup-dotnet` run (`dotnet-version: 9.0.x`)

`setup-dotnet` does **not** modify the existing system installation at `/usr/local/share/dotnet/`.
Instead, it creates a private installation directory and updates the shell environment:

```
DOTNET_ROOT → /Users/chiranjib/.dotnet
PATH        → /Users/chiranjib/.dotnet prepended
```

The active `dotnet` binary changed:

```
Before:  /usr/local/share/dotnet/dotnet
After:   /Users/chiranjib/.dotnet/dotnet
```

SHA256 of the muxer binary:

```
Before (system muxer):  6f00aa40903edb...
After first run:        7118e57d820...    ← new muxer binary written to ~/.dotnet by Pass 1
```

Although only `9.0.x` was requested, Pass 1 installed the latest active LTS runtime, resulting in:

```
~/.dotnet/host/fxr/
├── 9.0.18/libhostfxr.dylib
├── 10.0.0/libhostfxr.dylib
└── 10.0.10/libhostfxr.dylib   ← highest → Host: 10.0.10

dotnet --info  Host:  10.0.10
```

### Second `setup-dotnet` run (`dotnet-version: 10.0.x`)

```
Muxer SHA256 before:  7118e57d820...
Muxer SHA256 after:   7118e57d820...   ← identical — muxer NOT overwritten
```

```
host/fxr/  before:  9.0.18, 10.0.0, 10.0.10
host/fxr/  after:   9.0.18, 10.0.0, 10.0.10   ← unchanged
```

```
sdk/ added:  10.0.302   ← only new versioned component added
```

`Host: Version` remained `10.0.10`.

### Key findings from self-hosted investigation

| Finding | Evidence |
|---|---|
| Action installs to `~/.dotnet`, not system path | `DOTNET_ROOT=/Users/chiranjib/.dotnet` set by action |
| System installation (`/usr/local/share/dotnet/`) untouched | No changes after action runs |
| First install creates a new muxer | SHA256 changed: `6f00aa40903...` → `7118e57d820...` |
| Subsequent installs **do not overwrite** the muxer | SHA256 unchanged before and after second run |
| Only versioned components added on subsequent installs | Only `sdk/10.0.302/` was new; `host/fxr/` and muxer unchanged |
| `-SkipNonVersionedFiles` working as designed | SHA256 identity confirms non-versioned muxer preserved on re-install |

### Verification: Pass 1 skipped on all platforms

After extending the Pass 1 skip to **all platforms** (not just Windows), the same macOS
self-hosted runner was re-tested with an install sequence of `8.0.423` → `9.0.x`
(run `30627991281`, job `91147568456`):

**After `8.0.423` install:**

```
~/.dotnet/dotnet  109K  written at 17:13
SHA256:  6a854538c480bf57ca7f578a19d14ececc65213025728389ffb3ac371de474d1

host/fxr/
└── 8.0.29/           ← only the 8.0.x hostfxr — no LTS (10.0.x) entry

Host:    8.0.29
SDK:     8.0.423
```

**After `9.0.x` install:**

```
~/.dotnet/dotnet  109K  written at 17:13   ← same timestamp — binary NOT touched
SHA256:  6a854538c480bf57ca7f578a19d14ececc65213025728389ffb3ac371de474d1   ← identical

host/fxr/
├── 8.0.29/           ← unchanged from first install
└── 9.0.18/           ← added by 9.0.x SDK install

Host:    9.0.18   ← correctly updated to new highest hostfxr
SDKs:    8.0.423, 9.0.316
```

The muxer SHA256 is **identical** across both installs. The binary written by the `8.0.423`
SDK installer was preserved unchanged when `9.0.x` was installed (`--skip-non-versioned-files`
skipped it because it already existed). The muxer written by an older SDK installer has no
problem loading a newer hostfxr — it correctly selected `9.0.18` as the highest available.

**Comparison: original behaviour (Pass 1 running on macOS) vs Pass 1 fully skipped:**

| | Original (Pass 1 on macOS) | Pass 1 skipped (all platforms) |
|---|---|---|
| `host/fxr/` after `9.0.x` | `9.0.18, 10.0.0, 10.0.10` | `8.0.29, 9.0.18` |
| `Host: Version` after `9.0.x` | `10.0.10` (from LTS pre-pass) | `9.0.18` (from SDK install) |
| Extra runtime downloaded | LTS 10.0.10 (~extra MB) | None |
| Muxer SHA256 preserved on 2nd install | ✅ | ✅ |
| Both SDKs functional | ✅ | ✅ |

---

## 7. What Pass 1 (LTS Runtime Pre-pass) Does to the Muxer

In the **unpatched `v4`** of `setup-dotnet`, `installDotnet()` runs two passes on all platforms:

### Pass 1 (old behaviour — runs on ALL platforms including Windows)
```
install-dotnet.ps1 -SkipNonVersionedFiles -Runtime dotnet -Channel LTS
```

- Downloads `dotnet-runtime-10.0.10-win-x64.zip` (~37 MB) — the latest active LTS runtime
- Extracts to `C:\Program Files\dotnet\`
- Writes `shared\Microsoft.NETCore.App\10.0.10\` — **versioned**, always extracted
- Writes `host\fxr\10.0.10\hostfxr.dll` — **versioned** (path contains `10.0.10/`), always extracted
- `dotnet.exe` — **non-versioned** (no version in path), **skipped** because `-SkipNonVersionedFiles` is set and it already exists

After Pass 1:
```
host\fxr\
├── 8.0.28\hostfxr.dll
├── 9.0.18\hostfxr.dll
├── 10.0.8\hostfxr.dll   ← from runner image
├── 10.0.9\hostfxr.dll   ← from runner image (highest before Pass 1)
└── 10.0.10\hostfxr.dll  ← NEW — added by Pass 1, becomes new highest
```

`dotnet.exe` now loads `10.0.10\hostfxr.dll` (highest version). **The active hostfxr changed.**

Confirmed from live runner log (job `90809130877`, run `30523527111`):
```
Pass 1:  install-dotnet.ps1 -SkipNonVersionedFiles -Runtime dotnet -Channel LTS
         → dotnet-install: Installed version is 10.0.10

dotnet --info Host:
  Version:      10.0.10    ← upgraded from 10.0.9
```

### Pass 2 (same in both v4 and patched)
```
install-dotnet.ps1 -SkipNonVersionedFiles -Channel 9.0
```

- Downloads `dotnet-sdk-9.0.316-win-x64.zip` (~298 MB)
- Extracts `sdk\9.0.316\` and `shared\...\9.0.18\` — versioned, written
- `host\fxr\9.0.18\` — already exists if 9.0.18 hostfxr was there, otherwise written
- `dotnet.exe` — **skipped** (non-versioned, already exists)

---

## 8. How `-SkipNonVersionedFiles` Works

The install script classifies every file in the archive by this regex:

```
.*/\d+\.\d+[^/]+/
```

Files whose path **matches** this regex → **versioned** → always extracted  
Files whose path **does not match** → **non-versioned** → skipped if `-SkipNonVersionedFiles` is set AND the file already exists locally

### Classification table

| Path in archive | Matches version regex? | With `-SkipNonVersionedFiles` |
|---|---|---|
| `dotnet.exe` | ❌ No | Skipped if already exists |
| `host/fxr/10.0.10/hostfxr.dll` | ✅ Yes (`10.0.10/`) | Always extracted |
| `sdk/9.0.316/dotnet.dll` | ✅ Yes (`9.0.316/`) | Always extracted |
| `shared/Microsoft.NETCore.App/9.0.18/...` | ✅ Yes (`9.0.18/`) | Always extracted |

**This is why Pass 1 adds `host\fxr\10.0.10\` even with `-SkipNonVersionedFiles`** — the
`hostfxr.dll` lives inside a versioned directory, so it is never skipped.
### SHA256 evidence from self-hosted investigation (§ 6)

Direct SHA256 measurement confirms the muxer binary is preserved on subsequent installs:

| Install | `~/.dotnet/dotnet` SHA256 | Notes |
|---|---|---|
| After first `setup-dotnet` run (`9.0.x`) | `7118e57d820...` | New muxer created (Pass 1 wrote it) |
| After second `setup-dotnet` run (`10.0.x`) | `7118e57d820...` | Identical — non-versioned file not overwritten |

Only versioned components (`sdk/10.0.302/`) were added on the second run.
---

## 9. Side-by-side: v4 (unpatched) vs issue_642 (patched)

### v4 — Windows (unpatched)

```
Runner boots → dotnet.exe (ProductVersion: 10.0.8, Host: 10.0.9), host\fxr: [8.0.28, 9.0.18, 10.0.8, 10.0.9]

Pass 1: -Runtime dotnet -Channel LTS
    → Downloads runtime 10.0.10 (~37 MB)
    → Adds host\fxr\10.0.10\hostfxr.dll   ← NEW highest hostfxr
    → dotnet.exe SKIPPED (non-versioned, exists)
    → Active hostfxr: 10.0.10

Pass 2: -SkipNonVersionedFiles -Channel 9.0
    → Downloads SDK 9.0.316 (~298 MB)
    → Adds sdk\9.0.316\
    → dotnet.exe SKIPPED
    → Active hostfxr: 10.0.10 (unchanged)

Final Host: 10.0.10  |  Active SDK: 9.0.316
```

### issue_642 — Windows (patched — Pass 1 skipped)

```
Runner boots → dotnet.exe (ProductVersion: 10.0.8, Host: 10.0.9), host\fxr: [8.0.28, 9.0.18, 10.0.8, 10.0.9]

[Pass 1 skipped entirely on Windows]

Pass 2: -SkipNonVersionedFiles -Channel 9.0
    → Downloads SDK 9.0.316 (~298 MB)
    → Adds sdk\9.0.316\, host\fxr\9.0.18\ (already exists → skipped)
    → dotnet.exe SKIPPED
    → Active hostfxr: 10.0.9 (unchanged from runner image)

Final Host: 10.0.9  |  Active SDK: 9.0.316
```

### Key difference

| | v4 (unpatched) | issue_642 (patched) |
|---|---|---|
| Extra download (Pass 1) | ~37 MB runtime zip | None |
| `host\fxr` after install | 8.0.28, 9.0.18, 10.0.8, 10.0.9, **10.0.10** | 8.0.28, 9.0.18, 10.0.8, 10.0.9 |
| Active `hostfxr.dll` | **10.0.10** (upgraded by Pass 1) | **10.0.9** (runner image) |
| `dotnet.exe` binary | Unchanged (skipped) | Unchanged (skipped) |
| Active SDK | 9.0.316 | 9.0.316 |
| Risk of file-lock on `dotnet.exe` | Higher (more operations) | Eliminated |

Both produce a working `dotnet 9.0.316`. The patch removes an unnecessary download and
avoids the `hostfxr.dll` upgrade side-effect on Windows.

---

## 10. Verified Test Results from GitHub Actions Runs

All results from repo: `chiranjib-swain/test-setup-dotnet`

### Run #4 — issue_642 patch, all 3 jobs ✅ (run `30521410168`)

| Job | Runner | What action downloaded | Host after install | Active SDK |
|---|---|---|---|---|
| Windows single (9.0.x) | Fresh VM | SDK 9.0.316 (298 MB) — real download | **10.0.9** (runner image) | 9.0.316 |
| Windows multi (8.0.x + 9.0.x) | Fresh VM | SDK 8.0.423 (285 MB) + SDK 9.0.316 (298 MB) | **10.0.9** (runner image) | 9.0.316 |
| Ubuntu multi baseline | Fresh VM | Already installed — no-op | N/A | 9.0.316 |

### Run — v4 official (run `30523527111`, job `90809130877`)

| Pass | Command | Downloaded | Effect on host\fxr |
|---|---|---|---|
| Pass 1 | `-Runtime dotnet -Channel LTS` | runtime 10.0.10 (37 MB) | Added `host\fxr\10.0.10\` |
| Pass 2 | `-Channel 9.0` | SDK 9.0.316 (298 MB) | No new fxr entry |
| **Final Host** | | | **10.0.10** (upgraded) |

---


