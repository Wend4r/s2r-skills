# CMake Conventions

Project-owned CMake lives under `cmake/` and in the root `CMakeLists.txt`. `thirdparty/protobuf/**`
CMake is vendored — do not touch it unless the task is explicitly about vendored protobuf.

## Contents

- [Style](#style)
- [Module Layout](#module-layout)
- [Aggregate Lists](#aggregate-lists)
- [Platforms](#platforms)
- [Root Options](#root-options)
- [Helper Functions](#helper-functions)
- [Game Manifests](#game-manifests)
- [Protobuf](#protobuf)
- [Targets](#targets)

## Style

Lowercase built-in commands, uppercase project variables, tabs, multi-line `set()` and
`list(APPEND ...)`:

```cmake
set(SOURCESDK_TIER1_SOURCE_FILES
	${SOURCESDK_TIER1_DIR}/bitbuf.cpp
	${SOURCESDK_TIER1_DIR}/tier1.cpp
)
```

## Module Layout

| Size | Form | Existing examples |
| --- | --- | --- |
| A library module that contributes to the shared lists | `cmake/sourcesdk/targets/<module>.cmake`, pulled in with `include()` from the root `CMakeLists.txt` | `tier1.cmake`, `kv3lib.cmake`, `entity2.cmake` |
| A large subproject with its own targets, options or tests | Its own `CMakeLists.txt` in its directory, added with `add_subdirectory()` | `tests/`, vendored protobuf |

A large subproject gets its own `CMakeLists.txt` instead of growing a `.cmake` module that the
root file includes. `add_subdirectory()` gives it its own variable scope and binary directory, so
its internal variables do not leak into the root, and it can be switched on and off as a whole:

```cmake
if(SOURCESDK_ENABLE_TESTS)
	enable_testing()
	add_subdirectory(tests)
endif()
```

Results the rest of the build needs leave the subproject through targets and their
`${PROJECT_NAME}::<target>` aliases, or through `set(... PARENT_SCOPE)` — not through variables
that only an `include()` would share.

## Aggregate Lists

These lists are accumulated across included modules on purpose:

- `SOURCESDK_SOURCE_FILES`
- `SOURCESDK_INCLUDE_DIRS`
- `SOURCESDK_COMPILE_DEFINITIONS`
- `SOURCESDK_LINK_LIBRARIES`
- `PLATFORM_COMPILE_OPTIONS`
- `PLATFORM_LINK_OPTIONS`
- `PLATFORM_COMPILE_DEFINITIONS`

When adding sources, include dirs, definitions, link options or libraries, append to the nearest
existing `SOURCESDK_<MODULE>_*`, `SOURCESDK_*` or `PLATFORM_*` list instead of bypassing the module
structure.

## Platforms

- Detection stays centralized in `cmake/platform/shared.cmake`. Use the repository booleans
  `WINDOWS`, `LINUX`, `MACOS` rather than new direct checks.
- Platform-specific compiler and linker flags belong in `cmake/platform/windows.cmake`,
  `linux.cmake` and `macos.cmake`.
- Binary-compatibility flags stay deliberate. Do not casually change `_GLIBCXX_USE_CXX11_ABI`,
  export maps, RPATH, MSVC runtime selection, SSE flags or the `_WIN32` / `POSIX` definitions.

## Root Options

Extend these before adding a new cache option.

| Option | Effect |
| --- | --- |
| `SOURCESDK_GAME_TARGET` | Selects the `CMakeGameManifests.json` entry (default `cs2`) |
| `SOURCESDK_AM_DEFINES` | Enables the manifest `am_defines` block |
| `SOURCESDK_COMPILE_PROTOBUF` | Builds and generates protobuf here |
| `SOURCESDK_CREATE_INTEFACE_OVERRIDE` | Custom `CreateInterface` (spelling as-is) |
| `SOURCESDK_ENABLE_TESTS` | Builds `tests/` |
| `SOURCESDK_CONFIGURE_EXPORT_MAP`, `SOURCESDK_LINK_ENABLE_RPATH`, `SOURCESDK_LINK_USE_MOLD`, `SOURCESDK_LINK_STRIP_SYMBOLS`, `SOURCESDK_LINK_STRIP_CPP_EXPORTS` | Unix link behavior |
| `SOURCESDK_LINK_TIER0`, `SOURCESDK_LINK_STEAMWORKS` | Imported shared libraries |
| `SOURCESDK_GENERATE_CLANGD` | Writes `.clangd` at the SDK root with the `sourcesdk` target flags, so clangd resolves headers without a `compile_commands.json` entry |
| `SOURCESDK_MALLOC_OVERRIDE`, `SOURCESDK_MSVC_RUNTIME_LIBRARY`, `SOURCESDK_USE_ABI0` | ABI and runtime compatibility |

## Helper Functions

### `append_sourcesdk_shared_library( LIB_NAME LIB_FILENAME_OUT IMPLIB_FILENAME_OUT )`

Resolves imported binary paths under `lib/<platform>/` and returns the shared-library and
import-library filenames through parent-scope outputs. May copy and patch Linux shared libraries
when `SOURCESDK_LINK_STRIP_CPP_EXPORTS` is on. Reuse it for imported SDK shared libraries such as
`tier0` and `steam_api`.

### `sourcesdk_generate_clangd( TARGET <target> [OUTPUT <file>] [PATH_MATCH <regex>...] )`

Defined in `cmake/sourcesdk/clangd.cmake`. Writes a `.clangd` fragment with the target's include
directories, definitions, compile options and C++ standard flag via generator expressions.
`PATH_MATCH` regexes are relative to the output file's directory. Projects that
`add_subdirectory()` the SDK call it for their own targets, e.g. the headers of a single-TU plugin.

### `append_proto_dirs( OUT_ARGS PROTO_DIRS )`

Converts proto include dirs into `-I...` arguments and writes them to the named parent-scope
variable. Pass list variables carefully.

### `sourcesdk_compile_protos( PROTO_FILENAMES PROTO_ARGS PROTO_DIR PROTO_OUTPUT_DIR LOGS_DIR ERROR_LOGS_DIR PROTO_OUT_PREFIX )`

Invokes the repository `protoc`, creates output, log and error dirs, and skips files whose `.pb.cc`
already exists. Do not replace it with an ad hoc `execute_process()`.

## Game Manifests

`sourcesdk_parse_game_manifests(...)` recursively parses `CMakeGameManifests.json`, follows
`inherits`, extracts `name`, `game_dir`, `protobufs_dir`, `defines` and conditional `am_defines`,
and contributes `SE_NAME`, `SE_GAME_DIR` and game-specific definitions.

Keep inherited entries minimal and validate the configure output for the selected
`SOURCESDK_GAME_TARGET`.

## Protobuf

- Selection is driven by `SOURCESDK_PROTOS`, `SOURCESDK_CUSTOM_PROTOS`, `SOURCESDK_SKIP_PROTOS`
  and `SOURCESDK_CUSTOM_SKIP_PROTOS`. Entries are base names without `.proto`.
- Generated outputs live under `${CMAKE_CURRENT_BINARY_DIR}/protos`. Do not commit generated
  `.pb.*` files unless the task requires it.
- `cmake/sourcesdk/proto/clean_prev.cmake` deletes stale generated proto headers from `common/`
  when `SOURCESDK_GAME_TARGET` changes. It deletes files during configure — change it carefully.

## Targets

Target modules under `cmake/sourcesdk/targets/` create static project libraries or imported shared
libraries and expose `${PROJECT_NAME}::<target>` aliases. Follow the existing target-property
block for C/C++ standards, MSVC runtime, macOS architecture, compile options, definitions,
includes and link libraries.
