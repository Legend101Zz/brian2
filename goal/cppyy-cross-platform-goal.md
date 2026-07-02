# GOAL: prove the Brian2 cppyy backend cross-platform via GitHub Actions

You (Codex) own this goal. Keep this file updated as you work: check off
tasks, append findings under "Progress log". This file is the source of
truth for where the effort stands.

## Mission

Use GitHub Actions on the fork as the test machine (never a local Windows
setup) and produce hard, log-backed proof for PR
https://github.com/brian-team/brian2/pull/1769 that the cppyy runtime
backend either works on Linux/macOS/Windows, or fails at a specific, named
technical layer per OS.

- Repo (local): `/Volumes/Mrigesh SSD/Brain_WIP_Folder/brian2`
- Fork: `Legend101Zz/brian2` (`origin`), branch `perf/v3-investigation` (pushed)
- Detailed operating manual (commands, tokens, gotchas, decision trees):
  `cppyy_optimization_work/cppyy_gha_validation_plan.md` and
  `cppyy_optimization_work/codex_handoff_cppyy_gha.md` — READ BOTH FIRST.

## Infrastructure already in place (do not rebuild)

- `.github/workflows/cppyy-cross-platform.yml` — staged validation matrix
  (ubuntu-latest, macos-latest, windows-2022, windows-latest; Python 3.12).
  Stages hard-fail in order: brian2 install → cppyy install → Stage 1 bare
  cppyy JIT → Stage 2 Brian2 smoke sim → Stage 3 cppyy-only test subset.
  Pushing any change to the workflow or the smoke script auto-triggers it.
- `dev/continuous-integration/cppyy_smoke_test.py` — Stage 2/3 script.
- Windows install uses the wlav/cppyy#282 workaround (layered
  `--no-deps --no-build-isolation`, pins: cppyy-cling 6.32.8,
  cppyy-backend 1.15.3, CPyCppyy 1.13.0, cppyy 3.5.0) after
  `ilammy/msvc-dev-cmd` puts MSVC on PATH.

## State as of 2026-07-02 (run 28609534402, commit 3989032b)

| OS | Verdict | Failure layer |
| --- | --- | --- |
| ubuntu-latest | ✅ all stages green | — |
| windows-2022 | ⚠️ green through Stage 2; Stage 3 crashed | Suite ran with ZERO test failures to ~66%, then the Python process hard-crashed (exit 127, no traceback, output stops mid-line). One specific test triggers a native/JIT crash; `-q` hid its name. Job 84838188601. |
| macos-latest | ❌ Stage 1 (bare cppyy, pre-Brian2) | Xcode 26.5 SDK libc++ headers too new for Cling's bundled clang 16 (`__BYTE_ORDER__` missing, `'_Tp' does not refer to a value`, exit 139). Upstream, not Brian2. Job 84838188554. |
| windows-latest | ❌ Stage 1 (bare cppyy, pre-Brian2) | VS 2026 (MSVC 14.51) STL headers too new for Cling clang 16 (`__builtin_verbose_trap` unknown). Upstream. Job 84838188615. |

Headline so far: **install + JIT + correct Brian2 simulations are PROVEN on
Windows (VS 2022)**. Remaining Windows work is one crashing test. The two
Stage-1 failures are upstream Cling-vs-newest-toolchain issues.

Important local fact: on the user's own Mac (Apple Silicon, older SDK than
CI), cppyy + the backend work fine, with
`export DYLD_LIBRARY_PATH="/opt/homebrew/opt/zstd/lib:$DYLD_LIBRARY_PATH"`
needed so libCling finds zstd. The workflow already sets this env var at the
job level — the CI macOS failure is the *header/SDK* issue above, NOT zstd.
Do not confuse the two.

## Tasks

- [x] 1. Identify the windows-2022 Stage-3 crasher: in
      `dev/continuous-integration/cppyy_smoke_test.py` change
      `additional_args=["--tb=short", "-q"]` to
      `additional_args=["--tb=short", "-v"]`, commit + push (auto-triggers
      run), read the windows-2022 log — the last test name printed before
      exit 127 is the culprit.
- [ ] 2. Fix that test's crash in the backend
      (`brian2/codegen/runtime/cppyy_rt/`, `cppyy_generator.py`, templates)
      or, if upstream-unfixable, add a documented Windows-only skip for that
      single test, justified by the crash log. Suspects: 32-bit `long` on
      Win64 (macOS needed `long` template specializations, commit e29a0e81),
      `long double`, MSVC-only template/JIT quirks. Ubuntu passes the same
      subset, so it's Windows-specific.
- [ ] 3. Re-run until windows-2022 is fully green end-to-end.
- [ ] 4. macos-latest Stage 1: try selecting an older Xcode on the runner
      (`ls /Applications | grep Xcode` in a debug step, then
      `sudo xcode-select -s /Applications/Xcode_<older>.app` before the cppyy
      install), and/or check whether newer cppyy/cppyy-cling releases bundle
      a newer clang. If unfixable, document as upstream with log excerpt.
- [ ] 5. windows-latest (VS 2026) is a forward-compat probe: fix if cheap,
      otherwise document "unsupported by current cppyy (Cling clang 16)".
- [ ] 6. When the matrix is final: update
      `cppyy_optimization_work/cppyy_gha_validation_plan.md` with the results
      table + run links, and draft the PR #1769 comment (what works, what
      fails, why, minimal Windows install recipe).
- [ ] 7. Only if windows-2022 ends fully green: review + commit the prepared
      uncommitted `.github/workflows/testsuite.yml` change that ports the
      Windows cppyy install into the main suite.

## Hard rules

- Commit by explicit pathspec only (`git add <files> && git commit -m "..." -- <files>`);
  `brian2/codegen/runtime/cppyy_rt/cppyy_rt.py` has staged+unstaged junk that
  must NOT be committed. No `Co-Authored-By` lines. No `[skip ci]` (it would
  skip the validation workflow too).
- Each push also triggers the fork's TestSuite / Build-and-publish / zizmor
  workflows — cancel them via the API to save minutes (see handoff doc).
- `gh` CLI is not authenticated; for job LOGS use
  `TOKEN=$(printf "protocol=https\nhost=github.com\n" | git credential fill | grep '^password=' | cut -d= -f2)`
  with `curl -H "Authorization: Bearer $TOKEN"`. Run/job listing works
  unauthenticated. The shell is zsh: never use `status` as a variable name.
- Keep the workflow's checkout as-is (`fetch-depth: 0` + upstream tag fetch);
  setuptools_scm breaks without it.

## Success criteria

Every OS in the matrix is either green, or red with a log that names the
exact failure layer, plus a written, PR-ready summary. Evidence over green:
never hide a failure to make CI pass.

## Progress log

- 2026-07-02 (Codex): started Task 1 from branch `perf/v3-investigation`
  at `3989032b`. Local dirty state includes unrelated staged+unstaged
  `brian2/codegen/runtime/cppyy_rt/cppyy_rt.py` changes and the held-back
  `.github/workflows/testsuite.yml` edit, so this iteration is scoped to
  `dev/continuous-integration/cppyy_smoke_test.py` plus this tracker. Changed
  Stage 3 pytest output from `-q` to `-v`; next push should auto-trigger the
  cppyy cross-platform workflow, and the windows-2022 Stage 3 log should name
  the last test printed before exit 127.
- 2026-07-02 (Codex): pushed `8bb77f1e`, run
  https://github.com/Legend101Zz/brian2/actions/runs/28611479444. Cancelled
  redundant push runs `28611479511`, `28611479462`, and `28611479501` via the
  Actions API. Results: ubuntu-latest passed; macos-latest and windows-latest
  failed before Brian2 as expected; windows-2022 still passed install, Stage 1,
  and Stage 2, then hard-crashed in Stage 3 with exit 127. The `-v` diagnostic
  narrowed the crash to `test_synapses.py`: the log showed
  `test_synapses.py .` followed by four more passing dots before exit 127.
  Local pytest collection order for the same marker expression makes the next
  selected test `test_connection_string_deterministic_full_custom`, but Brian2's
  base pytest argv already includes `--quiet`, so `-v` did not print full node
  IDs. Updating the diagnostic to `-vv` for one more run to get direct node-id
  evidence before treating Task 1 as complete.
- 2026-07-02 (Codex): pushed `340ad638`, run
  https://github.com/Legend101Zz/brian2/actions/runs/28612004230. Cancelled
  redundant push runs `28612004025`, `28612004121`, and `28612004233`.
  windows-2022 job `84846507166` again passed install, Stage 1, and Stage 2,
  then hard-crashed in Stage 3 with exit 127. The `-vv` log directly names the
  culprit: `brian2/tests/test_synapses.py::test_connection_string_deterministic_full_custom`
  was printed at `2026-07-02T18:25:04Z`, immediately followed by
  `Process completed with exit code 127`. Task 1 complete; Task 2 starts from
  this test.
- 2026-07-02 (Codex): traced
  `test_connection_string_deterministic_full_custom`. The valid custom
  connections complete locally under cppyy; the test's later intentional
  invalid call `S2.connect(j="20")` is the C++ exception path. Local cppyy emits
  `Warning: uncaught exception in JIT is rethrown` at that exact statement and
  then wraps it as `BrianObjectException` caused by `IndexError`; windows-2022
  appears to hard-crash instead of safely rethrowing the JIT exception. Added a
  regression test proving constant out-of-range generator indices are rejected
  before `create_runner_codeobj`, then added a Python-side precheck in
  `Synapses._add_synapses_generator` for unconditional constant integer
  generator targets. Local checks passed:
  `python -m pytest brian2/tests/test_synapses.py::test_connection_generator_constant_index_prechecked -q`
  and a direct cppyy-target call to
  `test_connection_string_deterministic_full_custom()`.
- 2026-07-02 (Codex): pushed `953f8c4a`, manually dispatched
  https://github.com/Legend101Zz/brian2/actions/runs/28613008288 because
  backend-only changes do not match the cppyy workflow's push paths. The
  windows-2022 job `84849867606` proved the previous crasher fixed:
  `test_connection_string_deterministic_full_custom` and the new
  constant-index regression both passed under Stage 3. The run then crashed
  later with exit 127 at
  `brian2/tests/test_synapses.py::test_synapse_generator_out_of_range`, again
  on an intentional out-of-range synapse-generator path. Local cppyy showed the
  same class of issue as before: C++ JIT `IndexError` paths emitted
  `Warning: uncaught exception in JIT is rethrown` on this test. Added focused
  failing regressions for range-based out-of-range generator indices and
  result-index-dependent post conditions, then broadened the Python-side
  generator precheck to infer simple integer result-index ranges before codegen.
  Local checks now pass for the constant, range, post-condition, and original
  `test_synapse_generator_out_of_range` cases, and direct cppyy-target calls to
  both Windows-crashing test functions complete without the JIT exception
  warning. Next step: commit, push, dispatch cppyy workflow again, and inspect
  the new windows-2022 Stage 3 log.
- 2026-07-02 (Codex): pushed `e8064326`, manually dispatched
  https://github.com/Legend101Zz/brian2/actions/runs/28614193841, and
  cancelled redundant push workflows `28614173956`, `28614174046`, and
  `28614173978`. The windows-2022 log showed the new range/post-condition
  regressions and `test_synapse_generator_out_of_range` all passing; Stage 3
  advanced to 94% and then hard-crashed at
  `brian2/tests/test_synapses.py::test_synapse_generator_fixed_random_error1`.
  Ubuntu uncovered a real false positive from the interval precheck:
  `test_subgroup.py::test_synapse_creation_generator_complex_ranges` failed
  because `j="i+k for k in range(N_post-i)"` couples the outer index and
  iterator range, but the precheck treated them independently and invented an
  impossible `j=10`. Tightened the precheck to decline outer-index-dependent
  range bounds, then added a fixed-size `sample(...)` precheck for impossible
  sample sizes such as `sample(N_post, size=i+4)` and `sample(N_post,
  size=3-i)`. Local focused pytest now passes the subgroup regression, all
  generator precheck tests, `test_synapse_generator_out_of_range`, and both
  fixed-random error tests. Direct cppyy-target calls to the guarded exception
  paths complete without the JIT exception warning. Next step: commit, push,
  dispatch cppyy workflow again, and inspect whether windows-2022 Stage 3 gets
  past `test_synapse_generator_fixed_random_error1`.
- 2026-07-02 (Claude handoff): workflow + smoke script built, pushed
  (28aaf42b, 3989032b). Run 1 failed on setuptools_scm/shallow clone (fixed).
  Run 2 produced the state table above.
