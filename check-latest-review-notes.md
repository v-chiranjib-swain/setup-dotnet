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
>
> **Comment 3 — `README.md` / `action.yml`, `check-latest: false` docs:**
> Non-blocking, documentation-only: `check-latest: false` reuses whatever's under
> `sdk/<version>/` based purely on folder name + presence of `dotnet.dll` — there's
> no hash/signature verification that the reused SDK is an unmodified, legitimate
> build. That's an inherent and reasonable trade-off for the air-gapped use case
> (mirrors `setup-node`/`setup-python`'s existing `check-latest: false` semantics),
> but the README doesn't currently say so explicitly. Given `DOTNET_CHECK_LATEST`
> can also be set at the self-hosted runner level (affecting every job scheduled on
> that runner, as the README's own "a dedicated runner is recommended" note already
> hints at), it'd be worth adding a one-line callout: only enable this on runners
> whose pre-installed SDKs/base image you already trust, since no integrity check is
> performed on the reused SDK.

## Security review

Walked the diff for OWASP-Top-10-style issues: no injection, auth-bypass, SSRF, or
deserialization concerns found.

- All install-script invocations still go through `exec.getExecOutput` with an
  **argument array**, not shell-string interpolation — unchanged from before, no new
  command-injection surface.
- New regexes (`FeatureBandSyntax`, major/minor/channel matchers) are simple, linear,
  non-backtracking patterns over short bounded strings — no ReDoS risk.
- No new network calls, URLs, or credential handling introduced.
- **Trust-model shift (real, but inherent to the feature, see Comment 3 above):**
  `check-latest: false` trusts a local SDK folder based only on name + presence of
  `dotnet.dll`, with no signature/hash verification — a deliberate trade-off for the
  air-gapped scenario, consistent with `setup-node`/`setup-python` precedent.
- **Blast radius of `DOTNET_CHECK_LATEST`:** a runner-level env var affects every job
  on a shared self-hosted runner, not just the opting-in workflow; the PR's README
  already recommends a dedicated runner for this.
- Symlinked SDK folder handling has a theoretical check-then-use (TOCTOU) gap, but it
  requires an attacker who already has local filesystem write access to the runner —
  not a meaningfully new privilege-escalation vector.
- **Positive:** the feature deliberately avoids `@actions/tool-cache`/`actions/cache`,
  sidestepping the known cross-branch/cross-workflow cache-poisoning attack class
  entirely — reuse is scoped strictly to the runner's own local disk.

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

## Real CI validation of all edge cases (all 10 local scenarios, re-verified live)

Added a second `workflow_dispatch`-only workflow
(`.github/workflows/check-latest-edge-cases-test.yml`) to `test-setup-dotnet`,
replicating every scenario proven locally, as real steps against
`v-chiranjib-swain/setup-dotnet@feature/check-latest-local-sdk-reuse` on actual
GitHub-hosted runners. Each step builds a synthetic `DOTNET_INSTALL_DIR` fixture and,
where relevant, sets `https_proxy`/`http_proxy` to an unreachable address
(`http://127.0.0.1:1`) to force a real air-gapped condition; assertions check either
the `dotnet-version` output (for expected-success cases) or the step's `outcome` via
`continue-on-error: true` (for expected-failure/fallback cases).

Run: https://github.com/v-chiranjib-swain/test-setup-dotnet/actions/runs/34973400817
— **all steps passed** across 2 jobs (`edge-cases-ubuntu`, 10m14s; `edge-cross-arch-macos`, 2m9s).

| Scenario | Step | Result |
|---|---|---|
| A | `check-latest: false` reuses local SDK offline | ✓ |
| B | `check-latest` default `true` ignores local SDK, fails offline | ✓ |
| C | `DOTNET_CHECK_LATEST` env var alone triggers reuse | ✓ |
| D | Invalid `DOTNET_CHECK_LATEST=yes` warns (message confirmed verbatim in run annotations) + falls back to `true` + fails offline | ✓ |
| E | `global.json` floor above local SDK → rejected, falls online, fails offline | ✓ |
| F | `global.json` floor at/below local SDK → reused | ✓ |
| G | Cross-arch (`arm64` on ubuntu x64) bypasses local reuse, fails offline | ✓ |
| G (macOS) | Cross-arch (`x64` on macOS arm64) bypasses local reuse, fails offline | ✓ (separate job) |
| H | Feature-band matching picks `8.0.105` (1xx band), not `8.0.203` (2xx band) | ✓ |
| I | Orphaned SDK folder (no `dotnet.dll`) ignored, falls online, fails offline | ✓ |
| J | Symlinked SDK folder recognized and reused | ✓ |

The live run's annotations captured the exact documented warning text verbatim:
`Value 'yes' is not supported for the DOTNET_CHECK_LATEST environment variable.
Supported values are: true, false. The 'check-latest' option falls back to 'true'.`
— matching the code and the earlier local test byte-for-byte.

## Outstanding / cleanup

- Local repo (`~/Desktop/setup-dotnet`) is on branch `feature/check-latest-local-sdk-reuse`.
- Fork branch `feature/check-latest-local-sdk-reuse` pushed to
  `v-chiranjib-swain/setup-dotnet`.
- Test workflows added to `v-chiranjib-swain/test-setup-dotnet` (`main`, 4 commits):
  - `.github/workflows/check-latest-local-reuse-test.yml`
  - `.github/workflows/check-latest-edge-cases-test.yml`
- Decide whether to delete the branch/workflows now or keep them for future re-runs.
