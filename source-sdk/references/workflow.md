# Build, Verification, Git and Commits

## Contents

- [Pick the Verification](#pick-the-verification)
- [Presets](#presets)
- [Tests](#tests)
- [CI](#ci)
- [Git and Workspace Safety](#git-and-workspace-safety)
- [Commit Messages](#commit-messages)
- [Documentation](#documentation)
- [Long Sessions](#long-sessions)

## Pick the Verification

Choose the narrowest check that covers the change:

| Change | Verification |
| --- | --- |
| Header-only or formatting | Formatting check or a targeted compile |
| C++ implementation | Build the smallest relevant target, or the existing test or compile command |
| CMake or tooling | `cmake --preset <name>` configure, or the relevant preset build |
| Gamedata | Validate JSON syntax; verify signatures and offsets against binary evidence |

- Do not run expensive full builds unless the risk justifies it or the user asks.
- If verification could not be run, state exactly what was not run and why.
- Never report a build or test as passing that you did not observe pass.

## Presets

Defined in `CMakePresets.json`:

| Kind | Names |
| --- | --- |
| Configure | `VisualStudio` (Windows only), and the Ninja configs `Debug`, `RelWithDebInfo`, `Release` |
| Build and test | The same names, plus `VisualStudio\Debug` and `VisualStudio\Release` |

- Every configure preset forces `SOURCESDK_ENABLE_TESTS` on; the root option itself defaults to
  `OFF`.
- Build dirs are `build/<hostSystemName>/<presetName>`.
- Test presets run CTest with `outputOnFailure`.

## Tests

`tests/` uses the in-repo runner under `tests/common/`, not an external framework. Each source file
is one executable, registered as `<name>_tests` or `smoke_<name>_tests`.

Narrow a run with a regex:

```sh
ctest --preset Debug -R utlvector
```

## CI

`.github/workflows/` configures and builds `Debug`, `RelWithDebInfo` and `Release` on Linux, macOS
and Windows, and runs `ctest` on the Debug preset only.

## Git and Workspace Safety

- The worktree may contain user changes. Do not reset, checkout, delete or rewrite files you did
  not intentionally modify.
- Check `git status --short` before and after editing.
- Commits, branch operations, rebases and force updates are out of scope unless the user
  explicitly asks.
- Do not modify generated files unless the task requires it.

## Commit Messages

Use a short imperative subject with a capitalized verb: `Add`, `Update`, `Remove`, `Fix`,
`Correct`, `Move` or `Actualize`. A `CMake:` prefix is acceptable for CMake-only changes:

```text
CMake: add `cs2-beta` game
```

- Keep the subject concise and technical, one line unless a body is requested.
- Put C++ symbols, interfaces, classes, methods and file-like identifiers in backticks:

```text
Add `CFieldPath` class
Update `CEntityIndex` & `CPlayerSlot`
Fix missing `AddRef` in `CSmartPtr::CopyFrom`
```

- Avoid vague subjects such as `build fix` or typoed verbs, even where older history has them.
- An AI co-author goes in a trailer after a blank line:

```text
Co-authored-by: Codex <codex@openai.com>
```

## Documentation

- Write technical documentation in clear English.
- Prefer direct, operational guidance over broad style advice.
- Keep examples short and in the project's C++ style.
- Update nearby documentation when behavior, public APIs, build flags or gamedata formats change.

## Long Sessions

When context is summarized, keep the current task, repository state, files changed, commands run,
verification status, open risks and exact next steps. These conventions stay in force afterwards —
especially the code style, the CMake conventions, the IDA workflow and workspace safety.
