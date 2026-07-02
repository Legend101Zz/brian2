# cppyy cross-platform validation via GitHub Actions

**Status:** active validation plan
**Date:** 2026-07-02
**Branch:** `perf/v3-investigation` (fork `Legend101Zz/brian2`, PR [#1769](https://github.com/brian-team/brian2/pull/1769))
**Workflow:** `.github/workflows/cppyy-cross-platform.yml`
**Smoke script:** `dev/continuous-integration/cppyy_smoke_test.py`

## Purpose

Produce hard, per-OS evidence for the PR discussion: either the cppyy backend
works on Linux, macOS and Windows, or it fails on a specific OS at a specific,
named layer. GitHub Actions is the test machine; nothing here depends on a
local Windows setup.

## What the workflow tests (per OS, hard-fail stages)

| Stage | Step | Proves |
| --- | --- | --- |
| 0 | environment/toolchain report | which compiler, Python, env vars the job saw |
| — | `pip install .[test]` | Brian2 itself installs |
| — | cppyy install | Linux/macOS: plain `pip install cppyy>=3.1`. Windows: wlav/cppyy#282 layered `--no-deps --no-build-isolation` workaround (pinned: cppyy-cling 6.32.8, cppyy-backend 1.15.3, CPyCppyy 1.13.0, cppyy 3.5.0), after `ilammy/msvc-dev-cmd` puts MSVC on PATH |
| 1 | bare cppyy JIT smoke | cppyy can `cppdef` + run C++ with no Brian2 involved |
| 2 | Brian2 smoke simulation | the cppyy backend registers, generates + JIT-compiles code, and produces numerically correct results (analytic decay + spike/synapse invariant); asserts the code object really is `CppyyCodeObject` |
| 3 | `brian2.test(['cppyy'], ...)` | the full runtime test-suite subset passes with target=cppyy (no long/GSL/codegen-independent tests) |

Matrix: `ubuntu-latest`, `macos-latest`, `windows-2022` (primary Windows
evidence — VS 2022, the toolchain the #282 workaround was verified on), and
`windows-latest` (Server 2025 + VS 2026 since June 2026; forward-compatibility
probe). Python 3.12 everywhere — supported by Brian2 and the exact version the
#282 workaround was verified on. `fail-fast: false`, so every OS reports.

Stage 2 was verified locally on macOS (arm64, cppyy 3.5.0) before pushing:
`CPPYY_SMOKE_RESULT: PASS`.

## How to run it

1. **Automatic:** pushing a commit that touches the workflow or the smoke
   script triggers it (`push.paths` filter). The initial push of this branch
   already ran it.
2. **Manual re-run:** GitHub → fork → *Actions* → *cppyy cross-platform
   validation* → *Run workflow* → branch `perf/v3-investigation`.
   (The workflow appears in the Actions sidebar after its first run even
   though it is not on the fork's default branch.)
3. **CLI:** `gh workflow run cppyy-cross-platform.yml --ref perf/v3-investigation`
   (requires `gh auth login`).

Note: pushing also triggers the regular `TestSuite` workflow (its normal
`on: push` behavior); that run can be cancelled from the Actions tab if you
only want the cppyy matrix.

## How to interpret results

- **All four jobs green** → cppyy backend proven cross-platform; the PR can
  drop the "Windows unsupported" caveat in favor of documenting the install
  workaround.
- **Linux/macOS green, both Windows jobs fail at the cppyy-install step** →
  wlav/cppyy#282 workaround no longer sufficient; the failure log shows the
  exact pip/link error. Windows stays documented as unsupported, with this
  run as evidence.
- **windows-2022 green, windows-latest red** → cppyy works on Windows with
  VS 2022 but not the VS 2026 image; scope the support statement accordingly.
- **Windows install green, Stage 1 red** → cppyy installed but Cling cannot
  JIT (likely DLL load / toolchain env issue) — upstream problem, not Brian2.
- **Stages 1 green, Stage 2 red** → first genuinely Brian2-owned failure
  layer: our generated C++ or backend glue is not MSVC/Windows-clean. The
  smoke script's `CPPYY_SMOKE_FAIL:` marker names the failing part.
- **Stage 2 green, Stage 3 red** → backend fundamentally works on Windows;
  remaining failures are individual test issues (read the pytest tail).

## Suspected Windows failure modes (ranked)

1. **Install-time link error** (LNK2001 `GetNumBasesLongestBranch`) if the
   workaround stopped matching current PyPI artifacts — the reason the four
   versions are pinned.
2. **`std::complex` / template specializations** — macOS already needed
   `long` template fixes (commit e29a0e81); MSVC may need analogous ones.
3. **Cling JIT + MSVC runtime quirks** (exception-handling flags `/EHc`,
   DLL resolution) — would surface in Stage 1, i.e. upstream, not ours.
4. **Long-double / integer-width differences** (`long` is 32-bit on Win64) —
   would surface as numerical test failures in Stage 3.

## Next fix candidates once evidence is in

- Windows green → port the install workaround into `testsuite.yml`'s Windows
  jobs (a prepared, uncommitted version of that change already exists locally)
  and document the user-facing install recipe in the docs.
- Windows red at a Brian2-owned layer → fix the named layer (Stage 2/3 logs
  point at the generated code or test involved) and re-dispatch.
- Windows red at install/JIT → keep "Linux/macOS only" position in the PR,
  attach the failing job link as proof, track wlav/cppyy#282.

## Related local files not committed with this change

- `.github/workflows/cppyy_windows_smoke.yml` (earlier Windows-only draft) —
  superseded by the cross-platform workflow; safe to delete.
- Modified `.github/workflows/testsuite.yml` (Windows cppyy install in the
  main suite) — intentionally held back until this workflow proves the
  approach; commit it as the follow-up if Windows comes back green.
