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

> **Runner image updated ~2026-07-28** (release `win25/20260728.188`): the image now ships
> `Microsoft.NETCore.App 10.0.10`, `Microsoft.AspNetCore.App 10.0.10`, and
> `Microsoft.WindowsDesktop.App 10.0.10` preinstalled, alongside `host\fxr\10.0.10\`.
> Older entries (`10.0.9`, `9.0.17`, `8.0.28`) were replaced by their latest patch equivalents.
> See runner-images releases: `github.com/actions/runner-images/releases`

```
C:\Program Files\dotnet\
├── dotnet.exe              ← muxer binary
├── host\fxr\
│   ├── 8.0.22\
│   ├── 10.0.8\
│   └── 10.0.10\  ← highest → Host: 10.0.10   ✅ LTS NOW present (image updated)
├── sdk\  8.0.129, 8.0.206, 8.0.319, 8.0.423, 9.0.119, 9.0.205, 9.0.316, 10.0.110, 10.0.204, 10.0.302
└── shared\  NETCore + AspNetCore + WindowsDesktop: 8.0.6, 8.0.22, 8.0.29, 9.0.6, 9.0.18, 10.0.8, 10.0.10
```

`dotnet --version  10.0.302`  (highest SDK),  `dotnet --info  Host: 10.0.10`

Confirmed from [run 30785651154](https://github.com/chiranjib-swain/test-setup-dotnet/actions/runs/30785651154) (arch-multiversion-test.yml, `windows-latest` job, BEFORE step, 2026-08-03).

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

#### As of runner image update ~2026-07-28 (all platforms now ship LTS 10.0.10)

| Runner | Install root | Initial highest `host/fxr/` | `Host: Version` | LTS 10.0.10 pre-installed? |
|---|---|---|---|---|
| `windows-latest` | `C:\Program Files\dotnet\` | `10.0.10` | `10.0.10` | ✅ Yes (updated) |
| `windows-11-arm` | `C:\Program Files\dotnet\` | `10.0.10` | `10.0.10` | ✅ Yes |
| `ubuntu-latest` | `/usr/share/dotnet/` | `10.0.10` | `10.0.10` | ✅ Yes |
| `ubuntu-24.04-arm` | `/usr/share/dotnet/` | `10.0.10` | `10.0.10` | ✅ Yes |
| `macos-15-intel` | `/Users/runner/.dotnet/` | `10.0.10` | `10.0.10` | ✅ Yes |
| `macos-latest` (arm64) | `/Users/runner/.dotnet/` | `10.0.10` | `10.0.10` | ✅ Yes |

#### Previously (before runner image update — observed in earlier investigation)

| Runner | Initial highest `host/fxr/` | LTS 10.0.10 pre-installed? |
|---|---|---|
| `windows-latest` | `10.0.9` | ❌ No |
| `ubuntu-latest` | `10.0.10` | ✅ Yes |
| `macos-latest` | `10.0.10` | ✅ Yes |

### Implication for Pass 1

The LTS runtime pre-pass (Pass 1) is **redundant on all GitHub-hosted runners**:

- **`ubuntu-latest` / `macos-latest`**: always redundant — `host/fxr/10.0.10/` was already
  present before the action ran even before the image update.
- **`windows-latest` (after ~2026-07-28 image update)**: the runner image now ships
  `host\fxr\10.0.10\` preinstalled. Pass 1 in `actions/setup-dotnet@v4` downloads
  ~37 MB of runtime zip and performs a full install — but since `10.0.10` is already present,
  every file it would write either already exists (non-versioned → skipped by
  `-SkipNonVersionedFiles`) or is identical (versioned runtime entries). **Pass 1 is a
  complete no-op**. Verified empirically: `host\fxr\` is identical BEFORE and AFTER Pass 1
  on [run 30785651154](https://github.com/chiranjib-swain/test-setup-dotnet/actions/runs/30785651154) (2026-08-03).

Skipping Pass 1 on **all platforms** is therefore correct:
- Windows: eliminates a ~37 MB redundant download; removes the hostfxr upgrade side-effect and file-lock risk entirely
- Ubuntu / macOS: eliminates a redundant download (the LTS runtime was already there before the image update too)

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

Confirmed from live runner log (job `90809130877`, [run 30523527111](https://github.com/chiranjib-swain/test-setup-dotnet/actions/runs/30523527111)):
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

### Run #4 — issue_642 patch, all 3 jobs ✅ ([run 30521410168](https://github.com/chiranjib-swain/test-setup-dotnet/actions/runs/30521410168))

| Job | Runner | What action downloaded | Host after install | Active SDK |
|---|---|---|---|---|
| Windows single (9.0.x) | Fresh VM | SDK 9.0.316 (298 MB) — real download | **10.0.9** (runner image at the time) | 9.0.316 |
| Windows multi (8.0.x + 9.0.x) | Fresh VM | SDK 8.0.423 (285 MB) + SDK 9.0.316 (298 MB) | **10.0.9** (runner image at the time) | 9.0.316 |
| Ubuntu multi baseline | Fresh VM | Already installed — no-op | N/A | 9.0.316 |

### Run — v4 official ([run 30523527111](https://github.com/chiranjib-swain/test-setup-dotnet/actions/runs/30523527111), job `90809130877`) — pre-image-update

| Pass | Command | Downloaded | Effect on host\fxr |
|---|---|---|---|
| Pass 1 | `-Runtime dotnet -Channel LTS` | runtime 10.0.10 (37 MB) | Added `host\fxr\10.0.10\` |
| Pass 2 | `-Channel 9.0` | SDK 9.0.316 (298 MB) | No new fxr entry |
| **Final Host** | | | **10.0.10** (upgraded from 10.0.9) |

---

### Run — v4 official, 6 runners ✅ — post-image-update ([run 30785651154](https://github.com/chiranjib-swain/test-setup-dotnet/actions/runs/30785651154), 2026-08-03)

Workflow: `arch-multiversion-test.yml`, installs `8.0.x + 9.0.x` via `actions/setup-dotnet@v4`.
All 6 jobs passed.

**Key finding:** `host\fxr\` on `windows-latest` is **identical BEFORE and AFTER** Pass 1 —
confirming Pass 1 is a complete no-op after the runner image update.

| Runner | host/fxr BEFORE setup-dotnet | host/fxr AFTER setup-dotnet | Pass 1 effect |
|---|---|---|---|
| `windows-latest` | `10.0.10, 10.0.8, 8.0.22` | `10.0.10, 10.0.8, 8.0.22` | **no-op** — 10.0.10 was already there |
| `windows-11-arm` | `10.0.10, ...` | `10.0.10, ...` | no-op |
| `ubuntu-latest` | `10.0.10, 10.0.8, 9.0.18, ...` | `10.0.10, 10.0.8, 9.0.18, ...` | no-op (was always redundant) |
| `ubuntu-24.04-arm` | `10.0.10, ...` | `10.0.10, ...` | no-op |
| `macos-15-intel` | `10.0.10, ...` | `10.0.10, ...` | no-op |
| `macos-latest` | `10.0.10, ...` | `10.0.10, ...` | no-op (was always redundant) |

**`DOTNET_ROOT` after setup-dotnet on `windows-latest`:** `C:\Program Files\dotnet`

---

### Run — issue_642 branch (no Pass 1), 6 runners ✅ — post-image-update ([run 30786052855](https://github.com/chiranjib-swain/test-setup-dotnet/actions/runs/30786052855), 2026-08-03)

Workflow: `issue-642-no-pass1-test.yml`, installs `8.0.x + 9.0.x` via `chiranjib-swain/setup-dotnet@issue_642`.
All 6 jobs passed.

Branch change: `installDotnet()` in `src/installer.ts` — Pass 1 block commented out entirely
(commit `40df988`, `chiranjib-swain/setup-dotnet`).

**`windows-latest` BEFORE vs AFTER (issue_642 — Pass 2 only):**

```
BEFORE:
  dotnet --version : 10.0.302
  SDKs  : 8.0.129, 8.0.206, 8.0.319, 8.0.423, 9.0.119, 9.0.205, 9.0.316,
          10.0.110, 10.0.204, 10.0.302   ← all preinstalled by runner image
  host/fxr: 10.0.10, 10.0.8, 8.0.22    ← LTS already present

AFTER (no Pass 1, only Pass 2 for 8.0.x + 9.0.x):
  dotnet --version : 10.0.302            ← unchanged
  SDKs  : identical to BEFORE            ← already present, no new download
  host/fxr: 10.0.10, 10.0.8, 8.0.29,
            9.0.18, 9.0.6, 8.0.22       ← new 8.0.29 and 9.0.18 entries added
            by SDK installers
  DOTNET_ROOT = C:\Program Files\dotnet
```

| | `actions/setup-dotnet@v4` ([run 30785651154](https://github.com/chiranjib-swain/test-setup-dotnet/actions/runs/30785651154)) | `issue_642` ([run 30786052855](https://github.com/chiranjib-swain/test-setup-dotnet/actions/runs/30786052855)) |
|---|---|---|
| Pass 1 executed | Yes (but no-op — LTS already present) | No (skipped) |
| Extra download | ~37 MB runtime zip (wasted) | None |
| host/fxr AFTER | `10.0.10, 10.0.8, 8.0.22` (unchanged) | `10.0.10, 10.0.8, 8.0.29, 9.0.18, ...` |
| All 6 runners passed | ✅ | ✅ |
| Functional difference | None | None |

**Runner-image release evidence:** `actions/runner-images` release `win25/20260728.188` (2026-07-28)
shows `.NET Core Tools Added: Microsoft.NETCore.App 8.0.29, 9.0.18, 10.0.10` and
`Deleted: Microsoft.NETCore.App 8.0.28, 9.0.17, 10.0.9` — this is the update that brought
`host\fxr\10.0.10\` to the Windows image, making Pass 1 redundant on Windows too.

---

### Run — Ubuntu container clean-slate test, both jobs ✅ ([run 30894061687](https://github.com/chiranjib-swain/test-setup-dotnet/actions/runs/30894061687), 2026-08-04)

Workflow: `container-ubuntu-test.yml`.
Runner: `ubuntu-latest` (host). Container: `ubuntu:24.04` (job environment — **zero .NET preinstalled**).
This is the TL-requested scenario: what happens when Pass 1 is the very first install, not a redundant one.

**BEFORE (both jobs):**
```
(no dotnet — clean slate confirmed)
/root/.dotnet not present
/usr/share/dotnet not present
```

**v4 job — Pass 1 + Pass 2 on clean slate:**
```
Pass 1: install-dotnet.sh --skip-non-versioned-files --runtime dotnet --channel LTS
        → Downloaded dotnet-runtime-10.0.10-linux-x64.tar.gz  (36,651,444 bytes = ~37 MB)
        → Installed version is 10.0.10
        → DOTNET_ROOT = /usr/share/dotnet

Pass 2: install-dotnet.sh --channel 9.0
        → Downloaded dotnet-sdk-9.0.316-linux-x64.tar.gz  (218,192,299 bytes = ~218 MB)
        → Installed version is 9.0.316

AFTER:
  DOTNET_ROOT = /usr/share/dotnet
  SDK:      9.0.316
  Runtimes: Microsoft.AspNetCore.App 9.0.18
            Microsoft.NETCore.App   9.0.18
            Microsoft.NETCore.App  10.0.10   ← added by Pass 1
  host/fxr: 10.0.10, 9.0.18               ← Pass 1 wrote 10.0.10 first
```

**issue_642 job — Pass 2 only on clean slate:**
```
[Pass 1 skipped]

Pass 2: install-dotnet.sh --channel 9.0
        → Downloaded dotnet-sdk-9.0.316-linux-x64.tar.gz  (218,192,299 bytes = ~218 MB)
        → Installed version is 9.0.316

AFTER:
  DOTNET_ROOT = /usr/share/dotnet
  SDK:      9.0.316
  Runtimes: Microsoft.AspNetCore.App 9.0.18
            Microsoft.NETCore.App   9.0.18   ← only the SDK's own runtime, no LTS
  host/fxr: 9.0.18                          ← only the SDK's own hostfxr, no 10.0.10
```

**Side-by-side comparison (container clean-slate):**

| | `v4` (Pass 1 + Pass 2) | `issue_642` (Pass 2 only) |
|---|---|---|
| BEFORE: any .NET? | ❌ None | ❌ None |
| Downloads | ~37 MB (LTS runtime) + ~218 MB (SDK) = **~255 MB** | ~218 MB (SDK only) = **~218 MB** |
| `host/fxr/` AFTER | `10.0.10`, `9.0.18` | `9.0.18` only |
| Extra LTS runtime installed | `Microsoft.NETCore.App 10.0.10` | None |
| `dotnet --version` works | ✅ | ✅ |
| SDK functional | ✅ | ✅ |

**Conclusion:** Even on a completely clean Ubuntu container (zero .NET), removing Pass 1 works correctly.
The SDK installer (Pass 2) writes the muxer, `host/fxr/9.0.18/`, and the runtime in a single step.
No LTS runtime is installed unnecessarily. `dotnet --version` and `dotnet --list-sdks` work as expected.
The ~37 MB LTS runtime download is eliminated with no functional regression.

---

### Run — Self-hosted Windows x64, clean-slate, EOL+current versions ([run 30904891459](https://github.com/chiranjib-swain/test-setup-dotnet/actions/runs/30904891459), 2026-08-04)

Workflow: `windows-self-hosted.yml`.
Runner: `Muxer-test` (self-hosted, Windows, x64). Runner version: `2.336.0`.
Both jobs had `clean_slate: true` — `C:\Program Files\dotnet` deleted before each job.
Versions requested: `6.0.x + 7.0.x`.

> **Note on build failure:** Both jobs failed at the final `dotnet build` step.
> The `test-setup-dotnet` repo contains a `global.json` requiring SDK `9.0.100`.
> On this persistent self-hosted runner, 8.0 and 9.0 SDKs were present in the
> **runner's tool cache** (`C:\Windows\System32\actions-runner\_work\_tool\`) from a prior run.
> The clean-slate step removes `C:\Program Files\dotnet` (DOTNET_ROOT) but does **not** touch
> to DOTNET_ROOT. After clean slate, `dotnet --list-sdks` (which reads DOTNET_ROOT) only saw
> 6.0 and 7.0, so `dotnet build` with `global.json: 9.0.100` failed.
> This is a tool cache / clean-slate interaction specific to persistent self-hosted runners —
> not related to the Pass 1 / muxer investigation. The AFTER step data is fully valid.

**BEFORE (both jobs):**
```
C:\Program Files\dotnet exists: False
(no dotnet — clean slate confirmed)
```

**v4 job ([job 91977578375](https://github.com/chiranjib-swain/test-setup-dotnet/actions/runs/30904891459/job/91977578375)) — Pass 1 + Pass 2:**
```
Pass 1: install-dotnet.ps1 -SkipNonVersionedFiles -Runtime dotnet -Channel LTS
        → Installed Microsoft.NETCore.App 10.0.10 runtime
        → Added host\fxr\10.0.10\hostfxr.dll  ← NEW highest

Pass 2 (6.0): Downloaded dotnet-sdk-6.0.428-win-x64.zip (265 MB) → Installed 6.0.428
Pass 2 (7.0): Downloaded dotnet-sdk-7.0.410-win-x64.zip (288 MB) → Installed 7.0.410

AFTER (DOTNET_ROOT = C:\Program Files\dotnet):
  SDKs at DOTNET_ROOT : 6.0.428, 7.0.410   (8.0/9.0 in tool cache only)
  Runtimes            : NETCore+AspNetCore+WinDesktop 6.0.36, 7.0.20
                        Microsoft.NETCore.App 10.0.10  ← added by Pass 1
  host\fxr            : 10.0.10, 6.0.36, 7.0.20       ← 10.0.10 highest (Pass 1)
  Host: Version       : 10.0.10  Architecture: x64  RID: win-x64
                        Commit: f7d90799ce

  Muxer binary        : C:\Program Files\dotnet\dotnet.exe
  Size                : 167,208 bytes
  ProductVersion      : 10.0.10 @Commit: f7d90799ce4ef09a0bb257852a57248d2a8fb8dd
  FileVersion         : 10,0,1026,32716 @Commit: f7d90799ce4ef09a0bb257852a57248d2a8fb8dd
  SHA256              : 4377F10C78400F0370B88156773DE9843C07F14E65B3232005EC3179EF38D463
```

**issue_642 job ([job 91977578329](https://github.com/chiranjib-swain/test-setup-dotnet/actions/runs/30904891459/job/91977578329)) — Pass 2 only:**
```
[Pass 1 skipped — no LTS pre-pass]

Pass 2 (6.0): Downloaded dotnet-sdk-6.0.428-win-x64.zip (265,214,223 bytes ~265 MB) → Installed 6.0.428
Pass 2 (7.0): Downloaded dotnet-sdk-7.0.410-win-x64.zip (288,313,762 bytes ~288 MB) → Installed 7.0.410

AFTER (DOTNET_ROOT = C:\Program Files\dotnet):
  SDKs at DOTNET_ROOT : 6.0.428, 7.0.410
  Runtimes            : NETCore+AspNetCore+WinDesktop 6.0.36, 7.0.20
                        NO 10.0.10 runtime  ← Pass 1 skipped ✅
  host\fxr            : 6.0.36, 7.0.20     ← NO 10.0.10 ✅  highest = 7.0.20
  Host: Version       : 7.0.20  Architecture: x64
                        Commit: 0fb6ac59fb

  Muxer binary        : C:\Program Files\dotnet\dotnet.exe
  Size                : 139,536 bytes       ← 6.0 vintage muxer (smaller than 10.0 build)
  ProductVersion      : 6.0.36 @Commit: f1dd57165bfd91875761329ac3a8b17f6606ad18
  FileVersion         : 6,0,3624,51421 @Commit: f1dd57165bfd91875761329ac3a8b17f6606ad18
  SHA256              : D4401F5FBDEA869BB7211B00594746EF6962B7DD2BFCEB749889B920070C3F9F
```

**Side-by-side comparison (self-hosted Windows x64, clean slate, EOL+current):**

| | `v4` ([job 91977578375](https://github.com/chiranjib-swain/test-setup-dotnet/actions/runs/30904891459/job/91977578375)) | `issue_642` ([job 91977578329](https://github.com/chiranjib-swain/test-setup-dotnet/actions/runs/30904891459/job/91977578329)) |
|---|---|---|
| Pass 1 (LTS pre-pass) | ✅ Ran — downloaded LTS 10.0.10 runtime | ❌ Skipped |
| SDKs in DOTNET_ROOT | 6.0.428, 7.0.410 | 6.0.428, 7.0.410 |
| `host\fxr\` entries | **10.0.10**, 6.0.36, 7.0.20 | 6.0.36, 7.0.20 |
| Highest hostfxr loaded | **10.0.10** | **7.0.20** |
| `Host: Version` | 10.0.10 | 7.0.20 |
| Muxer `ProductVersion` | **10.0.10** (167,208 bytes) | **6.0.36** (139,536 bytes) |
| Muxer SHA256 | `4377F10C...` | `D4401F5F...` (different binary) |
| LTS runtime `10.0.10` present | ✅ Yes | ❌ No |

**Key findings from this run:**

1. **Old muxer loads newer hostfxr** — issue_642 writes the 6.0.36 muxer binary (139 KB, 6.0 vintage).
   That binary successfully loads `host\fxr\7.0.20\hostfxr.dll` (the highest available), proving
   the muxer does not need to match or exceed the hostfxr version.

2. **Pass 1 purpose confirmed on clean slate** — On v4, Pass 1 installs LTS 10.0.10 runtime
   BEFORE any SDK pass. This writes the 10.0.10 muxer binary AND `host\fxr\10.0.10\`.
   Without it (issue_642), the muxer is whatever the first SDK installer writes (6.0.36 here).

3. **Tool cache interaction on persistent self-hosted runners** — Cleaning DOTNET_ROOT is not
   a true clean slate on self-hosted runners because the action's tool cache is separate.
   Versions already in the tool cache are not re-extracted to DOTNET_ROOT after a clean slate.
   For a genuine clean slate on Windows, the tool cache must also be cleared:
   `Remove-Item -Recurse -Force "C:\Windows\System32\actions-runner\_work\_tool\dotnet*"`

---


