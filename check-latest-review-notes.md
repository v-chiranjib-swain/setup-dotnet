# Review notes: `check-latest` local SDK reuse PR

**PR under review:** https://github.com/actions/setup-dotnet/pull/774
**Related issue:** https://github.com/actions/setup-dotnet/issues/762 (setup-dotnet fails on air-gapped runners for floating versions)

## Summary of the change

Adds a `check-latest` input (default `true`, non-breaking) to `actions/setup-dotnet`.
When `false`, `DotnetCoreInstaller` first scans the local `sdk/` install directory for
an already-installed SDK satisfying the requested version and skips all network calls
(including the unconditional runtime pre-install) if one is found. Also adds a
`DOTNET_CHECK_LATEST` environment variable fallback (input → env var → default `true`),
so GitHub-generated workflows that can't be edited (e.g. Automatic Dependency
Submission) can still opt in. `global.json` `rollForward` handling tracks a
`minimumVersion` floor so a reused local SDK never violates the declared minimum.

Key files: `src/installer.ts` (`DotnetCoreInstaller.findLocalSdkVersion`,
`getInstalledSdkVersions`, `hasDotnetMuxer`, `qualityApplies`), `src/setup-dotnet.ts`
(`getCheckLatestInput`, `getVersionFromGlobalJson`), `action.yml`, `README.md`,
`__tests__/installer.test.ts`, `__tests__/setup-dotnet.test.ts`,
`.github/workflows/e2e-tests.yml` (new `test-check-latest-false` job).

## Background concepts (for context)

- **Air-gapped runner**: self-hosted runner with no outbound internet at all.
- **Floating/channel version** (`8.0.x`, `8.0`, `8`, `latest`) requires resolving an
  exact version online (`latest.version` file), unlike a **pinned version**
  (`8.0.404`).
- Two network calls happen unconditionally today, before any local check: (1) the
  runtime pre-install always resolves `Runtime/LTS/latest.version`, (2) floating/channel
  requests resolve `Sdk/<channel>/latest.version`. Both fail hard offline.
- `setup-dotnet` does **not** use `@actions/tool-cache` (unlike `setup-java`), so
  preloading the tool cache is not a workaround here.
- `check-latest` mirrors the existing convention from `setup-node`/`setup-python`.

## Local validation performed

Checked out the PR branch from upstream, ran against `main` locally:

| Check | Result |
|---|---|
| `tsc --noEmit` | clean |
| `eslint src __tests__` | clean |
| `npm run build` (ncc) | output identical to committed `dist/` |
| `npm test` (full suite) | **201/201 tests pass** (30 new for `check-latest`) |

## Live end-to-end testing (synthetic fixture + simulated air-gapped network)

Built a synthetic `DOTNET_INSTALL_DIR` fixture (fake `dotnet` muxer + `sdk/<version>/dotnet.dll`
files) and ran the real compiled `dist/setup/index.js` directly with `INPUT_*` /
`GITHUB_*` env vars, using `HTTPS_PROXY=http://127.0.0.1:1` (unreachable) to simulate
an air-gapped network. Nothing here touched the real `~/.dotnet` install.

| # | Scenario | Result |
|---|---|---|
| 1 | `check-latest: false` + version available locally | Reuses in ~0.08s, zero network calls |
| 2 | `check-latest: true` (default) + same version + blocked network | **Fails**, reproducing the exact log from issue #762 (2 min of retries) |
| 3 | `DOTNET_CHECK_LATEST=false` env var, no input at all (ADS scenario) | Reuses correctly |
| 4 | `DOTNET_CHECK_LATEST=yes` (invalid value) | Warns exactly as documented, falls back to `true`, then fails offline |
| 5 | `global.json rollForward: latestFeature`, floor `8.0.500` > local `8.0.423` | Correctly **rejects** the too-old local SDK, falls back online |
| 6 | Same floor lowered to `8.0.100` (≤ local `8.0.423`) | Correctly **reuses** `8.0.423` |
| 7 | Cross-architecture request (`x64` on `arm64` host) | Bypasses local reuse entirely, always goes online |
| 8 | Feature-band matching (`8.0.1xx` among `8.0.100`/`8.0.105`/`8.0.203`) | Correctly picks `8.0.105`, not the higher `8.0.203` (wrong band) |
| 9 | Orphaned SDK folder (dir exists, no `dotnet.dll`) | Ignored, logged as `<none>` locally installed |
| 10 | Symlinked SDK folder | Detected and reused just like a real directory |

## Two minor nitpicks identified and empirically verified

1. **`src/setup-dotnet.ts`, `getVersionFromGlobalJson` — `rollForward: latestMajor`**
   Produces `{version: '', minimumVersion: <sdk.version>}` (verified by running the
   function standalone). The `minimumVersion` is unreachable: `version=''` never
   matches anything in `findLocalSdkVersion()`, so it always falls to the online path
   regardless of what's installed locally — confirmed live with a local SDK well above
   the floor still triggering an online install attempt. Harmless, but worth a comment
   or skipping the computation for this branch.

2. **`src/installer.ts`, `qualityApplies()` — bare `latest` with no `dotnet-channel`**
   Returns `true` unconditionally (no major digit to check), whereas the online
   resolver derives its `qualityFlag` from the actual *resolved* LTS/STS version's
   major. Verified live: with a local GA and a local preview SDK, requesting
   `latest` + `dotnet-quality: preview` + `check-latest: false` correctly reused the
   preview build. Correct today (all supported LTS/STS channels are ≥ .NET 6), but
   the two code paths check different things — worth a comment noting the assumption.

Both are **non-blocking** nitpicks; overall recommendation: **Approve with minor
comments**.

### Drafted review comment text

> **Comment 1 — `src/setup-dotnet.ts`, `getVersionFromGlobalJson`:**
> Nit (verified locally): For `rollForward: latestMajor`, `version` is set to `''`,
> but `minimumVersion` is still computed as `globalJson.sdk.version`. Confirmed this
> is dead weight — `findLocalSdkVersion()` can never match an empty version spec, so
> `minimumVersion` is computed but never consulted for this branch. Consider skipping
> the computation for `latestMajor` or adding a one-line comment.
>
> **Comment 2 — `src/installer.ts`, `qualityApplies()`:**
> Nit (verified locally): For `dotnet-version: latest` with no `dotnet-channel`,
> `qualityApplies()` returns `true` unconditionally. Verified this is exercised in
> practice and correct today (LTS/STS channels are all ≥ .NET 6), but it's a
> different code path than the online resolver's `qualityFlag`. Worth a short
> comment noting the assumption so it doesn't silently drift.

## Real CI validation (not just local simulation)

Since the PR's own `.github/workflows/e2e-tests.yml` added a `test-check-latest-false`
job matrixed across `ubuntu-latest`/`windows-latest`/`macos-latest`, replicated that
exact scenario for real:

1. Renamed the local review branch to `feature/check-latest-local-sdk-reuse` (no PR
   number in the name, per convention) and pushed it to the user's `setup-dotnet` fork.
   - Discovered during push that the fork account has migrated to
     **`v-chiranjib-swain`** (GitHub auto-redirected); memory updated with this fact.
2. Added a `workflow_dispatch`-only workflow
   (`.github/workflows/check-latest-local-reuse-test.yml`) to the separate
   `test-setup-dotnet` repo, referencing
   `v-chiranjib-swain/setup-dotnet@feature/check-latest-local-sdk-reuse`.
3. Dispatched via `gh workflow run` — first run failed on all 3 OSes due to a harness
   bug (`verify-dotnet.ps1`'s relative path assumed running from the setup-dotnet repo
   root; fixed by adding `working-directory: setup-dotnet-src` to the verify step).
   Note: even in the failing run, the underlying fix was already proven correct — logs
   showed `Found installed versions: 9.0.100` (single version, no re-download) and the
   dotnet-version output check passed; only the unrelated csproj path lookup failed.
4. Re-dispatched after the fix: **all 3 platforms passed**
   (run: https://github.com/v-chiranjib-swain/test-setup-dotnet/actions/runs/34971743697).

## Outstanding / cleanup

- Local repo (`~/Desktop/setup-dotnet`) is on branch `feature/check-latest-local-sdk-reuse`.
- Fork branch `feature/check-latest-local-sdk-reuse` pushed to
  `v-chiranjib-swain/setup-dotnet`.
- Test workflow `.github/workflows/check-latest-local-reuse-test.yml` added to
  `v-chiranjib-swain/test-setup-dotnet` (`main`, 2 commits).
- Decide whether to delete the branch/workflow now or keep them for future re-runs.
