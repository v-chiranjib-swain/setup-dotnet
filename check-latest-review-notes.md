# Review notes: `check-latest` local SDK reuse PR

**PR under review:** https://github.com/actions/setup-dotnet/pull/774
**Related issue:** https://github.com/actions/setup-dotnet/issues/762 (setup-dotnet fails on air-gapped runners for floating versions)

## Final review summary (share with team)

**Verdict: Approve with minor comments.**

Validated locally (build/lint/type-check clean, 201/201 unit tests) and live —
both against a synthetic offline fixture and on real GitHub-hosted runners
(ubuntu/windows/macos) — across all 10 documented and edge-case scenarios
(local-SDK reuse, `DOTNET_CHECK_LATEST` env fallback, `global.json` floor
enforcement, cross-arch bypass, feature-band matching, orphaned/symlinked SDK
folders), plus a full live matrix against all 9 official `global.json`
`rollForward` values. No blocking issues found.

**Superseded finding — no longer being raised:** the originally-drafted "guard
`minimumVersion` with `version` being non-empty" comment (for `rollForward:
latestMajor`) is **withdrawn**. Deeper investigation (see "Deeper finding"
below) showed that `minimumVersion` isn't actually dead weight to remove —
it's exactly the piece of data a *correct* implementation of `latestMajor`
would need. Suppressing it treats the symptom (a misleading debug line)
rather than the real gap (local reuse for `latestMajor` isn't implemented at
all). Raising the real gap instead is more useful to the author than asking
for a guard that would need to be reverted if they ever implement `latestMajor`
properly.

> **Deeper finding — `latestMajor` (and related `latestMinor`) local reuse gap:**
>
> Per the [official `global.json` docs](https://learn.microsoft.com/en-us/dotnet/core/tools/global-json#rollforward),
> `rollForward: latestMajor` should: *"Use the highest installed .NET SDK with
> a version that's greater than or equal to the specified value. If not found,
> fail."* That means **any major** ≥ the declared floor should be reusable
> offline (e.g. a declared `7.0.200` should happily reuse a locally installed
> `9.0.423`).
>
> Verified live (https://github.com/v-chiranjib-swain/test-setup-dotnet/actions/runs/35339385300)
> that this is **not** what happens: `check-latest: false` never attempts local
> reuse for `latestMajor` at all — `version` is unconditionally cleared to `''`
> in `getVersionFromGlobalJson`, and `''` never matches any pattern
> `findLocalSdkVersion()` checks, so it always falls back online regardless of
> what's installed, even when a local SDK clearly satisfies the documented rule.
>
> A related gap was also found in `latestMinor`: even though it *is*
> implemented, the widened spec (`version = major`) resolves through
> `channelForMajor()` to `major.0`, hardcoding `minor = 0` — so a genuinely
> higher-minor SDK that satisfies the official "any minor ≥ floor" rule (e.g.
> `8.1.100` for a declared `8.0.100`) still isn't reused locally.
>
> Suggested framing for the PR author: implementing proper local reuse for
> `latestMajor` (and fixing the `latestMinor` minor-constraint) would actually
> need `minimumVersion` to be set and consulted for `latestMajor` — the
> opposite of removing it — by adding a branch to `findLocalSdkVersion()` that,
> for an empty `version` with a `minimumVersion` present, picks the highest
> installed SDK (any major) that's `>= minimumVersion`.

The one remaining non-blocking comment:

> **Could we add a short comment explaining why `qualityApplies()` returns `true`
> when the major version is unknown?**
>
> For `dotnet-version: latest` without `dotnet-channel`, the major version cannot be
> determined locally. Returning `true` is intentional so that `preview`/`daily`
> quality can still be honored when matching a locally installed SDK.
>
> Something like:
>
> ```ts
> /**
>  * For bare 'latest' without a channel, the major version is unknown locally.
>  * Default to true so preview/daily quality can be honored for local SDKs.
>  */
> ```

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

## Code review findings

1. **`src/setup-dotnet.ts`, `getVersionFromGlobalJson` — `rollForward: latestMajor`**
   (superseded — see "Deeper finding" at the top; withdrawn as a comment to send)
   Produces `{version: '', minimumVersion: <sdk.version>}` (verified by running the
   function standalone). Initially flagged as dead weight, since `version=''` never
   matches anything in `findLocalSdkVersion()`. Further investigation showed this
   framing was backwards: `minimumVersion` isn't dead weight to remove — it's exactly
   what a *correct* `latestMajor` local-reuse implementation would need to consult.
   The real problem is that `findLocalSdkVersion()` never attempts local reuse for
   `latestMajor` at all, regardless of the floor. Suppressing `minimumVersion` treats
   the symptom (a misleading debug line), not the actual missing capability.

2. **`src/installer.ts`, `qualityApplies()` — bare `latest` with no `dotnet-channel`**
   (investigated, concluded **not** worth raising — see below)
   Initially flagged because it returns `true` unconditionally (no major digit to
   check) while the online resolver derives its `qualityFlag` from the actual
   *resolved* channel's major. Further tracing showed this isn't just "correct today,"
   it's **functionally required**: `findLocalSdkVersion()` computes
   `wantsPrerelease = ['preview','daily'].includes(quality) && qualityApplies()`. If
   `qualityApplies()` returned `false` for this case instead, `wantsPrerelease` would
   always be `false` for bare `latest` with no channel — meaning
   `check-latest: false` + `dotnet-version: latest` + `dotnet-quality: preview` would
   **never** match a local prerelease SDK, silently breaking the very case it's meant
   to support. So `true` isn't a convenient default, it's the only value that keeps
   this combination working at all. (It also happens to agree with the real online
   resolver today — verified live against the actual `releases-index.json`: current
   non-EOL channels are `11.0`/`10.0`/`9.0`/`8.0`, all major ≥ 6, and every channel
   below major 6 is already permanently `eol` — but that agreement is a bonus, not
   the reason this is correct.)

Point 1 is withdrawn (see "Deeper finding" — the real gap is worth raising instead of
the guard); point 2 was investigated and dropped after tracing its actual effect on
`wantsPrerelease`.

### Local code changes: historical, not part of the current recommendation

The `version && ` guard was applied and verified locally in
[src/setup-dotnet.ts](src/setup-dotnet.ts) `getVersionFromGlobalJson` earlier in this
review (201/201 tests pass, lint/type-check clean), and the `qualityApplies()` doc
comment in [src/installer.ts](src/installer.ts) was tightened. Both remain in the
local checkout / fork branch as an artifact of the investigation, but **the guard is
no longer being recommended to the PR author** now that the deeper `latestMajor` gap
is the more useful thing to raise. The `qualityApplies()` comment tightening still
stands on its own merits (documentation-only, no behavior change) but isn't one of
the comments being sent either — see "Final review summary" at the top for what's
actually being sent.

### Final comments to send to the PR author

See the **"Final review summary"** section at the top of this document for the
finalized wording (the `latestMajor`/`latestMinor` gap finding plus the
`qualityApplies()` comment request). The previously-drafted "guard `minimumVersion`"
comment shown in earlier revisions of this document has been withdrawn.

### Optional additional note (not one of the final comments, raise only if asked for more)

> **`README.md` / `action.yml`, `check-latest: false` docs:**
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
- **Trust-model shift (real, but inherent to the feature, see the optional note above):**
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

## Deeper investigation: `rollForward` matrix vs. official semantics

Followed up on the withdrawn "guard `minimumVersion`" comment by checking `latestMajor`
against the actual documented contract at
https://learn.microsoft.com/en-us/dotnet/core/tools/global-json#rollforward, then
tested **all 9** official `rollForward` values live in one workflow
(`.github/workflows/global-json-rollforward-matrix-test.yml` on `test-setup-dotnet`).

**First run** (against our own fork's `efe6a36` commit — includes the withdrawn,
now-historical `minimumVersion` guard fix):
https://github.com/v-chiranjib-swain/test-setup-dotnet/actions/runs/35339385300
— all 9 jobs passed their assertions.

**Re-run against the real, unmodified PR source** — `v-mahabaleshwars/setup-dotnet@feature/762-check-latest-input`
(the actual PR author's branch, not our fork), to rule out any doubt that the
guard fix (or any other fork-only commit) influenced the results:
https://github.com/v-chiranjib-swain/test-setup-dotnet/actions/runs/35342585139
— **identical outcome**, all 8 jobs (`latestPatch`, `latestFeature`, `latestMinor`,
`latestMinor-higher-minor-gap`, `latestMajor`, `disable`, `disable-no-exact-match`,
`legacy-patch-value-pinned-behavior`) completed successfully, same gap findings
confirmed against the literal PR-author's branch.

| `rollForward` | Official semantics | Fixture | Result |
|---|---|---|---|
| `latestPatch` | Latest patch in same major.minor.band, ≥ floor | `8.0.100` (below), `8.0.199` (above, same band) | ✅ Correctly reused `8.0.199` |
| `latestFeature` | Highest feature band+patch in same major.minor, ≥ floor | `8.0.300` (below), `8.0.402` (above, different band) | ✅ Correctly reused `8.0.402` |
| `latestMinor` | Highest minor+band+patch in same major, ≥ floor | `8.0.050` (below), `8.0.200` (above, same minor) | ✅ Correctly reused `8.0.200` |
| `latestMinor` (gap check) | Same rule — a *higher minor* also satisfies it | only `8.1.100` installed | ❌ **Gap**: not reused (`channelForMajor()` hardcodes `minor=0`) |
| `latestMajor` | Highest installed SDK, **any major**, ≥ floor | `9.0.423` (higher major, satisfies floor) | ❌ **Gap**: local reuse never attempted at all |
| `disable` | Exact match only | exact `8.0.302` installed | ✅ Correctly reused the exact match |
| `disable` (negative) | Exact match only — must reject substitutes | only `8.0.303` installed | ✅ Correctly refused to substitute |
| `patch`/`feature`/`minor`/`major` (legacy) | Should roll forward if exact missing | only `8.0.105` installed, declared `8.0.100` | ❌ **Pre-existing gap** (not from this PR): always treated as pinned-exact |

Confirms the `latestMajor` finding against the literal doc wording, and surfaces one
additional related gap (`latestMinor`'s hardcoded `minor=0`) plus one pre-existing,
out-of-scope-for-this-PR gap (the four legacy non-`latest`-prefixed values). Verified
identically against both our fork commit and the real PR author's branch, so none of
the findings are artifacts of our own fork's extra commits.

## Outstanding / cleanup

- Local repo (`~/Desktop/setup-dotnet`) is on branch `feature/check-latest-local-sdk-reuse`.
- Fork branch `feature/check-latest-local-sdk-reuse` pushed to
  `v-chiranjib-swain/setup-dotnet`, including the (now-withdrawn-as-a-comment,
  historical) `minimumVersion` guard fix and rebuilt `dist/`.
- Test workflows added to `v-chiranjib-swain/test-setup-dotnet` (`main`):
  - `.github/workflows/check-latest-local-reuse-test.yml`
  - `.github/workflows/check-latest-edge-cases-test.yml`
  - `.github/workflows/check-latest-latestmajor-fix-diff-test.yml`
  - `.github/workflows/global-json-rollforward-matrix-test.yml`
  - Repo secret `ACTIONS_STEP_DEBUG=true` set to enable debug-level log capture.
- Decide whether to delete the branch/workflows now or keep them for future re-runs.

